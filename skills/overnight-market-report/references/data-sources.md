# 데이터 소스 역할 분담 매트릭스

**실제 검증된 결과**에 기반한 역할 분담입니다. 각 소스의 강점을 살려 조합하세요.

## 📊 시세(숫자) = MCP를 주력으로 사용

**원칙: 가격/변동률은 반드시 MCP에서 가져온다. 뉴스 기사에서 숫자를 긁어오는 건 마지막 수단이다.** 실제 테스트에서 뉴스 기사의 종가와 MCP 실시간 데이터가 달랐던 사례가 있었습니다 — MCP가 진실입니다.

### 🥇 `mcp__finnhub__finnhub_stock_market_data` (operation=`get_quote`) — **주력**
✅ 검증 완료: SPY, QQQ, DIA, IWM, XLK, XLE, XLV, USO, GLD, UUP, NVDA, AAPL, TSLA, MSFT 등 **모든 US-listed ETF/Stock 작동**.

응답 필드:
- `c` = current price (종가)
- `d` = change (변동)
- `dp` = change percent (변동률 %)
- `h` = day high, `l` = day low, `o` = day open, `pc` = previous close

### 🥈 `mcp__alphavantage__get_forex_rate` — 환율 전용
✅ 검증 완료. USD/KRW, USD/JPY 등 환율.

### ⚠️ `mcp__alphavantage__get_stock_price` — 백업
작동하지만 **무료 티어 하루 25회 제한**. finnhub이 실패한 특정 티커에만 선별적으로 사용.

### ❌ `mcp__yfinance__getStockAnalysis` — 사용 금지
현재 환경에서 모든 티커를 `¥0 / 데이터 없음`으로 반환. 호출 자체가 시간 낭비. **절대 쓰지 마세요**.

### ❌ finnhub `^VIX`, `^TNX` 등 지수 심볼
"CFD subscription required" 에러. 지수 심볼 대신 **ETF 프록시** 또는 **WebSearch**로 우회.

## 📰 뉴스·매크로 = Tavily + WebSearch **반드시 병렬로**

이 skill은 **Tavily MCP를 설치 전제**로 설계되어 있습니다. Tavily와 WebSearch는 서로를 대체하는 것이 아니라 **교차검증용으로 함께 사용**합니다. Tavily는 금융 뉴스 원문 추출(extract)에 특화되어 있어 WebSearch가 놓치는 원문 세부 정보를 보완합니다.

### 🥇 Tavily MCP — **필수 도구**
- 도구 이름: `tavily_search`, `tavily_extract` (정확한 이름은 세션 시작 시 `ToolSearch`의 `select:tavily_search,tavily_extract` 또는 `query: "tavily"` 로 확인)
- 사용법:
  - `tavily_search`: 여러 쿼리를 병렬 호출해 금융 뉴스·매크로·지정학 정보 수집
  - `tavily_extract`: 핵심 기사 URL을 통째로 원문 추출 (WebSearch 스니펫보다 정확)
- **반드시 WebSearch와 같은 메시지 안에서 병렬 호출**해 교차검증

### 🥇 `WebSearch` — **필수 도구**
매크로 이벤트, 지정학, 경제지표 발표, 기업 실적 서프라이즈 등 뉴스성 정보의 기본 소스. Tavily와 병렬로 실행.

**⚠️ 쿼리 작성 규칙**: WebSearch는 연/월을 명시하지 않으면 과거 자료를 반환할 수 있음. **모든 쿼리에 현재 연/월 포함**.

### Tavily가 세션에 로드되지 않은 경우

`ToolSearch`로 `tavily_search`를 찾을 수 없으면 Tavily MCP가 이 세션에서 아직 활성화되지 않았다는 뜻입니다 (설치 직후 Claude Code 재시작 전일 수 있음). 이 경우:

1. 사용자에게 "Tavily MCP가 이 세션에 로드되지 않아 WebSearch만으로 뉴스를 수집합니다"라고 고지
2. WebSearch 쿼리를 평소보다 **2배 다양한 각도**로 실행해 Tavily 부재를 보완. 구체 쿼리 쌍 예시:

   | 주제 | 쿼리 쌍 (2개 → 4~5개로 확장) |
   |---|---|
   | 연준·금리 | `"FOMC decision {Month} {YYYY}"` + `"Powell speech hawkish dovish {Month} {YYYY}"` + `"CME FedWatch probability {Month} {YYYY}"` + `"dot plot projection {Month} {YYYY}"` |
   | 인플레·지표 | `"CPI report {Month} {YYYY}"` + `"core PCE release {Month} {YYYY}"` + `"jobs report nonfarm payrolls {Month} {YYYY}"` + `"retail sales {Month} {YYYY}"` |
   | 지정학 | `"Middle East Iran Israel {Month} {YYYY} market"` + `"Ukraine Russia conflict {Month} {YYYY} impact"` + `"Taiwan strait tension {Month} {YYYY}"` + `"Trump tariff {Month} {YYYY}"` |
   | 원자재 | `"WTI crude oil price {Month} {D} {YYYY} reason"` + `"gold rally {Month} {YYYY}"` + `"copper price {Month} {YYYY}"` + `"natural gas {Month} {YYYY}"` |
   | 기업 실적 | `"earnings surprise {Month} {D} {YYYY}"` + `"guidance cut raise {Month} {D} {YYYY}"` + `"analyst downgrade upgrade {Month} {D} {YYYY}"` |

3. 보고서 말미 "데이터 품질" 섹션에 "Tavily 미로드" 명시

### ⚠️ `mcp__finnhub__finnhub_news_sentiment` — subagent 경유만
응답이 75KB 초과로 컨텍스트 한도를 초과한 적 있음. 메인 context에 직접 호출 금지. 필요 시 `Agent` tool로 위임해 "Top 5 헤드라인 + 1줄 요약 + URL"만 받아오세요.

### ❌ `mcp__finnhub__finnhub_calendar_data`
API key 이슈로 실패. 경제지표 캘린더는 WebSearch로 대체.

## 일반 원칙

- **실패 허용**: 어떤 도구가 실패해도 전체를 중단하지 말고 다른 소스로 대체. "데이터 품질" 섹션에 실패 내역 명시.
- **병렬 원칙**: 독립적인 호출은 **반드시 한 메시지에 동시 실행**. 섹션당 10~20개 티커를 묶으면 전체 수집이 한 번에 끝납니다.
