# Preflight & Setup Wizard

처음 설치한 사용자가 **README를 전혀 읽지 않고도** 즉시 사용할 수 있도록, 3개 MCP 서버의 상태를 자동 확인하고 필요 시 API 키 설정을 대화형으로 안내합니다. 이 플러그인은 `.mcp.json`을 포함하지 않으므로(치환 이슈 회피), MCP 등록은 **사용자별 `claude mcp add` CLI 호출**로 이뤄집니다.

## ⚡ Preflight 캐싱 (v0.3.0~)

매 실행마다 preflight subagent를 돌리는 것은 이미 세팅된 사용자(99%)에게 **10~20초의 불필요한 지연**입니다. 다음 규칙으로 캐싱하세요.

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
`last_ok_at`이 **24시간 이내**이고 `checks`가 모두 `"ok"`면 **preflight subagent를 건너뛰고 메인 워크플로우로 바로 진행**합니다.

### Miss/재검증 트리거
다음 중 하나라도 해당하면 full preflight 실행:
1. 캐시 파일 없음 (첫 실행)
2. `last_ok_at`이 24시간 초과
3. 지난 실행에서 finnhub/alphavantage/tavily 중 하나라도 **실제 호출이 실패**했음 (메인 워크플로우 진행 중 도구 호출 시 401/invalid key 감지)
4. 사용자가 명시적으로 "preflight 다시 돌려줘" 또는 "MCP 재확인" 요청

### Full preflight 통과 시
`{"last_ok_at": "<현재 KST ISO8601>", "checks": {...}}`로 캐시 파일을 갱신한 뒤 메인 워크플로우 진행.

### Full preflight 실패 시
캐시 파일의 `last_ok_at`은 **갱신하지 않고** Setup Wizard로 넘어갑니다. Wizard 완료 후 재검증이 통과하면 그때 캐시 갱신.

### 디렉토리 자동 생성
캐시 파일을 쓰기 전에 `~/.claude/data/overnight-market-report/`가 없으면 생성하세요 (`mkdir -p`).

## 🔎 preflight-check subagent

`Agent` tool (`subagent_type: general-purpose`)로 아래 프롬프트를 전달. 응답은 30초 이내 **JSON 하나만** 반환하도록 엄격 제약.

```
역할: overnight-market-report-plugin이 사용하는 3개 MCP 서버의 정상 동작 여부를 확인하는 preflight checker. 오직 상태 판정과 JSON 반환만 수행.

## 검증 대상
1. `mcp__finnhub__finnhub_stock_market_data` — 미국 주식 시세
2. `mcp__alphavantage__get_forex_rate` — 환율
3. `mcp__tavily__tavily_search` — 뉴스 검색

## 절차
1. ToolSearch로 세 도구 스키마를 한 번에 로드 시도:
   ToolSearch(query="select:mcp__finnhub__finnhub_stock_market_data,mcp__alphavantage__get_forex_rate,mcp__tavily__tavily_search", max_results=5)

2. 로드에 성공한 도구들을 **한 메시지에서 병렬**로 테스트 호출:
   - finnhub: operation="get_quote", symbol="SPY"
   - alphavantage: from_currency="USD", to_currency="KRW"
   - tavily: query="test", max_results=5

3. 각 결과를 아래 분류로 판정:
   - **ok**: 유효 JSON + 실제 데이터 (finnhub의 `c` 필드 > 0, alphavantage의 `Realtime Currency Exchange Rate` 키 존재, tavily의 `results` 배열 길이 > 0)
   - **not_loaded**: ToolSearch가 해당 도구 스키마를 반환하지 않음 (MCP 서버 자체가 미등록)
   - **auth_error**: 응답 본문에 "401" / "invalid" / "unauthorized" / "api key" / "authentication" 같은 단어가 포함된 에러
   - **other_error**: 기타 실패 (네트워크, 레이트리밋, 스키마 불일치 등)

4. 아래 형식으로만 반환 (**JSON 객체 하나만**, 다른 텍스트 일절 금지):

{
  "overall": "ready" 또는 "needs_setup",
  "checks": {
    "finnhub":      { "status": "<ok|not_loaded|auth_error|other_error>", "detail": "한 줄 요약" },
    "alphavantage": { "status": "<ok|not_loaded|auth_error|other_error>", "detail": "한 줄 요약" },
    "tavily":       { "status": "<ok|not_loaded|auth_error|other_error>", "detail": "한 줄 요약" }
  }
}

- `overall`은 세 status가 모두 `ok`면 `"ready"`, 그 외엔 `"needs_setup"`
- `detail`은 실패 시 에러 메시지 첫 100자 이하 요약, 성공 시 받은 값 한 줄 (예: "SPY c=679.46")

## 제약
- 본문·인사말·설명·이모지 금지. JSON 객체 하나만.
- 응답 전체 300단어 이하.
- 30초 이내 완료.
- 테스트 호출 응답 전체를 붙여넣지 말 것.
```

## 🧙 Setup Wizard (메인 세션)

preflight이 `overall: "needs_setup"`을 반환하면 **메인 세션의 Claude 자신**이 다음 절차를 수행합니다. 사용자와 직접 대화하므로 subagent가 아닌 메인 orchestrator가 실행합니다.

### 1단계. 상황 설명 (짧게)
> "이 플러그인은 3개의 무료 API 키가 필요합니다. 현재 {N}개가 미설정 상태라 자동 설정을 도와드릴게요. 각 키는 모두 무료 발급 가능합니다."

`{N}`은 preflight JSON의 `checks` 중 `status != "ok"`인 항목 수. 어느 MCP가 실패했는지도 한 줄로 명시.

### 2단계. 누락된 키만 순차 수집
`AskUserQuestion` 도구를 **실패한 MCP에 대해서만** 1개씩 순차 호출 (하나씩이어야 사용자가 키를 복사해 붙여넣기 편함):

- **finnhub** (status != "ok"일 때만):
  > "Finnhub API 키를 입력해주세요. 무료 발급: https://finnhub.io/register (가입 → Dashboard에서 키 복사, 보통 40자 영숫자)"

- **alphavantage** (status != "ok"일 때만):
  > "Alpha Vantage API 키를 입력해주세요. 무료 발급: https://www.alphavantage.co/support/#api-key (16자 대문자·숫자)"

- **tavily** (status != "ok"일 때만):
  > "Tavily API 키를 입력해주세요. 무료 발급: https://tavily.com (가입 → API Keys 메뉴, `tvly-`로 시작)"

각 응답은 trim. 빈 문자열이거나 10자 미만이면 "키가 너무 짧습니다. 다시 확인해 입력해주세요"로 한 번 재요청. 2번째도 실패면 Wizard halt.

### 3단계. MCP 서버 등록 (Bash)
각 MCP에 대해 **기존 등록이 있으면 먼저 remove 후 add** (기존 잘못된 키로 등록돼 있을 수 있으므로):

```
claude mcp remove finnhub 2>/dev/null
claude mcp add finnhub -e FINNHUB_API_KEY="<입력받은 키>" -- npx -y @aigroup/finnhub-mcp

claude mcp remove alphavantage 2>/dev/null
claude mcp add alphavantage -e ALPHA_VANTAGE_API_KEY="<입력받은 키>" -- npx -y alpha-vantage-mcp

claude mcp remove tavily 2>/dev/null
claude mcp add tavily -e TAVILY_API_KEY="<입력받은 키>" -- npx -y tavily-mcp@latest
```

**보안**: 사용자 입력을 쉘 인자로 삽입할 때 반드시 쌍따옴표로 감싸고, 내부에 `"`나 `$`(가) 있으면 거부 + 재입력 요청.

### 4단계. 등록 결과 검증
```
claude mcp list 2>&1 | grep -E 'finnhub|alphavantage|tavily'
```
세 서비스가 모두 출력에 포함되는지 확인. 누락이 있으면 stderr를 사용자에게 그대로 보여주고 수동 편집 안내.

### 5단계. 사용자에게 완료 + 재시작 안내
> "✅ MCP 서버 {등록_개수}개 등록 완료 (finnhub / alphavantage / tavily). **Claude Code를 한 번 재시작**한 뒤 다시 '미국주식 야간 보고서'라고 말씀해주시면 자동으로 리포트를 만들어드립니다."

### 6단계. halt
이 호출에서는 **메인 워크플로우(Step 1~7)로 절대 진행하지 않습니다.** 재시작·재호출 후 다음 invocation의 preflight이 통과하면 그때 정상 워크플로우가 돌아갑니다. **캐시 파일은 이 시점에 갱신하지 않습니다** — 재시작 후 첫 실행에서 preflight을 다시 돌려 실제 작동 여부를 검증해야 `last_ok_at`을 기록하는 것이 안전합니다.

## 🛡 Fault tolerance

- **preflight subagent 실패/빈 응답** → 메인 세션이 직접 `ToolSearch("select:mcp__finnhub__finnhub_stock_market_data,mcp__alphavantage__get_forex_rate,mcp__tavily__tavily_search")`를 시도해 스키마가 모두 확보되는지 확인. 실패 시 needs_setup으로 간주하고 Setup Wizard 실행
- **사용자가 키 입력 거부/취소** → Wizard halt, "나중에 필요하면 다시 호출해주세요"로 마무리
- **`claude mcp add` 실패** → stdout/stderr를 사용자에게 그대로 보여주고 수동 명령 복사·실행 안내
- **`claude` CLI가 PATH에 없음** → 드문 케이스. 에러 메시지에서 감지되면 `~/.claude.json`의 `mcpServers` 블록에 직접 추가하는 JSON 스니펫을 출력
- **실행 중 401/invalid key 감지** → preflight 캐시의 `last_ok_at`을 무효화(파일 삭제 또는 `last_ok_at`을 오래된 값으로 덮어쓰기)하고 다음 호출에서 full preflight이 돌게 유도
