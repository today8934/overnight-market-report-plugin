# Preflight & Setup Wizard

처음 설치한 사용자가 **README를 전혀 읽지 않고도** 즉시 사용할 수 있도록, 3개 MCP 서버의 상태를 자동 확인하고 필요 시 API 키 설정을 대화형으로 안내합니다. 이 플러그인은 `.mcp.json`을 포함하지 않으므로 MCP 등록은 **사용자별 `claude mcp add` CLI 호출**로 이뤄집니다.

## ⚡ Preflight 캐싱

매 실행마다 preflight를 돌리는 것은 이미 세팅된 사용자(99%)에게 **불필요한 지연**입니다. 다음 규칙으로 캐싱하세요.

### 캐시 파일
- 경로: `~/.claude/data/overnight-market-report/preflight.json`
- 포맷:
  ```json
  {
    "last_ok_at": "2026-04-17T07:30:00+09:00",
    "checks": {
      "finnhub": "ok",
      "alphavantage": "ok",
      "tavily": "ok"
    }
  }
  ```

### Skip 규칙 (cache-hit)
`last_ok_at`이 **24시간 이내**이고 `checks`가 모두 `"ok"`면 **preflight을 건너뛰고 메인 워크플로우로 바로 진행**합니다.

### Miss/재검증 트리거
다음 중 하나라도 해당하면 inline preflight 실행:
1. 캐시 파일 없음 (첫 실행)
2. `last_ok_at`이 24시간 초과
3. 지난 실행에서 finnhub/alphavantage/tavily 중 하나라도 **실제 호출이 실패**했음 (401/invalid key 감지)
4. 사용자가 명시적으로 "preflight 다시 돌려줘" 또는 "MCP 재확인" 요청

### Full preflight 통과 시
`{"last_ok_at": "<현재 KST ISO8601>", "checks": {...}}`로 캐시 파일을 갱신한 뒤 메인 워크플로우 진행. 디렉토리가 없으면 `mkdir -p`로 생성.

### Full preflight 실패 시
캐시 파일의 `last_ok_at`은 **갱신하지 않고** Setup Wizard로 넘어갑니다. Wizard 완료 후 재검증이 통과하면 그때 캐시 갱신.

## 🔎 Inline preflight (메인 세션이 직접 수행)

**이 단계는 subagent를 쓰지 않습니다.** 메인 세션이 직접 4단계로 처리합니다. 이전 버전의 subagent 방식 대비 10~20초 단축 + 메시지 왕복 감소.

### 단계

1. **스키마 로드**: 한 번에 3개 도구를 `ToolSearch`로 확보.
   ```
   ToolSearch(query="select:mcp__finnhub__finnhub_stock_market_data,mcp__alphavantage__get_forex_rate,mcp__tavily__tavily_search", max_results=5)
   ```

2. **병렬 cheap test** (한 메시지 안에서 동시 호출):
   - finnhub: `operation="get_quote"`, `symbol="SPY"`
   - alphavantage: `from_currency="USD"`, `to_currency="KRW"`
   - tavily: `query="test"`, `max_results=5`

   스키마 로드에 실패한 도구는 `not_loaded`로 마크하고 호출은 생략.

3. **판정**:
   | 상태 | 조건 |
   |---|---|
   | `ok` | 유효 JSON + 실제 데이터 (finnhub `c > 0`, alphavantage `Realtime Currency Exchange Rate` 키 존재, tavily `results` 배열 길이 > 0) |
   | `not_loaded` | ToolSearch가 해당 도구 스키마를 반환하지 않음 |
   | `auth_error` | 응답에 `401` / `invalid` / `unauthorized` / `api key` / `authentication` 포함 |
   | `other_error` | 기타 실패 |

4. **분기**:
   - 셋 다 `ok` → 캐시 갱신 + 메인 워크플로우 진행
   - 하나라도 `ok`가 아님 → **Setup Wizard 진입** (아래 절차). 캐시는 갱신하지 않음.

## 🧙 Setup Wizard (메인 세션)

inline preflight이 실패 항목을 반환하면 **메인 세션이 직접** 다음 절차로 사용자와 대화하며 해결합니다 (subagent 아님).

### 1단계. 상황 설명 (짧게)
> "이 플러그인은 3개의 무료 API 키가 필요합니다. 현재 {N}개가 미설정 상태라 자동 설정을 도와드릴게요. 각 키는 모두 무료 발급 가능합니다."

`{N}`은 inline preflight 결과 중 `status != "ok"`인 항목 수. 어느 MCP가 실패했는지도 한 줄로 명시.

### 2단계. 누락된 키만 순차 수집
`AskUserQuestion` 도구를 **실패한 MCP에 대해서만** 1개씩 순차 호출:

- **finnhub** (status != "ok"일 때만):
  > "Finnhub API 키를 입력해주세요. 무료 발급: https://finnhub.io/register (가입 → Dashboard에서 키 복사, 보통 40자 영숫자)"

- **alphavantage** (status != "ok"일 때만):
  > "Alpha Vantage API 키를 입력해주세요. 무료 발급: https://www.alphavantage.co/support/#api-key (16자 대문자·숫자)"

- **tavily** (status != "ok"일 때만):
  > "Tavily API 키를 입력해주세요. 무료 발급: https://tavily.com (가입 → API Keys 메뉴, `tvly-`로 시작)"

각 응답은 trim. 빈 문자열이거나 10자 미만이면 "키가 너무 짧습니다. 다시 확인해 입력해주세요"로 한 번 재요청. 2번째도 실패면 Wizard halt.

### 3단계. MCP 서버 등록 (Bash)
각 MCP에 대해 **기존 등록이 있으면 먼저 remove 후 add**:

```
claude mcp remove finnhub 2>/dev/null
claude mcp add finnhub -e FINNHUB_API_KEY="<입력받은 키>" -- npx -y @aigroup/finnhub-mcp

claude mcp remove alphavantage 2>/dev/null
claude mcp add alphavantage -e ALPHA_VANTAGE_API_KEY="<입력받은 키>" -- npx -y alpha-vantage-mcp

claude mcp remove tavily 2>/dev/null
claude mcp add tavily -e TAVILY_API_KEY="<입력받은 키>" -- npx -y tavily-mcp@latest
```

**보안**: 사용자 입력을 쉘 인자로 삽입할 때 반드시 쌍따옴표로 감싸고, 내부에 `"`나 `$`가 있으면 거부 + 재입력 요청.

### 4단계. 등록 결과 검증
```
claude mcp list 2>&1 | grep -E 'finnhub|alphavantage|tavily'
```
세 서비스가 모두 출력에 포함되는지 확인. 누락이 있으면 stderr를 사용자에게 그대로 보여주고 수동 편집 안내.

### 5단계. 사용자에게 완료 + 재시작 안내
> "✅ MCP 서버 {등록_개수}개 등록 완료 (finnhub / alphavantage / tavily). **Claude Code를 한 번 재시작**한 뒤 다시 '미국주식 야간 보고서'라고 말씀해주시면 자동으로 리포트를 만들어드립니다."

### 6단계. halt
이 호출에서는 **메인 워크플로우로 절대 진행하지 않습니다.** 재시작·재호출 후 다음 invocation의 preflight이 통과하면 그때 정상 워크플로우가 돌아갑니다. **캐시 파일은 이 시점에 갱신하지 않습니다** — 재시작 후 첫 실행에서 inline preflight을 다시 돌려 실제 작동 여부를 검증해야 `last_ok_at`을 기록하는 것이 안전합니다.

## 🛡 Fault tolerance

- **ToolSearch 실패** → 네트워크/플러그인 로더 문제. 사용자에게 "Claude Code 재시작" 안내 후 halt
- **cheap test 호출 응답 무응답/타임아웃** → `other_error`로 마크, Setup Wizard로 진행
- **사용자가 키 입력 거부/취소** → Wizard halt, "나중에 필요하면 다시 호출해주세요"로 마무리
- **`claude mcp add` 실패** → stdout/stderr를 사용자에게 그대로 보여주고 수동 명령 복사·실행 안내
- **`claude` CLI가 PATH에 없음** → 드문 케이스. 에러 메시지에서 감지되면 `~/.claude.json`의 `mcpServers` 블록에 직접 추가하는 JSON 스니펫을 출력
- **실행 중 401/invalid key 감지** → preflight 캐시의 `last_ok_at`을 무효화(파일 삭제 또는 `last_ok_at`을 오래된 값으로 덮어쓰기)하고 다음 호출에서 inline preflight이 다시 돌게 유도
