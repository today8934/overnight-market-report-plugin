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
- `references/news-harvester.md` — news-harvester subagent 호출 프롬프트 전문 (Step 3에서 Read)
- `references/readability-pass.md` — readability-pass subagent 호출 프롬프트 전문 (Step 8에서 Read)
- `references/report-template.md` — 보고서 템플릿 전체 + mode별 diff 섹션 분기 + 섹션 순서 체크리스트 (Step 6에서 Read)
- `references/glossary.md` — readability-pass 전용 용어 사전

## 출력 위치

기본 경로: `~/workspace/overnight-market-report/YYYY-MM-DD.md` (오늘 한국 날짜 기준).

### 경로 결정 절차
1. `~/.claude/data/overnight-market-report/config.json`이 있고 `output_dir` 필드가 존재하면 그 값을 우선 사용 (예: `~/Documents/obsidian/overnight`)
2. 없으면 기본 경로 사용
3. 디렉토리가 없으면 먼저 생성 (`mkdir -p`)
4. 같은 날짜 파일이 이미 존재하면 덮어쓰지 말고 `YYYY-MM-DD-HHMM.md` 형태로 시간 suffix

### config.json 예시
```json
{
  "output_dir": "~/Documents/obsidian/overnight-market-report"
}
```

사용자가 "출력 경로 바꿔줘" / "저장 위치를 X로" 같이 요청하면 이 파일을 Write해 설정.

## 실행 순서

0. **Preflight (캐시 우선)** — `~/.claude/data/overnight-market-report/preflight.json`을 읽어 `last_ok_at`이 24h 이내 + 모든 checks가 `"ok"`면 **skip**하고 Step 1로. 캐시 miss / TTL 초과 / 지난 실행에서 401 감지 시 `references/setup-wizard.md`를 Read해 **inline preflight** 실행 (subagent 아님, 메인이 직접 ToolSearch + 3개 cheap test 병렬). 실패 항목이 있으면 Setup Wizard 진입 후 halt (메인 워크플로우 진입 금지).
1. **오늘 한국 날짜·시각(KST) 확인 + 기준 세션 결정** (아래 "기준 세션 결정 로직" 참조) → 파일 경로 결정
2. **출력 디렉토리 확인/생성** + 직전 파일 Read → **mode 결정** (아래 "Mode 결정" 참조, YAML frontmatter `mode`와 diff 섹션 분기에 사용)
3. **한 메시지에서 병렬 실행** (하이브리드 오케스트레이션):
   - Tier 1 시세 MCP 호출 × 31 (finnhub `get_quote`) — `references/theme-tickers.md`의 Tier 1 리스트 (v0.4.0부터 축소: 도메인 개별주는 Tier 2 트리거로 이전)
   - 환율 MCP × 2 (alphavantage USD/KRW, USD/JPY)
   - 보조 WebSearch 1~3 (VIX/10Y 등 지수 수치 보완)
   - **`Agent` tool로 news-harvester subagent 1회** (뉴스 수집 위임 — 프롬프트는 `references/news-harvester.md`를 Read해 사용)
4. **Tier 2/3 조건부 2차 배치** — news-harvester 응답 수령 후 카테고리 키워드를 `references/theme-tickers.md` Tier 2/3 매핑과 substring 매칭해 매칭된 테마의 티커를 **2차 finnhub 병렬 호출** (최대 15개 cap, Tier 1 중복은 제외).
5. **Sanity check 게이트** (아래 "Sanity check" 참조) — 통과 시 Step 5.5, 실패 시 리포트 상단에 경고 배너를 추가한 뒤 계속.
5.5. **출처 번호 normalize + YAML sources 기록** — 본문 작성 완료 후 저장 **직전** 필수 수행:
   - 본문 전체에서 `[[n]](url)` 패턴을 **등장 순서대로** 스캔
   - 등장 순서대로 `1..N`으로 **재할당** (같은 URL이 여러 번 등장하면 같은 번호로 통일)
   - "주요 링크" 블록을 재할당된 번호 기준으로 **재작성**, 본문에서 사용되지 않은 URL은 제외
   - 결과: gap 없이 `[[1]]..[[N]]` 연속 번호만 존재해야 함 ([[6]], [[8]], [[12]] 누락 같은 상태 금지)
   - **YAML frontmatter에 `sources` 배열 동시 기록**: `[{id: 1, url: "...", title: "..."}, ...]`. 주간 롤업·대시보드 파싱용. title은 뉴스 기사 제목(news-harvester가 제공한 1줄 요약 앞부분) 또는 도메인명.
6. **보고서 작성·저장** — `references/report-template.md`의 구조 그대로. 섹션 순서(요약): YAML frontmatter → TL;DR → 3대 지수 → 섹터 → 주요 종목 → 테마별 트리거 종목(Tier 2/3이 활성화된 경우만) → 원자재/금리/환율 → 글로벌 → 국제정세 → 한국장 관전 포인트 → {mode별 diff 섹션} → 전체 티커 시세(접이식) → 면책 → 데이터 품질 & 소스(말미). 인과 주장에는 inline `[[n]](url)` 출처 링크 필수.
7. **Preflight 캐시 갱신** — 전체 실행이 문제없이 완료되면 `preflight.json`의 `last_ok_at`을 현재 KST ISO8601로 갱신.
8. **`Agent` tool로 readability-pass subagent 1회** (순차 실행) — 완성된 리포트를 입력받아 `-쉬운버전.md` 생성. 프롬프트는 `references/readability-pass.md`를 Read해 사용.
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

## Mode 결정

동일 디렉토리의 직전 보고서 파일 (가장 최근 `YYYY-MM-DD*.md`)을 Read한 뒤 그 파일의 YAML `session_date`를 기준으로 분기:

| 상황 | `mode` | Diff 섹션 |
|---|---|---|
| 직전 파일 없음 | `initial` | 생략 |
| 직전 파일의 `session_date` == 이번 실행의 `session_date` | `intraday_refresh` | "## Intraday 업데이트 내역" |
| 직전 파일의 `session_date` != 이번 실행의 `session_date` | `next_day` | "## 전일 대비 변화" |

**주의**: `intraday_refresh`일 때 지수 4개는 동일 세션 종가라 diff가 0. 표기 생략하고 **신규 캡처 티커 / 뉴스·breadth 업데이트 / TL;DR 변화**만 기록. "지수 변동 없음" 같은 공백 bullet 금지.

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

Step 3에서 **`references/news-harvester.md`를 Read**해 그 안의 프롬프트를 `Agent` tool (`subagent_type: general-purpose`)로 전달. 프롬프트 본문에 `{YYYY-MM-DD}`, `{Month}`, `{D}`, `{YYYY}`, `{전일 NY 정규장 마감 날짜}` 플레이스홀더를 실제 값으로 치환.

### 🛡 Fault tolerance
- news-harvester 실패 → 메인이 WebSearch 3~5개 직접 실행. 시세 리포트는 정상 완성 (fault isolation)
- Tier 2/3 활성화 실패 → 무시하고 Tier 1만으로 작성 (데이터 품질에 한 줄 명시)
- finnhub 일부 티커 실패 → 해당 티커만 `N/A`로 표시, 섹션 전체는 유지

### 🎨 Readability-pass subagent (순차 후처리)

원본 리포트 저장 **뒤** 별도 subagent 1회 실행 — `-쉬운버전.md` 생성. Step 8에서 **`references/readability-pass.md`를 Read**해 그 안의 프롬프트를 `Agent` tool (`subagent_type: general-purpose`)로 전달. `{원본 절대경로}`, `{references 디렉토리 절대경로}` 플레이스홀더만 치환.

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

전체 템플릿(YAML frontmatter · mode별 diff 분기 · 섹션 순서 체크리스트 포함)은 **`references/report-template.md`에 별도 관리**. Step 6에서 필요 시 Read.

**핵심 제약 (여기서 재확인)**:
- 빈 섹션은 생략 대신 "특이사항 없음"으로 명시
- TL;DR bullet은 정확히 3~5개, 한줄평 중복 금지
- "주요 종목" 표는 `|dp| ≥ 1.5%` 또는 트리거 뉴스 있는 티커만, 최대 20행, 나머지는 `<details>` collapse
- mode별 diff 섹션은 SKILL.md "Mode 결정" 규칙에 따라 정확히 하나만 생성
- 인과 주장은 `[[n]](url)` inline 출처 필수, 최종 저장 직전 Step 5.5에서 번호 normalize

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
