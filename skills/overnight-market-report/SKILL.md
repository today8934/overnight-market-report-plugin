---
name: overnight-market-report
description: 간밤 미국 증시(지수·섹터·주요 종목)와 국제정세를 finnhub·alphavantage·Tavily·WebSearch로 동시 수집·분석해 한국장 개장 전 브리핑용 한국어 마크다운 보고서를 생성합니다. 사용자가 "미국주식", "야간시황", "overnight-market-report", "밤사이 증시", "미장 마감", "간밤 뉴욕", "미국 증시 브리핑", "전일 미국장", "어제 미장 어땠어", "밤사이 무슨 일 있었어" 같은 직·간접 표현을 쓸 때 반드시 이 skill을 실행하세요. 단순 가격 조회가 아니라 지수·섹터·매크로·지정학 이벤트를 엮어 인과를 설명하는 종합 리포트가 필요한 모든 상황에서 트리거합니다.
---

# Overnight Market Report

간밤 미국 증시와 국제정세를 병렬 수집·분석해 한국장 개장 전 브리핑용 한국어 마크다운 보고서를 만듭니다.

## 왜 이 skill이 필요한가

한국 투자자는 개장 전에 미국 증시 마감 결과만이 아니라 **왜 그렇게 움직였는지**를 알고 싶어합니다. 단순 숫자 나열은 쓸모가 제한적이고, 지수 → 섹터 → 주요 종목 → 매크로/지정학 이벤트의 인과를 연결해야 실전에 쓸 수 있는 브리핑이 됩니다. 여러 데이터 소스를 한 번에 묶어 자동화합니다.

## 출력 위치

`~/workspace/overnight-market-report/YYYY-MM-DD.md` (오늘 한국 날짜 기준)

디렉토리가 없으면 먼저 생성합니다. 같은 날짜 파일이 이미 존재하면 덮어쓰지 말고 `YYYY-MM-DD-HHMM.md` 형태로 시간 suffix를 붙여 새 파일을 만드세요 — 장중에 다시 돌려도 이전 브리핑을 잃지 않기 위함입니다.

## 실행 순서

1. 오늘 한국 날짜·시각(KST) 확인 → 파일 경로 결정
2. 출력 디렉토리 확인/생성
3. **한 메시지에서 병렬 실행** (하이브리드 오케스트레이션):
   - 시세 MCP 호출 × 20~34 (finnhub get_quote, 직접)
   - 환율 MCP × 2 (alphavantage, 직접)
   - 보조 WebSearch 1~3 (VIX/10Y 등, 직접)
   - **`Agent` tool로 news-harvester subagent 1회** (뉴스 수집 위임)
4. subagent 응답(압축 요약)을 받으면 시세 데이터와 종합하여 인과 분석
5. 템플릿에 채워 md 파일 작성 — **데이터 품질 & 소스 섹션은 반드시 리포트 말미**. TL;DR이 최상단. (자세한 순서는 "보고서 템플릿" 섹션 참조)
6. **`Agent` tool로 readability-pass subagent 1회** (순차 실행, 원본 저장 완료 후) — 완성된 리포트를 입력받아 주식 입문자용 `-쉬운버전.md`를 같은 디렉토리에 생성. 자세한 프롬프트는 "Readability-pass subagent" 섹션 참조
7. 사용자에게 **원본 + 쉬운버전 두 저장 경로 + TL;DR**만 짧게 보고 — 본문 전체를 채팅에 다시 붙여넣지 마세요

## 데이터 소스 역할 분담 (중요)

**실제 검증된 결과**에 기반한 역할 분담입니다. 각 소스의 강점을 살려 조합하세요.

### 📊 시세(숫자) = MCP를 주력으로 사용
**원칙: 가격/변동률은 반드시 MCP에서 가져온다. 뉴스 기사에서 숫자를 긁어오는 건 마지막 수단이다.** 실제 테스트에서 뉴스 기사의 종가와 MCP 실시간 데이터가 달랐던 사례가 있었습니다 — MCP가 진실입니다.

#### 🥇 `mcp__finnhub__finnhub_stock_market_data` (operation=`get_quote`) — **주력**
✅ 검증 완료 (Iteration 2 테스트): SPY, QQQ, DIA, IWM, XLK, XLE, XLV, USO, GLD, UUP, NVDA, AAPL, TSLA, MSFT 등 **모든 US-listed ETF/Stock 작동**.

응답 필드:
- `c` = current price (종가)
- `d` = change (변동)
- `dp` = change percent (변동률 %)
- `h` = day high, `l` = day low, `o` = day open, `pc` = previous close

#### 🥈 `mcp__alphavantage__get_forex_rate` — 환율 전용
✅ 검증 완료. USD/KRW, USD/JPY 등 환율. 무제한.

#### ⚠️ `mcp__alphavantage__get_stock_price` — 백업
작동하지만 **무료 티어 하루 25회 제한**. finnhub이 실패한 특정 티커에만 선별적으로 사용.

#### ❌ `mcp__yfinance__getStockAnalysis` — 사용 금지
현재 환경에서 모든 티커를 `¥0 / 데이터 없음`으로 반환. 호출 자체가 시간 낭비. **절대 쓰지 마세요**.

#### ❌ finnhub `^VIX`, `^TNX` 등 지수 심볼
"CFD subscription required" 에러. 지수 심볼 대신 **ETF 프록시** 또는 **WebSearch**로 우회.

### 📰 뉴스·매크로 = Tavily + WebSearch **반드시 병렬로**
이 skill은 **Tavily MCP를 설치 전제**로 설계되어 있습니다. Tavily와 WebSearch는 서로를 대체하는 것이 아니라 **교차검증용으로 함께 사용**합니다. Tavily는 금융 뉴스 원문 추출(extract)에 특화되어 있어 WebSearch가 놓치는 원문 세부 정보를 보완합니다.

#### 🥇 Tavily MCP — **필수 도구**
- 도구 이름: `tavily_search`, `tavily_extract` (정확한 이름은 세션 시작 시 `ToolSearch`의 `select:tavily_search,tavily_extract` 또는 `query: "tavily"` 로 확인)
- 사용법:
  - `tavily_search`: 여러 쿼리를 병렬 호출해 금융 뉴스·매크로·지정학 정보 수집
  - `tavily_extract`: 핵심 기사 URL을 통째로 원문 추출 (WebSearch 스니펫보다 정확)
- **반드시 WebSearch와 같은 메시지 안에서 병렬 호출**해 교차검증

#### 🥇 `WebSearch` — **필수 도구**
매크로 이벤트, 지정학, 경제지표 발표, 기업 실적 서프라이즈 등 뉴스성 정보의 기본 소스. Tavily와 병렬로 실행.

**⚠️ 쿼리 작성 규칙**: WebSearch는 연/월을 명시하지 않으면 과거 자료를 반환할 수 있음. **모든 쿼리에 현재 연/월 포함**.

#### Tavily가 세션에 로드되지 않은 경우
`ToolSearch`로 `tavily_search`를 찾을 수 없으면 Tavily MCP가 이 세션에서 아직 활성화되지 않았다는 뜻입니다 (설치 직후 Claude Code 재시작 전일 수 있음). 이 경우:
1. 사용자에게 "Tavily MCP가 이 세션에 로드되지 않아 WebSearch만으로 뉴스를 수집합니다"라고 고지
2. WebSearch 쿼리를 평소보다 **2배 더 다양한 각도**로 실행해 Tavily 부재를 보완
3. 보고서 상단 "데이터 품질"에 "Tavily 미로드" 명시

#### ⚠️ `mcp__finnhub__finnhub_news_sentiment` — subagent 경유만
응답이 75KB 초과로 컨텍스트 한도를 초과한 적 있음. 메인 context에 직접 호출 금지. 필요 시 `Agent` tool로 위임해 "Top 5 헤드라인 + 1줄 요약 + URL"만 받아오세요.

#### ❌ `mcp__finnhub__finnhub_calendar_data`
API key 이슈로 실패. 경제지표 캘린더는 WebSearch로 대체.

### 일반 원칙
- **실패 허용**: 어떤 도구가 실패해도 전체를 중단하지 말고 다른 소스로 대체. "데이터 품질" 섹션에 실패 내역 명시.
- **병렬 원칙**: 독립적인 호출은 **반드시 한 메시지에 동시 실행**. 섹션당 10~20개 티커를 묶으면 전체 수집이 한 번에 끝납니다.

## 오케스트레이션 구조 (하이브리드)

이 skill은 **메인 orchestrator + 뉴스 harvester subagent** 하이브리드 구조로 동작합니다. 모든 작업을 subagent에 분산시키면 오버헤드가 크고 cross-domain 인과 분석이 불가능해지므로, **맥락이 필요한 압축 데이터는 메인에서 직접 병렬 호출**하고 **대용량·비정형 뉴스 정보는 subagent로 격리**합니다.

### 🧠 메인 Orchestrator가 직접 수행 (context에 남아야 분석 가능)
- 날짜·경로 계산
- 시세 MCP 병렬 호출 — `finnhub get_quote` × 20~34 티커 (지수 ETF, 섹터 ETF, Mag7, 원자재 ETF, 글로벌 ETF)
- 환율 MCP — `alphavantage get_forex_rate` × 2 (USD/KRW, USD/JPY)
- 짧은 매크로 `WebSearch` 1~3개 (VIX, 10Y 금리 같은 보조 수치)
- 최종 인과 분석 + 섹터↔원자재↔뉴스 cross-domain 연결
- 보고서 작성·저장

이유: 이 호출들은 응답이 작고(<1KB) 병렬로 가기 때문에 subagent 오버헤드가 더 큼. 또 **모든 데이터가 한 context에 있어야** 섹터 로테이션과 뉴스 헤드라인을 엮을 수 있음.

### 🤖 Subagent(news-harvester)에 위임
- `tavily_search` (여러 쿼리 동시)
- `tavily_extract` (긴 기사 원문 추출)
- `mcp__finnhub__finnhub_news_sentiment` (75KB+ 대용량 응답)
- `WebSearch` (뉴스/매크로 교차검증)
- 종목 급등락 원인 deep-dive 리서치

이유: 이 호출들은 응답이 수십 KB급으로 커서 메인 context에 부으면 분석 품질이 급격히 떨어짐. Subagent는 원문을 다 읽고 **압축된 요약만 반환**하므로 메인 context는 깨끗하게 유지됨.

### 🔧 News-harvester subagent 호출 방법

`Agent` tool로 subagent를 띄웁니다 (`subagent_type`은 `general-purpose`로 충분). 프롬프트 템플릿

```
역할: 오늘({YYYY-MM-DD})의 미국 증시·매크로·지정학 뉴스 harvester.
기준 세션: {전일 NY 정규장 마감 날짜}.

수집 단계:
1. tavily_search (ToolSearch로 tavily_search 찾기 시도, 없으면 WebSearch만):
   - "Federal Reserve FOMC rate decision {Month} {YYYY}"
   - "US stock market close {Month} {D} {YYYY} sector performance"
   - "CPI PCE jobs report {Month} {YYYY}"
   - "Middle East Iran Ukraine Taiwan {Month} {YYYY} market impact"
   - "earnings surprise {Month} {D} {YYYY}"
2. WebSearch로 동일 주제 교차검증 (각 쿼리에 현재 연/월 반드시 포함)
3. (선택) mcp__finnhub__finnhub_news_sentiment로 센티먼트 확인 — 응답이 크면 주요 5개 헤드라인만 추출
4. 핵심 기사 3~5개 URL은 tavily_extract로 원문 통째 추출해 주요 인용·숫자 확인

위 호출들은 반드시 **한 메시지 안에서 병렬로** 실행하세요.

반환 형식 (반드시 준수, 전체 응답 800단어 이하):

## 오늘의 Top 10 헤드라인
- [1줄 요약 + URL] × 10

## 카테고리별 임팩트
- 연준/금리:
- 인플레/경제지표:
- 지정학/중동/우크라이나:
- 기업 실적/가이던스:
- 원자재/에너지:

## 시장 방향성 추론
- (2~3줄, 왜 이런 흐름인지)

## 주요 출처
- (핵심 URL 5~10개)

중요: 원문을 직접 복붙하지 말고 반드시 요약. 응답 전체가 메인 context에 포함되므로 간결함이 핵심. 불확실한 인과는 "~로 보임"처럼 약하게 표현.
```

### 🛡 Fault tolerance
news-harvester가 실패하거나 빈 응답을 반환하면, 메인 orchestrator가 **fallback으로 WebSearch 3~5개만 직접 실행**해 핵심 뉴스를 확보하고 "데이터 품질" 섹션에 `news-harvester 실패, WebSearch fallback`을 명시합니다. **시세 수집과 뉴스 수집이 독립적이므로 뉴스가 실패해도 리포트는 완성됩니다.**

### 🎨 Readability-pass subagent (후처리, 순차 실행)

원본 리포트가 디스크에 저장된 **뒤에** 별도 subagent를 1회 실행해 같은 디렉토리에 `-쉬운버전.md` 파일을 생성합니다. 이 단계는 수집/분석 블록과 **독립**이며 원본이 완성된 뒤에만 돌기 때문에 병렬이 아니라 **순차 실행**입니다.

**왜 필요한가**: 원본 리포트는 실전 트레이딩 관점에서 디버전스·섹터 로테이션·FedWatch 같은 전문용어가 많습니다. 주식을 처음 접하는 독자에게는 이 어휘 장벽이 진입을 막기 때문에, 같은 내용을 **주식 입문자가 이해할 수 있는 언어로 리라이팅한 별도 파일**을 남겨 독자가 용도에 따라 두 버전을 선택해 읽을 수 있게 합니다.

**원본은 손대지 않음**: readability-pass는 원본 파일을 읽기만 하고 수정하지 않습니다. 전문가용 원본과 입문자용 쉬운버전이 **같은 디렉토리에 쌍으로** 공존합니다.

**호출 방법**: `Agent` tool (`subagent_type: general-purpose`)로 아래 프롬프트를 전달. `{원본 절대경로}` 부분만 실제 경로로 치환.

```
역할: 완성된 야간 미국 증시 브리핑 보고서를 **주식 투자 입문자**가 이해할 수 있게 리라이팅하는 readability pass.

## 입력
- 원본 리포트 절대경로: `{원본 절대경로}`
- 출력 경로: 같은 디렉토리에 파일명 + `-쉬운버전.md` suffix (예: `2026-04-11-1210.md` → `2026-04-11-1210-쉬운버전.md`)

## 작업 절차
1. Read 도구로 원본 리포트 전체를 읽는다.
2. 아래 "작업 원칙"에 따라 입문자용으로 리라이팅한다.
3. Write 도구로 `-쉬운버전.md`에 저장한다.
4. 한 줄로 `쉬운버전 저장 완료: <절대경로>`만 반환. **본문을 다시 복사해 반환 금지.**

## 작업 원칙 (엄수)

### 반드시 지킬 것
1. **모든 숫자·티커·날짜는 원본 그대로 유지**. 종가, 변동률, 금리, 환율, 섹터 변동률 등 어떤 수치도 변경·추정·생략 금지. 원본이 진실.
2. **구조(섹션 제목과 순서)는 원본과 동일**. TL;DR → 3대 지수 → 섹터 → 주요 종목 → 원자재/금리/환율 → 글로벌 → 국제정세 → 한국장 관전 포인트 → 데이터 품질 & 소스. 독자가 두 버전을 대조해 읽을 수 있어야 함.
3. **인과 해석 결론은 보존**, 어휘만 쉬운 말로. 원본의 "왜"는 같아야 함.
4. **URL과 출처 블록은 그대로 복사**.

### 전문용어 처리 룰
한 섹션에서 **처음 등장할 때 괄호 안에 짧은 해설**을 붙이고, 같은 섹션 안에서는 해설 없이 원어만 사용. 예:

- 디버전스 → "디버전스(서로 다른 방향으로 움직이는 현상)"
- 섹터 로테이션 → "섹터 로테이션(투자금이 업종 간에 이동하는 흐름)"
- 방어주 → "방어주(경기와 무관하게 꾸준한 필수소비재·헬스케어·유틸리티 등)"
- 경기민감주 → "경기민감주(경기 좋을 때 잘 오르는 산업재·소재·금융 등)"
- CPI → "CPI(소비자물가지수 — 생활물가가 얼마나 올랐는지 보여주는 지표)"
- 근원 CPI → "근원 CPI(에너지·식품을 제외한 물가지수, 추세 판단용)"
- PCE → "PCE(개인소비지출물가지수 — 연준이 가장 중시하는 물가 지표)"
- FOMC → "FOMC(연방공개시장위원회 — 미국 기준금리를 정하는 회의)"
- CME FedWatch → "CME FedWatch(시장이 예상하는 금리 방향을 확률로 보여주는 지표)"
- 10년물 국채금리 → "10년물 국채금리(미국 정부가 10년간 돈을 빌릴 때 내는 이자, 장기금리 기준)"
- GDPNow → "GDPNow(애틀랜타 연준이 실시간 추정하는 성장률)"
- VIX → "VIX(시장의 공포·변동성을 수치화한 지수, 20 아래면 안정적)"
- ETF → "ETF(여러 종목을 묶어 주식처럼 거래하는 상품)"
- 프록시 → "프록시(대리로 보는 지표)"
- Mag7 / Magnificent 7 → "Mag7(빅테크 7인방: Apple·Microsoft·NVIDIA·Google·Amazon·Meta·Tesla)"
- 밸류에이션 → "밸류에이션(주가가 기업가치 대비 비싼지 싼지 평가)"
- NIM → "NIM(예대금리차 — 은행이 대출로 받는 이자와 예금으로 주는 이자의 차이)"
- capex → "capex(설비투자)"
- 차익실현 → "차익실현(오른 종목을 팔아 수익을 확정하는 행위)"
- 테일리스크 → "테일리스크(가능성은 낮지만 터지면 충격이 큰 위험)"
- 스태그플레이션 → "스태그플레이션(경기는 멈추고 물가만 오르는 상황)"
- 헤드라인 쇼크 → "헤드라인 쇼크(표면 수치가 예상보다 나쁘게 나와 시장을 놀라게 하는 것)"
- 호르무즈 해협 → "호르무즈 해협(세계 원유 수송의 약 20%가 지나가는 이란 인근 해협)"
- HBM → "HBM(고대역폭 메모리, AI 가속기에 들어가는 고성능 D램)"
- SaaS → "SaaS(구독형 소프트웨어 사업, 예: Salesforce, ServiceNow)"
- 밸류체인 → "밸류체인(제품이 만들어지는 모든 연관 기업 사슬)"
- 강보합 / 약보합 → "강보합(살짝 오른 채 마감)" / "약보합(살짝 내린 채 마감)"
- 컨센서스 → "컨센서스(애널리스트들의 평균 예상치)"
- 가이던스 → "가이던스(기업이 스스로 제시한 실적 전망)"
- 다운그레이드 → "다운그레이드(투자의견 하향)"

목록에 없는 전문용어가 나오면 같은 방식으로 한 번만 괄호 해설.

### 문장 스타일
- "~임", "~됨" 같은 딱딱한 종결은 "~입니다"/"~했습니다"로
- 영어 약자가 처음 나올 때는 한글 풀네임 병기
- 인과는 "A 때문에 B가 이렇게 움직였습니다" 같은 단순 구조로

### 표 처리
시세 표는 그대로 복사하되, 각 표 바로 아래에 1~2줄 "쉬운 해석" 추가. 예시:
> Dow Jones(DIA)는 미국 전통 대기업 30곳을 묶은 지수입니다. 이 날 -0.55%로 3대 지수 중 가장 약했습니다.

### TL;DR은 특히 쉽게
원본 TL;DR을 기반으로 하되 문장을 더 풀어써도 OK (최대 10줄). 가장 중요한 것: **오늘 미국 장이 왜 그렇게 움직였는지**가 초보에게도 한눈에 들어와야 함.

### 필수: 상단에 미니 용어집 (TL;DR 바로 위)
파일 **최상단(헤더 메타 블록 바로 아래, TL;DR 직전)**에 `## 📘 이 리포트를 처음 보시나요? 미니 용어집` 섹션을 넣을 것. 본문에서 가장 중요한 5~10개 용어를 "용어명 — 한 줄 풀이" 형태로 정리해 독자가 본문을 읽기 전에 훑어볼 수 있게 합니다. 용어집이 있어야 TL;DR부터 바로 이해가 시작됨. 말미 배치는 금지 (원본 리포트 iteration 3에서 사용자 피드백으로 확정).

## 금지 사항
- 숫자·티커 만들어내기 금지
- 원본에 없는 해석·의견·종목 추천 추가 금지
- 섹션 삭제 또는 순서 변경 금지
- URL 임의 변경 금지
- 투자 조언 성격 문장 추가 금지 (원본의 면책 문구는 그대로 유지)

## 반환
파일을 쓴 뒤 **한 줄로만** `쉬운버전 저장 완료: <절대경로>` 형태로 반환. 본문 재출력 절대 금지.
```

**Fault tolerance**: readability-pass가 실패해도 원본 리포트는 이미 저장돼 있으므로 전체를 중단하지 말고 "쉬운버전 생성 실패" 한 줄만 사용자에게 고지하세요.

## 수집 블록별 권장 접근

### 1) 미국 3대 지수 + 변동성 — ETF 프록시 사용
**지수 심볼(^GSPC, ^IXIC, ^DJI)은 finnhub에서 막히므로 ETF로 우회**.

| 지수 | ETF 프록시 | finnhub 도구 |
|---|---|---|
| S&P 500 | **SPY** | `get_quote` ✅ |
| Nasdaq 100 | **QQQ** | `get_quote` ✅ |
| Dow Jones | **DIA** | `get_quote` ✅ |
| Russell 2000 | **IWM** | `get_quote` ✅ |
| VIX | — | WebSearch 또는 `VIXY` ETF 프록시 |

지수 포인트(6,816.89 같은 값)가 필요하면 WebSearch로 보완. ETF 종가/변동률만으로도 시장 방향성은 100% 파악 가능.

### 2) 섹터 ETF — 전부 finnhub
XLK(Tech), XLF(Financials), XLE(Energy), XLV(Healthcare), XLY(Consumer Discretionary), XLP(Staples), XLI(Industrials), XLU(Utilities), XLRE(Real Estate), XLB(Materials), XLC(Communication).

**한 메시지에 11개 ETF를 finnhub `get_quote`로 병렬 호출**하고 `dp`(변동률)로 정렬 → 상위 3 / 하위 3 섹터 식별 → 왜 그 방향인지 원자재·금리·뉴스와 연결.

### 3) 주요 종목 — 전부 finnhub
기본: AAPL, MSFT, NVDA, GOOGL, AMZN, META, TSLA
추가(선택): AMD, AVGO, TSM, NFLX

한 메시지에 병렬 호출. 크게 움직인 종목은 WebSearch/Tavily로 `"{ticker} stock news {Month} {D} {YYYY}"` 추가 조사.

### 4) 원자재 · 금리 · 환율
| 항목 | 방법 |
|---|---|
| WTI 유가 | **USO** ETF → finnhub `get_quote` ✅ |
| 금 | **GLD** ETF → finnhub `get_quote` ✅ |
| 달러인덱스 | **UUP** ETF → finnhub `get_quote` ✅ |
| 美 10년물 금리 | WebSearch (`"10 year treasury yield {Month} {D} {YYYY}"`) 또는 **IEF**/**TLT** ETF 프록시 |
| USD/KRW | **`mcp__alphavantage__get_forex_rate`** ✅ (from=USD, to=KRW) |
| USD/JPY | **`mcp__alphavantage__get_forex_rate`** ✅ (from=USD, to=JPY) |

### 5) 글로벌 증시 — ETF 프록시 또는 WebSearch
finnhub 무료 티어는 해외 지수 심볼(^N225 등)이 막힙니다. 방법:

| 시장 | 방법 A (ETF 프록시, finnhub) | 방법 B (WebSearch) |
|---|---|---|
| 일본 Nikkei | **EWJ** | `"Nikkei 225 close {Month} {D} {YYYY}"` |
| 영국 FTSE | **EWU** | `"FTSE 100 close {Month} {D} {YYYY}"` |
| 독일 DAX | **EWG** | `"DAX close {Month} {D} {YYYY}"` |
| 홍콩 HSI | **EWH** | `"Hang Seng close {Month} {D} {YYYY}"` |
| 한국 KOSPI | **EWY** | `"KOSPI close {Month} {D} {YYYY}"` |

ETF는 현지 증시 종가를 정확히 반영하진 않지만 방향성 신호로는 충분. 정확한 포인트가 필요하면 WebSearch 병행.

### 6) 뉴스 · 이벤트 (매크로 + 지정학) — **subagent 위임**

이 블록은 메인 orchestrator가 직접 처리하지 **않습니다**. 대신 **"오케스트레이션 구조" 섹션의 news-harvester subagent 템플릿**을 사용해 `Agent` tool로 위임하세요. 이유:

- Tavily extract, finnhub news_sentiment 같은 대용량 응답을 메인 context에 쏟지 않음
- 메인은 압축된 "Top 10 헤드라인 + 카테고리별 임팩트" 요약만 받음
- 뉴스 수집이 실패해도 시세 기반 리포트는 정상 완성 (fault isolation)

실행 타이밍: 시세 수집 블록(1~5)과 **동시에** subagent를 띄우세요. 즉 **한 메시지 안에서 finnhub 시세 호출 ×20~30 + Agent(news-harvester) 1개를 병렬** 실행하면, 리포트 작성 단계에 진입할 때 시세와 뉴스가 모두 준비되어 있습니다.

subagent가 빈 응답이나 오류를 내면 메인 orchestrator가 `WebSearch` 3~5개로 fallback하세요. 쿼리 예시는 news-harvester 템플릿과 동일합니다.

## 분석: 인과 연결

데이터를 모았다면 **"왜"**를 써야 합니다. 단순 나열과 분석의 차이는 인과입니다.

- ❌ "XLE -0.68%"
- ✅ "XLE -0.68% — USO(WTI) -1.69%와 연동. 이란 휴전 기대 뉴스가 유가 차익실현을 유발"

체크리스트:
- 섹터 로테이션을 설명할 매크로 이벤트가 있는가?
- 개별 종목 급등락 뒤에 실적/뉴스/애널리스트 코멘트가 있는가?
- 금리·달러·원자재 변동이 어떤 섹터에 반영됐는가? (예: 금리↑ → 성장주·REITs 압박)
- 지정학 이슈가 에너지·방산·반도체 같은 특정 섹터에 영향을 줬는가?
- 한국 시장과 밸류체인이 얽힌 섹터(반도체, 자동차, 2차전지)는 어떻게 해석해야 하는가?

**불확실성 표기**: 인과가 명확하지 않으면 "~와 관련된 것으로 보임", "~가 원인 가능성"처럼 약하게 표현하고 단정을 피하세요. 거짓 확신은 이 skill의 가치를 망칩니다.

## 보고서 템플릿

아래 구조를 뼈대로 사용하세요. 빈 섹션은 생략 대신 "특이사항 없음"으로 명시해 독자가 **"데이터가 없는 것"과 "조사 안 된 것"을 구분**할 수 있게 합니다.

```markdown
# {YYYY-MM-DD} 야간 미국 증시 & 국제정세 브리핑

> 생성 시각(KST): {HH:MM}
> 기준 세션: {전일 NY 정규장 마감 날짜}

## 핵심 요약 (TL;DR)
- (3~5줄, 오늘 가장 중요한 것만)

## 미국 3대 지수
| 지수 (ETF 프록시) | 종가 | 변동 | 변동률 |
|---|---|---|---|
| S&P 500 (SPY) |  |  |  |
| Nasdaq 100 (QQQ) |  |  |  |
| Dow Jones (DIA) |  |  |  |
| Russell 2000 (IWM) |  |  |  |
| VIX |  |  |  |

**해석**: (지수 움직임의 맥락과 원인)

## 섹터 동향 (일간 변동률, finnhub get_quote)
| 섹터 | ETF | 변동률 |
|---|---|---|
| 강세 1~3위 |  |  |
| 약세 1~3위 |  |  |

**로테이션 분석**: (왜 이런 방향성이 나왔는지)

## 주요 종목
| 티커 | 종가 | 변동률 | 트리거 |
|---|---|---|---|
|  |  |  |  |

## 원자재 · 금리 · 환율
- **WTI 유가 (USO)**:
- **금 (GLD)**:
- **달러인덱스 (UUP)**:
- **美 10년물 금리**:
- **USD/KRW**:
- **USD/JPY**:

## 글로벌 증시
- **유럽 마감**: FTSE / DAX / CAC (또는 EWU / EWG 프록시)
- **아시아 전일**: Nikkei / HSI / KOSPI (또는 EWJ / EWH / EWY 프록시)

## 국제정세 & 매크로 이벤트
- (연준 / 경제지표 / 지정학 / 원자재 이벤트를 항목별로)

## 오늘 한국장 관전 포인트
- (위 데이터를 바탕으로 한국 투자자가 주목할 포인트 3~5개)

## 데이터 품질 & 소스
> 이 섹션은 **보고서 말미에 배치**합니다. 독자가 원하는 것은 시장 브리핑이지 수집 메타정보가 아닙니다. 데이터 품질·도구 상태·충돌 해소 내역은 각주/부록 성격이므로 TL;DR 위로 올리지 마세요.

### 수집 요약
- 수집 성공률: {X}/{전체 섹션 수} 섹션 완전 확보
- 주력 소스: finnhub get_quote (시세) / alphavantage forex (환율) / Tavily·WebSearch (뉴스)
- 실패/부분 확보: {섹션 이름 + 사유}

### 데이터 충돌 해소 내역
- (있을 때만: 예 — subagent 인용 WTI +3.78% vs MCP USO -1.69% → 원칙대로 MCP 채택)

### 사용한 도구
- ✅/❌ 각 MCP·도구별 성공/실패 표시

### 주요 링크
- (인용한 뉴스 URL)
```

## 주의사항

- **시세는 MCP가 진실**: 뉴스 기사에서 추출한 종가와 finnhub 실시간 데이터가 다른 경우가 실제로 있었습니다 (예: NVDA 뉴스 $183 vs finnhub $188.63). 시세는 반드시 MCP 값을 인용하세요.
- **시간대 기준**: 한국 시간 기준 "어제 밤"은 미국 뉴욕 정규장 종료(KST 익일 새벽 5시경). 보고서 제목은 **한국 날짜**, 기준 세션은 **전일 NY 마감 날짜**.
- **실패 허용**: 특정 도구 실패 시 전체 중단 금지. 대체 소스로 보고서 완성 후 "데이터 품질"에 실패 내역 명시. `yfinance` 같은 **알려진 실패 도구는 호출 자체를 생략**하는 것이 효율적.
- **숫자의 진실성**: 모든 숫자는 이번 실행의 실제 도구 호출 값. 기억/추정 금지. 구하지 못하면 `N/A`.
- **대용량 응답**: `finnhub news_sentiment` 등 대용량 응답 도구는 `Agent` 경유로만 호출해 요약본만 받으세요.
- **뉴스 인용**: WebSearch/Tavily 결과는 출처 URL을 "데이터 소스"에 남기세요.
- **덮어쓰기 금지**: 같은 날짜 파일이 있으면 `YYYY-MM-DD-HHMM.md`로 새 파일.
- **보고 길이**: 사용자 채팅창에는 저장 경로 + TL;DR만. 본문 전체를 다시 붙여넣지 마세요.
- **투자 조언 아님**: 정보 제공 목적이며 매수/매도 권유가 아니라는 면책 문구를 말미에.
- **연/월 명시**: WebSearch 쿼리에 반드시 현재 연/월 포함.
