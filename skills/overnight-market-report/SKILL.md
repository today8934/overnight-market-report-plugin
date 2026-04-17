---
name: overnight-market-report
description: 간밤 미국 증시(지수·섹터·주요 종목)와 국제정세를 finnhub·alphavantage·Tavily·WebSearch로 동시 수집·분석해 한국장 개장 전 브리핑용 한국어 마크다운 보고서를 생성합니다. 사용자가 "미국주식", "야간시황", "overnight-market-report", "밤사이 증시", "미장 마감", "간밤 뉴욕", "미국 증시 브리핑", "전일 미국장", "어제 미장 어땠어", "밤사이 무슨 일 있었어" 같은 직·간접 표현을 쓸 때 반드시 이 skill을 실행하세요. 단순 가격 조회가 아니라 지수·섹터·매크로·지정학 이벤트를 엮어 인과를 설명하는 종합 리포트가 필요한 모든 상황에서 트리거합니다.
---

# Overnight Market Report

간밤 미국 증시와 국제정세를 병렬 수집·분석해 한국장 개장 전 브리핑용 한국어 마크다운 보고서를 만듭니다.

## 왜 이 skill이 필요한가

한국 투자자는 개장 전에 미국 증시 마감 결과만이 아니라 **왜 그렇게 움직였는지**를 알고 싶어합니다. 지수 → 섹터 → 주요 종목 → 매크로/지정학 이벤트의 인과를 연결해야 실전에 쓸 수 있는 브리핑이 됩니다.

## 참조 문서 (Progressive disclosure)

자세한 내용은 필요할 때만 `Read`로 로드하세요. 메인 세션 context를 아끼기 위한 분할입니다.

- `references/setup-wizard.md` — preflight-check subagent 프롬프트 + Setup Wizard 6단계 + **preflight 24h 캐싱 규칙**
- `references/data-sources.md` — MCP 도구별 ✅/⚠️/❌ 매트릭스 + Tavily 미로드 시 WebSearch 쿼리 쌍
- `references/theme-tickers.md` — **3-Tier 티커 유니버스** (Always-on / 테마 트리거 / 이벤트 트리거)
- `references/glossary.md` — readability-pass 전용 용어 사전

## 출력 위치

`~/workspace/overnight-market-report/YYYY-MM-DD.md` (오늘 한국 날짜 기준)

디렉토리가 없으면 먼저 생성합니다. 같은 날짜 파일이 이미 존재하면 덮어쓰지 말고 `YYYY-MM-DD-HHMM.md` 형태로 시간 suffix를 붙여 새 파일을 만드세요.

## 실행 순서

0. **Preflight (캐시 우선)** — `~/.claude/data/overnight-market-report/preflight.json`을 읽어 `last_ok_at`이 24h 이내 + 모든 checks가 `"ok"`면 **skip**하고 Step 1로. 캐시 miss / TTL 초과 / 지난 실행에서 401 감지 시 `references/setup-wizard.md`를 Read해 full preflight 실행. `needs_setup` 반환 시 Setup Wizard 진행 후 halt (메인 워크플로우 진입 금지).
1. **오늘 한국 날짜·시각(KST) 확인 + 기준 세션 결정** (아래 "기준 세션 결정 로직" 참조) → 파일 경로 결정
2. **출력 디렉토리 확인/생성** + 직전 파일 Read (전일 diff 섹션 재료 확보)
3. **한 메시지에서 병렬 실행** (하이브리드 오케스트레이션):
   - Tier 1 시세 MCP 호출 × 43 (finnhub `get_quote`) — `references/theme-tickers.md`의 Tier 1 리스트
   - 환율 MCP × 2 (alphavantage USD/KRW, USD/JPY)
   - 보조 WebSearch 1~3 (VIX/10Y 등 지수 수치 보완)
   - **`Agent` tool로 news-harvester subagent 1회** (뉴스 수집 위임)
4. **Tier 2/3 조건부 2차 배치** — news-harvester 응답 수령 후 카테고리 키워드를 `references/theme-tickers.md` Tier 2/3 매핑과 substring 매칭해 매칭된 테마의 티커를 **2차 finnhub 병렬 호출** (최대 15개 cap, Tier 1 중복은 제외).
5. **Sanity check 게이트** (아래 "Sanity check" 참조) — 통과 시 Step 6, 실패 시 리포트 상단에 경고 배너를 추가한 뒤 계속.
6. **보고서 작성·저장** — 아래 "보고서 템플릿" 구조 그대로. YAML frontmatter → TL;DR → 3대 지수 → 섹터 → 주요 종목 → 테마별 트리거 종목(Tier 2/3이 활성화된 경우만) → 원자재/금리/환율 → 글로벌 → 국제정세 → 한국장 관전 포인트 → 전일 대비 변화 → 면책 → 데이터 품질 & 소스(말미). 인과 주장에는 inline `[[n]](url)` 출처 링크 필수.
7. **Preflight 캐시 갱신** — 전체 실행이 문제없이 완료되면 `preflight.json`의 `last_ok_at`을 현재 KST ISO8601로 갱신.
8. **`Agent` tool로 readability-pass subagent 1회** (순차 실행) — 완성된 리포트를 입력받아 `-쉬운버전.md` 생성. 자세한 프롬프트는 아래 "Readability-pass subagent" 섹션 참조.
9. 사용자에게 **원본 + 쉬운버전 저장 경로 + TL;DR**만 짧게 보고 — 본문 전체를 채팅에 다시 붙여넣지 마세요.

## 기준 세션 결정 로직

**"기준 세션"** = 보고서가 분석 대상으로 삼는 **직전 NY 정규장 마감일**. 한국 시각 기준 판정:

| 한국 요일 (실행 시점) | 기준 세션 |
|---|---|
| 화~금 아침 | 전날 (월~목) NY 마감 |
| 토요일 전일 | 금요일 NY 마감 |
| 일요일 | 금요일 NY 마감 (주말 미장은 휴장) |
| 월요일 아침 | 금요일 NY 마감 |

### US 주요 공휴일 (NY 정규장 휴장)
- New Year's Day, MLK Day(1월 셋째 월), Presidents Day(2월 셋째 월), Good Friday, Memorial Day(5월 마지막 월), Juneteenth(6/19), Independence Day(7/4), Labor Day(9월 첫째 월), Thanksgiving(11월 넷째 목) + 다음날 조기 폐장, Christmas Day
- 반일장: Black Friday, 12/24

### 판정 절차
1. 후보 기준 세션 날짜(위 표)를 먼저 계산
2. 그 날짜가 US 공휴일 체크리스트에 해당하면 **하루 더 앞당김** (재귀적으로 영업일 도달까지)
3. 확실하지 않으면 news-harvester에 `"US stock market {Month} {D} {YYYY} closed holiday"` 쿼리 추가해 교차검증
4. 결정된 기준 세션을 YAML frontmatter `session_date`에 기록

## 오케스트레이션 구조 (하이브리드)

**메인 orchestrator + 뉴스 harvester subagent** 하이브리드. 응답이 작고 cross-domain 분석이 필요한 호출은 메인이 직접 병렬 처리하고, 대용량 뉴스는 subagent로 격리합니다.

### 🧠 메인 Orchestrator가 직접 수행
- 날짜·경로·기준 세션 계산
- Tier 1 시세 MCP 병렬 호출 (`references/theme-tickers.md` Tier 1 리스트)
- 환율 MCP (alphavantage `get_forex_rate` × 2)
- 짧은 매크로 WebSearch 1~3개 (VIX, 10Y 금리 보완)
- Tier 2/3 조건부 2차 배치
- 최종 인과 분석 + 섹터↔원자재↔뉴스 cross-domain 연결
- 보고서 작성·저장

### 🤖 news-harvester subagent에 위임
- `tavily_search` (여러 쿼리 동시)
- `tavily_extract` (긴 기사 원문 추출)
- `mcp__finnhub__finnhub_news_sentiment` (75KB+ 대용량 응답)
- `WebSearch` (뉴스/매크로 교차검증)

### 🔧 News-harvester subagent 호출 프롬프트

`Agent` tool (`subagent_type: general-purpose`)로 전달:

```
역할: 오늘({YYYY-MM-DD})의 미국 증시·매크로·지정학 뉴스 harvester.
기준 세션: {전일 NY 정규장 마감 날짜}.

수집 단계 (한 메시지에서 병렬 실행):
1. tavily_search (없으면 WebSearch만):
   - "Federal Reserve FOMC rate decision {Month} {YYYY}"
   - "US stock market close {Month} {D} {YYYY} sector performance"
   - "CPI PCE jobs report {Month} {YYYY}"
   - "Middle East Iran Ukraine Taiwan {Month} {YYYY} market impact"
   - "earnings surprise {Month} {D} {YYYY}"
   - "OPEC oil production {Month} {YYYY}"
2. WebSearch로 동일 주제 교차검증 (각 쿼리에 현재 연/월 반드시 포함)
3. (선택) mcp__finnhub__finnhub_news_sentiment로 센티먼트 확인 — 응답이 크면 주요 5개 헤드라인만 추출
4. 핵심 기사 3~5개 URL은 tavily_extract로 원문 통째 추출해 주요 인용·숫자 확인

반환 형식 (반드시 준수, 전체 응답 800단어 이하):

## 오늘의 Top 10 헤드라인
- [1줄 요약 + URL] × 10

## 카테고리별 임팩트
(각 bullet의 표현을 Tier 2/3 매핑에 쓸 것이므로 관련 키워드 원어 유지)
- 연준/금리:
- 인플레/경제지표:
- 지정학/중동/우크라이나:
- 기업 실적/가이던스:
- 원자재/에너지 (OPEC 포함):
- 중국/관세:
- 크립토/리스크 심리:
- AI/반도체/전력:

## 시장 방향성 추론
- (2~3줄, 왜 이런 흐름인지)

## 주요 출처
- (핵심 URL 5~10개, 번호 붙여 — 메인이 [[n]](url) inline 링크로 사용)

중요: 원문을 직접 복붙하지 말고 반드시 요약. 응답 전체가 메인 context에 포함되므로 간결함이 핵심. 불확실한 인과는 "~로 보임"처럼 약하게 표현.
```

### 🛡 Fault tolerance
- news-harvester 실패 → 메인이 WebSearch 3~5개 직접 실행. 시세 리포트는 정상 완성 (fault isolation)
- Tier 2/3 활성화 실패 → 무시하고 Tier 1만으로 작성 (데이터 품질에 한 줄 명시)
- finnhub 일부 티커 실패 → 해당 티커만 `N/A`로 표시, 섹션 전체는 유지

### 🎨 Readability-pass subagent (순차 후처리)

원본 리포트 저장 **뒤** 별도 subagent 1회 실행 — `-쉬운버전.md` 생성. `Agent` tool (`subagent_type: general-purpose`)로 아래 프롬프트 전달, `{원본 절대경로}`만 치환:

```
역할: 완성된 야간 미국 증시 브리핑 보고서를 **주식 투자 입문자**가 이해할 수 있게 리라이팅하는 readability pass.

## 입력
- 원본 리포트 절대경로: `{원본 절대경로}`
- 용어집 절대경로: `{플러그인 SKILL.md와 같은 디렉토리}/references/glossary.md`
- 출력 경로: 같은 디렉토리에 파일명 + `-쉬운버전.md` suffix

## 작업 절차
1. Read 도구로 원본 리포트 전체를 읽는다.
2. Read 도구로 용어집(`references/glossary.md`)을 읽어 치환 사전으로 사용한다.
3. 아래 "작업 원칙"에 따라 입문자용으로 리라이팅한다.
4. Write 도구로 `-쉬운버전.md`에 저장한다.
5. 한 줄로 `쉬운버전 저장 완료: <절대경로>`만 반환. **본문을 다시 복사해 반환 금지.**

## 작업 원칙 (엄수)

### 반드시 지킬 것
1. **모든 숫자·티커·날짜는 원본 그대로 유지**. 종가, 변동률, 금리, 환율, 섹터 변동률 등 어떤 수치도 변경·추정·생략 금지. 원본이 진실.
2. **구조(섹션 제목과 순서)는 원본과 동일**. TL;DR → 3대 지수 → 섹터 → 주요 종목 → 테마별 트리거(있으면) → 원자재/금리/환율 → 글로벌 → 국제정세 → 한국장 관전 포인트 → 전일 대비 변화 → 면책 → 데이터 품질 & 소스. 독자가 두 버전을 대조해 읽을 수 있어야 함.
3. **인과 해석 결론은 보존**, 어휘만 쉬운 말로. 원본의 "왜"는 같아야 함.
4. **URL과 출처 블록(`[[n]](url)` 포함)은 그대로 복사**.
5. **YAML frontmatter는 그대로 복사**. 내용 변경 금지.

### 전문용어 처리 룰
한 섹션에서 **처음 등장할 때 괄호 안에 짧은 해설**을 붙이고, 같은 섹션 안에서는 해설 없이 원어만 사용. 해설 텍스트는 `references/glossary.md`에서 참조. 목록에 없는 전문용어가 나오면 같은 방식으로 한 번만 괄호 해설을 생성.

### 문장 스타일
- "~임", "~됨" 같은 딱딱한 종결은 "~입니다"/"~했습니다"로
- 영어 약자가 처음 나올 때는 한글 풀네임 병기
- 인과는 "A 때문에 B가 이렇게 움직였습니다" 같은 단순 구조로

### 표 처리
시세 표는 그대로 복사하되, 각 표 바로 아래에 1~2줄 "쉬운 해석" 추가. 예시:
> Dow Jones(DIA)는 미국 전통 대기업 30곳을 묶은 지수입니다. 이 날 -0.55%로 3대 지수 중 가장 약했습니다.

### TL;DR은 특히 쉽게
원본 TL;DR의 **한줄평(첫 줄 굵은 인용구)**과 세부 bullet 둘 다 유지. 한줄평은 원본 의미는 보존하되 전문용어를 풀어 써서 주식 입문자도 한 번에 이해할 수 있게 리라이팅 (예: "CPI 쇼크" → "미국 물가가 예상보다 높게 나오고"). bullet은 원본 기반으로 더 풀어써도 OK (최대 10줄).

### 필수: 상단에 미니 용어집 (TL;DR 바로 위)
파일 최상단(YAML frontmatter 바로 아래, TL;DR 직전)에 `## 📘 이 리포트를 처음 보시나요? 미니 용어집` 섹션을 넣을 것. 본문에서 가장 중요한 5~10개 용어를 "용어명 — 한 줄 풀이" 형태로 정리. 말미 배치 금지.

## 금지 사항
- 숫자·티커 만들어내기 금지
- 원본에 없는 해석·의견·종목 추천 추가 금지
- 섹션 삭제 또는 순서 변경 금지
- URL 임의 변경 금지
- 투자 조언 성격 문장 추가 금지 (원본의 면책 문구는 그대로 유지)
- YAML frontmatter 변경 금지

## 반환
파일을 쓴 뒤 **한 줄로만** `쉬운버전 저장 완료: <절대경로>` 형태로 반환. 본문 재출력 절대 금지.
```

**Fault tolerance**: readability-pass 실패해도 원본은 저장 완료 상태. "쉬운버전 생성 실패" 한 줄만 고지.

## Sanity check 게이트

Step 6(보고서 저장) **직전**에 자동 검증. 아래 조건 중 **하나라도 위반**이면 보고서 상단에 `> ⚠️ **데이터 품질 경고**: <위반 내역>. 숫자를 맹신하지 말고 데이터 품질 섹션 참조.` 배너를 추가한 뒤 저장 진행.

### 검증 규칙
1. **지수 4개 (SPY/QQQ/DIA/IWM) 모두 `N/A`** → 시세 수집 전면 실패 가능성. 배너 표시 + 데이터 품질 섹션에 원인 명시.
2. **개별 티커 변동률 `|dp| > 10%`** → 일반적인 일간 변동이 아님. 스플릿·배당락·데이터 오류 가능성. 배너에 해당 티커 명시.
3. **환율 2건 모두 실패** → alphavantage 한도 또는 키 문제. 배너 표시.
4. **news-harvester가 빈 응답 또는 오류 반환** → 배너 대신 데이터 품질 섹션에 "뉴스 수집 실패, WebSearch fallback 사용" 명시 (배너는 시세 이상에 한정).

**위반 없음** → 배너 생략, 통상 저장.

## 분석: 인과 연결

데이터를 모았다면 **"왜"**를 써야 합니다.

- ❌ "XLE -0.68%"
- ✅ "XLE -0.68% — USO(WTI) -1.69%와 연동. 이란 휴전 기대 뉴스가 유가 차익실현을 유발 [[3]](https://...)"

### 출처 링크 규칙 (v0.3.0~)
**모든 인과 주장**(해석/로테이션 분석/트리거 셀)에는 news-harvester의 "주요 출처" 번호를 `[[n]](url)` inline 링크로 부착. 링크가 없는 인과는 **추론**으로만 표기 ("~로 보임"). TL;DR 한줄평은 면제, 본문 인과는 강제.

### 섹터 로테이션 자동 라벨
11개 섹터 변동률을 수집한 뒤 아래 룰로 라벨을 1개 선택해 "섹터 동향" 섹션 최상단에 표기:

| 라벨 | 조건 (변동률 기준) |
|---|---|
| **risk-on** | XLK/XLY/XLC 상위 3 + XLU/XLP 하위 3 |
| **risk-off** | XLU/XLP 상위 3 + XLK/XLY 하위 3 |
| **리플레이션** | XLF/XLE/XLB 상위 3 + XLU 하위 |
| **스태그플레이션 우려** | XLE/XLP 상위 + XLK/XLY 하위 |
| **혼조** | 위 4개에 해당 안 됨 |

### 체크리스트
- 섹터 로테이션을 설명할 매크로 이벤트가 있는가?
- 개별 종목 급등락 뒤에 실적/뉴스/애널리스트 코멘트가 있는가?
- 금리·달러·원자재 변동이 어떤 섹터에 반영됐는가?
- 지정학 이슈가 에너지·방산·반도체에 영향을 줬는가?
- 한국 시장과 밸류체인이 얽힌 섹터(반도체, 2차전지, 조선)는 어떻게 해석해야 하는가?

**불확실성 표기**: 명확하지 않으면 "~로 보임", "~가 원인 가능성"으로 약하게 표현. 거짓 확신 금지.

## 보고서 템플릿

아래 구조 그대로. 빈 섹션은 생략 대신 "특이사항 없음"으로 명시.

```markdown
---
date_kst: {YYYY-MM-DD}
session_date: {전일 NY 정규장 마감 YYYY-MM-DD}
generated_at: {ISO8601 KST}
tl_dr: "{한줄평}"
indices:
  SPY: {dp}
  QQQ: {dp}
  DIA: {dp}
  IWM: {dp}
rotation_label: {risk-on|risk-off|리플레이션|스태그플레이션 우려|혼조}
---

# {YYYY-MM-DD} 야간 미국 증시 & 국제정세 브리핑

> 생성 시각(KST): {HH:MM}
> 기준 세션: {전일 NY 정규장 마감 날짜}

{Sanity check 위반 시 여기에 ⚠️ 배너 삽입}

## 핵심 요약 (TL;DR)

> **{한줄평 — 오늘 장을 한 문장으로 요약, 60~90자 권장, 반드시 굵게}**
>
> "왜 그런 흐름이었는지"의 인과를 한 문장으로 압축. 트리거(매크로 이벤트·지정학·실적) + 결과(섹터 방향·지수 디버전스) 두 축을 모두 담을 것.

- (3~5줄 bullet, 한줄평을 뒷받침하는 세부 근거)

## 미국 3대 지수
| 지수 (ETF 프록시) | 종가 | 변동 | 변동률 |
|---|---|---|---|
| S&P 500 (SPY) |  |  |  |
| Nasdaq 100 (QQQ) |  |  |  |
| Dow Jones (DIA) |  |  |  |
| Russell 2000 (IWM) |  |  |  |
| VIX (VIXY 프록시) |  |  |  |

**해석**: (지수 움직임의 맥락과 원인 [[n]](url))

## 섹터 동향 — **{rotation_label}**
| 섹터 | ETF | 변동률 |
|---|---|---|
| 강세 1~3위 |  |  |
| 약세 1~3위 |  |  |

**로테이션 분석**: (왜 이런 방향성이 나왔는지 [[n]](url))

## 주요 종목
| 티커 | 종가 | 변동률 | 트리거 |
|---|---|---|---|
|  |  |  | [[n]](url) |

## 테마별 트리거 종목 (Tier 2/3 활성화 시에만)
- **테마 A** ({트리거 키워드}): {티커} {변동률} — {해석 [[n]](url)}

## 원자재 · 금리 · 환율
- **WTI 유가 (USO)**:
- **금 (GLD)**:
- **달러인덱스 (UUP)**:
- **美 10년물 금리**:
- **美 2년물 (SHY 프록시)** *(FOMC/CPI 당일만)*:
- **하이일드 (HYG)**:
- **장기국채 (TLT)**:
- **USD/KRW**:
- **USD/JPY**:

## 글로벌 증시
- **유럽 마감**: FTSE / DAX / CAC (또는 EWU / EWG 프록시)
- **아시아 전일**: Nikkei / HSI / KOSPI (또는 EWJ / EWH / EWY 프록시)

## 국제정세 & 매크로 이벤트
- (연준 / 경제지표 / 지정학 / 원자재 이벤트를 항목별로, 각각 [[n]](url))

## 오늘 한국장 관전 포인트
- (위 데이터를 바탕으로 한국 투자자가 주목할 포인트 3~5개)

## 전일 대비 변화
> 직전 보고서 파일이 있을 때만 작성. 없으면 "이전 보고서 없음 (첫 실행)"으로 표시.
- TL;DR 변화: {전일 한줄평} → {오늘 한줄평}
- SPY: {전일 dp} → {오늘 dp}
- QQQ: {전일 dp} → {오늘 dp}
- VIX: {전일} → {오늘}
- 로테이션 라벨: {전일} → {오늘}

## ⚠️ 면책 조항
본 보고서는 **정보 제공 목적**이며 특정 종목의 매수·매도 권유가 아닙니다. 모든 투자 결정은 본인 판단과 책임 하에 이뤄져야 합니다. 시세 수치는 MCP 데이터 소스(finnhub, alphavantage) 호출 시점 기준이며, 실제 체결가와 차이가 있을 수 있습니다.

## 데이터 품질 & 소스
> 이 섹션은 **보고서 말미**에 배치합니다. 독자가 원하는 것은 시장 브리핑이지 수집 메타정보가 아닙니다.

### 수집 요약
- 수집 성공률: {X}/{전체 섹션 수} 섹션 완전 확보
- 주력 소스: finnhub `get_quote` (시세) / alphavantage forex (환율) / Tavily·WebSearch (뉴스)
- 활성화된 Tier: Tier 1 always-on{+ Tier 2: [테마들]}{+ Tier 3: [이벤트들]}
- 실패/부분 확보: {섹션 이름 + 사유}

### 데이터 충돌 해소 내역
- (있을 때만. 예: subagent 인용 WTI +3.78% vs MCP USO -1.69% → 원칙대로 MCP 채택)

### 사용한 도구
- ✅/❌ 각 MCP·도구별 성공/실패 표시

### 주요 링크 (출처 번호 매핑)
- [[1]]: {URL}
- [[2]]: {URL}
- ...
```

## 주의사항

- **시세는 MCP가 진실**: 뉴스 기사에서 추출한 종가와 finnhub 실시간 데이터가 다르면 MCP 값 채택.
- **시간대 기준**: 한국 시간 기준 "어제 밤"은 NY 정규장 종료(KST 익일 새벽 5시경). 보고서 제목은 **한국 날짜**, YAML `session_date`는 **전일 NY 마감 날짜**.
- **실패 허용**: 특정 도구 실패 시 전체 중단 금지. `yfinance` 같은 알려진 실패 도구는 호출 자체를 생략.
- **숫자의 진실성**: 모든 숫자는 이번 실행의 실제 도구 호출 값. 기억/추정 금지. 구하지 못하면 `N/A`.
- **대용량 응답**: `finnhub news_sentiment` 등은 subagent 경유로만 호출.
- **뉴스 인용**: WebSearch/Tavily 결과는 번호 매겨 "주요 링크"에 기록. 본문은 `[[n]](url)` inline으로.
- **덮어쓰기 금지**: 같은 날짜 파일이 있으면 `YYYY-MM-DD-HHMM.md`로 새 파일.
- **보고 길이**: 사용자 채팅창에는 저장 경로 + TL;DR만. 본문 전체를 다시 붙여넣지 마세요.
- **연/월 명시**: WebSearch 쿼리에 반드시 현재 연/월 포함.
- **Tier 하드캡**: Tier 1 + 2 + 3 합쳐 finnhub 호출 55개 이하.
