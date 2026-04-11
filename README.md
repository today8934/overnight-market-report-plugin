# Overnight Market Report Plugin

간밤 미국 증시와 국제정세를 자동으로 수집·분석해 **한국장 개장 전 브리핑용 한국어 마크다운 보고서**를 만드는 Claude Code 스킬 플러그인입니다. 전문가용 원본 리포트와 **주식 입문자용 쉬운버전**을 한 번에 생성합니다.

> "미국주식", "야간시황", "밤사이 증시", "미장 마감", "어제 미장 어땠어" 등으로 호출하면 자동 트리거됩니다.

---

## ✨ 주요 특징

- **하이브리드 오케스트레이션** — 메인 세션이 `finnhub get_quote`로 30+ 종목·섹터·글로벌 ETF 시세를 병렬 수집하고, 별도 `news-harvester` 서브에이전트가 `Tavily search/extract + WebSearch`로 뉴스·매크로 이벤트를 격리 수집합니다. 메인 컨텍스트가 대용량 뉴스 원문으로 오염되지 않음
- **인과 분석** — 단순 숫자 나열이 아니라 "왜 그렇게 움직였는지"를 섹터 로테이션·원자재·매크로 이벤트와 연결해 설명
- **두 버전 자동 생성** —
  1. **원본 리포트** (전문가용, 디버전스·섹터 로테이션·FedWatch 같은 전문용어 사용)
  2. **쉬운버전** (입문자용, 상단 미니 용어집 + 본문에 괄호 해설 + 표 아래 "쉬운 해석" 삽입)
- **데이터 충돌 자동 해소** — 서브에이전트가 인용한 뉴스 수치와 MCP 실시간 값이 다르면 **MCP를 진실로 채택**하고 해소 내역을 리포트 말미에 기록
- **저장 경로**: `~/workspace/overnight-market-report/YYYY-MM-DD.md` + `YYYY-MM-DD-쉬운버전.md` (같은 날짜 파일이 이미 있으면 `YYYY-MM-DD-HHMM.md` 형태로 시간 suffix)

---

## 📋 사전 준비 — API 키 3종 발급 (모두 무료 플랜 가능)

이 플러그인은 세 개의 MCP 서버를 사용하며, 각각 무료 API 키가 필요합니다.

### 1. Finnhub (주력 시세 소스) — **필수**
- 가입: https://finnhub.io/register
- 무료 플랜: 분당 60 호출, 실시간 미국 주식·ETF 시세
- 이 플러그인은 한 번 실행 시 30~34개 종목을 병렬 호출하므로 무료 플랜으로 충분

### 2. Alpha Vantage (환율) — **필수**
- 가입: https://www.alphavantage.co/support/#api-key
- 무료 플랜: **하루 25회 제한**
- 이 플러그인은 USD/KRW, USD/JPY 두 건만 호출하므로 하루 10회 실행해도 여유

### 3. Tavily (뉴스·매크로 리서치) — **필수**
- 가입: https://tavily.com
- 무료 크레딧으로 시작 가능 (월 1,000 credits 정도)
- `tavily_search` + `tavily_extract`를 서브에이전트에서 사용

### 환경변수 설정
발급받은 키를 셸 프로필(`~/.zshrc` 또는 `~/.bashrc`)에 등록:

```bash
export FINNHUB_API_KEY="your-finnhub-key-here"
export ALPHA_VANTAGE_API_KEY="your-alpha-vantage-key-here"
export TAVILY_API_KEY="tvly-dev-your-tavily-key-here"
```

설정 후 새 셸을 열거나 `source ~/.zshrc`로 적용하세요.

---

## 🚀 설치

### 옵션 A — 로컬 디렉토리로 테스트
```bash
git clone https://github.com/today8934/overnight-market-report-plugin.git
cd overnight-market-report-plugin
claude --plugin-dir .
```

### 옵션 B — Claude Code 플러그인 마켓플레이스에 추가
프로젝트 또는 사용자 전역 `~/.claude/` 설정의 `.claude-plugin/marketplace.json`에 아래 항목을 추가한 뒤 Claude Code에서 `/plugin install` 실행:
```json
{
  "plugins": [
    {
      "name": "overnight-market-report-plugin",
      "source": "github:today8934/overnight-market-report-plugin"
    }
  ]
}
```

설치 후 Claude Code를 재시작하면 MCP 서버(finnhub·alphavantage·tavily)가 자동으로 로드됩니다. `/mcp` 명령으로 세 서버가 모두 `✓ Connected` 상태인지 확인하세요.

---

## 🎯 사용법

Claude Code 세션에서 아래와 같은 자연어 문구 중 아무거나 입력하면 자동 트리거됩니다:

- "미국주식 야간 보고서 뽑아줘"
- "밤사이 증시 어땠어?"
- "간밤 미국 시황 정리해줘"
- "어제 미장 어땠어"
- "밤사이 무슨 일 있었어"
- "미국 증시 브리핑"

Claude는 다음 단계를 순차/병렬로 수행합니다:

1. 한국 시각 확인 → 저장 경로 결정
2. `finnhub get_quote` ×30+ (지수/섹터/Mag7/원자재/글로벌 ETF), `alphavantage forex` ×2, `WebSearch` ×2~3, `news-harvester` 서브에이전트 ×1 — **한 메시지에서 병렬 실행**
3. 시세·뉴스 종합 분석 → 원본 리포트 작성·저장
4. `readability-pass` 서브에이전트 실행 → 같은 디렉토리에 `-쉬운버전.md` 생성
5. 사용자에게 두 경로와 TL;DR만 짧게 보고

대략 2~3분 내에 두 파일이 완성됩니다.

---

## 📄 산출물 예시 구조

### 원본 리포트 (`YYYY-MM-DD.md`)
1. 핵심 요약 (TL;DR)
2. 미국 3대 지수 (SPY/QQQ/DIA/IWM + VIX)
3. 섹터 동향 (XL* ETF 11종 일간 변동률 정렬)
4. 주요 종목 (Mag7)
5. 원자재 · 금리 · 환율
6. 글로벌 증시 (ETF 프록시)
7. 국제정세 & 매크로 이벤트 (연준·인플레·지정학·실적)
8. 오늘 한국장 관전 포인트
9. 데이터 품질 & 소스 *(말미)*

### 쉬운버전 (`YYYY-MM-DD-쉬운버전.md`)
- 최상단: **📘 미니 용어집** (디버전스·섹터 로테이션·CPI·FOMC·VIX·ETF·Mag7 등)
- 본문: 원본과 동일 구조, 단 전문용어 첫 등장 시 괄호 해설 자동 삽입
- 표 아래: "쉬운 해석" 한두 줄 추가
- 말미: 데이터 품질 & 소스

---

## 🧠 데이터 소스 역할 분담

| 소스 | 역할 | 주의사항 |
|---|---|---|
| `finnhub get_quote` | **주력 시세** (US ETF·주식 전수) | 지수 심볼(`^VIX`, `^GSPC`)은 CFD 구독 제한 → ETF 프록시(SPY/QQQ/VIXY)로 우회 |
| `alphavantage forex_rate` | 환율 (USD/KRW, USD/JPY) | 무료 25/day 제한 — 환율 전용으로만 사용 권장 |
| `tavily_search` + `tavily_extract` | 뉴스·매크로 원문 리서치 | 서브에이전트 격리 호출 (메인 context 오염 방지) |
| `WebSearch` (Claude 내장) | 뉴스 교차검증, VIX/10Y 금리 백업 | 쿼리에 항상 현재 연/월 명시 |

**사용 금지 도구** — 플러그인 내부에서 호출하지 않음:
- `yfinance` MCP: 이 저자 환경에서 모든 티커를 `¥0`으로 반환하는 버그 확인
- `finnhub news_sentiment` 메인 직접 호출: 응답 75KB+로 컨텍스트 오염 → 필요 시 서브에이전트 경유

---

## 🏗 오케스트레이션 구조

```
┌────────────────────────────────────────────────────┐
│  Main Orchestrator (Claude 세션)                    │
│                                                     │
│  한 메시지에서 병렬:                                  │
│  ├─ finnhub get_quote ×30+ (지수/섹터/Mag7/글로벌)   │
│  ├─ alphavantage get_forex_rate ×2                 │
│  ├─ WebSearch ×2~3 (VIX/10Y 보조)                  │
│  └─ Agent → news-harvester subagent ×1 ─┐          │
│                                          │          │
└────────────────────────────────────────┼──────────┘
                                         │
                     ┌───────────────────▼──────────────┐
                     │  news-harvester (격리 subagent)    │
                     │  - tavily_search 병렬 6쿼리         │
                     │  - WebSearch 교차검증               │
                     │  - tavily_extract 핵심 기사 3~5개   │
                     │  → 800단어 이하 압축 요약만 반환    │
                     └──────────────┬───────────────────┘
                                    │
                     ┌──────────────▼───────────────────┐
                     │  Main에서 시세+뉴스 종합 → 원본 작성 │
                     └──────────────┬───────────────────┘
                                    │
                     ┌──────────────▼───────────────────┐
                     │  readability-pass subagent (순차)  │
                     │  - 원본 읽기 → 입문자용 리라이팅     │
                     │  - 상단 미니 용어집 + 괄호 해설     │
                     │  - 숫자/티커/URL 원본 그대로 유지   │
                     │  → -쉬운버전.md 저장                │
                     └──────────────────────────────────┘
```

**Fault tolerance**: 뉴스 수집이 실패해도 시세 기반 원본 리포트는 정상 완성되며, readability-pass가 실패해도 원본은 이미 디스크에 저장된 상태.

---

## 🛠 문제 해결

### MCP 서버가 연결되지 않을 때
```bash
# 상태 확인
claude mcp list

# 특정 서버 재등록 (예: tavily)
claude mcp remove tavily
claude mcp add tavily -e TAVILY_API_KEY=your-key -- npx -y tavily-mcp@latest
```

### `tavily_search`, `tavily_extract`가 도구 목록에 없을 때
Tavily MCP가 로드는 됐지만 deferred tool로 등록된 경우입니다. Claude는 `ToolSearch("select:mcp__tavily__tavily_search,mcp__tavily__tavily_extract")`로 자동 로드합니다.

### finnhub 특정 티커에서 "CFD subscription required" 에러
해외 지수 심볼(`^N225`, `^FTSE` 등)은 무료 플랜 제한입니다. 스킬이 **자동으로 ETF 프록시(EWJ, EWU 등)로 우회**하므로 사용자 조치는 불필요.

### Alpha Vantage 25/day 제한 도달
이 플러그인은 환율 2건만 호출하므로 하루 10회 이상 실행해야 도달합니다. 제한에 걸리면 해당 리포트의 USD/JPY 등이 `N/A`로 표시되지만 전체 리포트는 정상 완성됩니다.

---

## 📝 라이선스

MIT License. 자세한 내용은 `LICENSE` 파일을 참조하세요.

---

## ⚠️ 면책 조항

이 플러그인의 산출물은 **정보 제공 목적**이며 특정 종목의 매수/매도 권유가 아닙니다. 투자 결정은 본인 판단과 책임 하에 진행하세요. MCP 데이터 소스(finnhub, alphavantage, Tavily)의 정확성·가용성은 해당 서비스 제공자에 의해 결정됩니다.

---

## 🙋 기여 & 문의

- GitHub: https://github.com/today8934/overnight-market-report-plugin
- Issues/PR 환영
