# Overnight Market Report Plugin

간밤 미국 증시와 국제정세를 자동으로 수집·분석해 **한국장 개장 전 브리핑용 한국어 마크다운 보고서**를 만드는 Claude Code 스킬 플러그인입니다. 전문가용 원본 리포트와 **주식 입문자용 쉬운버전**을 한 번에 생성합니다.

> "미국주식", "야간시황", "밤사이 증시", "미장 마감", "어제 미장 어땠어" 등으로 호출하면 자동 트리거됩니다.

---

## ✨ 주요 특징

- **3-Tier 티커 유니버스 (v0.4.0 슬림화)** — 매일 고정 **Tier 1(31개)** + 뉴스 키워드 기반 **Tier 2(테마 트리거)** + 달력 이벤트 기반 **Tier 3**. v0.3.0 Tier 1에 있던 도메인 개별주(SMH/SOXX/MU/JPM/BAC/XOM/LLY/UNH/FXI/KWEB/BABA/HYG/TLT)는 Tier 2 트리거 조건으로 이전. 무이슈 날 호출량 −28%, 이슈 많은 날은 기존과 유사 수준 유지
- **Mode별 diff 분기 (v0.4.0)** — `initial` / `intraday_refresh` / `next_day` 3개 모드를 YAML `mode` 필드로 구분. 동일 세션 인트라데이 리프레시일 땐 "Intraday 업데이트 내역"으로 신규 캡처 티커·뉴스 breadth만 기록, 다음 영업일일 땐 "전일 대비 변화"로 지수·로테이션 diff
- **주요 종목 표 필터 + 전체 티커 collapse (v0.4.0)** — 표에는 `|dp| ≥ 1.5%` 또는 트리거 뉴스 있는 티커만, 하드캡 20행. 나머지는 말미 `<details>전체 티커 시세</details>` 접이식 블록으로 격리해 가독성 보존
- **TL;DR 5줄 하드캡 + 중복 금지 (v0.4.0)** — 한줄평 인용문을 풀어 쓴 중복 bullet 금지. 각 bullet은 다른 각도에서 뒷받침하는 세부 근거만
- **출처 번호 normalize + YAML sources (v0.4.0)** — 저장 직전 본문의 모든 `[[n]](url)`을 등장 순서 기준 `1..N`으로 재할당, 미사용 URL 제거. `sources` 배열이 YAML frontmatter에 구조화 저장되어 주간/월간 롤업 파싱 가능
- **Inline preflight (v0.4.0)** — 이전 버전의 preflight-check 서브에이전트를 제거하고 메인 세션이 `ToolSearch` + 3개 MCP cheap test 병렬을 직접 수행. 캐시 miss 시 응답 지연이 절반 이하로 감소
- **Preflight 24h 캐싱** — 이미 세팅된 사용자는 매 실행 preflight 없이 바로 리포트 생성
- **하이브리드 오케스트레이션** — 메인 세션이 `finnhub get_quote`로 시세를 병렬 수집하고, 별도 `news-harvester` 서브에이전트가 `Tavily search/extract + WebSearch`로 뉴스·매크로 이벤트를 격리 수집. 메인 컨텍스트가 대용량 뉴스 원문으로 오염되지 않음
- **인과 분석 + 출처 링크 의무화** — 단순 숫자 나열이 아니라 "왜 그렇게 움직였는지"를 섹터 로테이션·원자재·매크로 이벤트와 연결해 설명. 본문의 모든 인과 주장에 inline `[[n]](url)` 출처 링크 부착
- **섹터 로테이션 자동 라벨** — risk-on / risk-off / 리플레이션 / 스태그플레이션 우려 / 혼조 중 하나를 섹터 섹션 최상단에 자동 표기
- **두 버전 자동 생성** —
  1. **원본 리포트** (전문가용, 디버전스·섹터 로테이션·FedWatch 같은 전문용어 사용)
  2. **쉬운버전** (입문자용, 상단 미니 용어집 + 본문에 괄호 해설 + 표 아래 "쉬운 해석" 삽입)
- **US 휴장/주말 자동 가드** — 실행 시점의 한국 요일과 US 공휴일을 반영해 기준 세션 날짜를 결정
- **Sanity check 게이트** — 지수 전면 N/A, 변동률 ±10% 초과, 환율 전면 실패 등 이상치 감지 시 리포트 상단 ⚠️ 배너
- **데이터 충돌 자동 해소** — 서브에이전트가 인용한 뉴스 수치와 MCP 실시간 값이 다르면 **MCP를 진실로 채택**하고 해소 내역을 리포트 말미에 기록
- **주간/월간 롤업 skill (v0.4.0 신규)** — `overnight-weekly-digest` skill이 누적된 일간 리포트의 YAML frontmatter를 파싱해 지수 누적·로테이션 분포·반복 테마·핵심 뉴스 출처 빈도를 합성. "지난 주 미장 정리" / "monthly digest" 등으로 트리거
- **저장 경로 configurable (v0.4.0)** — 기본 `~/workspace/overnight-market-report/`. `~/.claude/data/overnight-market-report/config.json`에 `output_dir` 필드로 사용자 지정 가능 (예: Obsidian vault 하위). 같은 날짜 파일이 이미 있으면 `YYYY-MM-DD-HHMM.md` 형태로 시간 suffix

---

## 🚀 설치 (README 읽을 필요 없음)

> **이 섹션은 플러그인이 어떻게 동작하는지 궁금할 때만 보세요.** 실제 설치는 아래 두 명령만 실행하면 끝납니다 — 키 발급·환경변수 설정 같은 사전작업이 **전혀 필요 없습니다**.

### ⚡ 한 번만 하면 되는 설치 + 첫 실행

**1. 플러그인 설치** (Claude Code 세션에서):
```
/plugin marketplace add today8934/wooksang-marketplace
/plugin install overnight-market-report-plugin@wooksang-marketplace
```

**2. 바로 호출**:
```
미국주식 야간 보고서 뽑아줘
```

**3. 자동 설정 마법사(Setup Wizard)가 뜨고** 다음과 같이 진행됩니다:
- 플러그인이 내부적으로 `preflight-check` 서브에이전트를 돌려 finnhub·alphavantage·tavily 3개 MCP 상태를 확인
- 미설정 상태면 Claude가 누락된 키를 **한 개씩 순차로 요청**합니다 (각 키마다 무료 발급 링크도 함께 안내)
- 사용자는 링크 타고 가서 무료 가입 → 키 복사 → 대화창에 붙여넣기
- 플러그인이 `claude mcp add` CLI로 자동 등록
- "Claude Code를 재시작해주세요" 안내 → 재시작 → 다시 `미국주식 야간 보고서` 호출 → 이번엔 바로 리포트 생성

**두 번째 호출부터는** 이미 키가 등록돼 있으므로 preflight이 바로 통과하고 리포트가 즉시 생성됩니다. 마법사는 다시 뜨지 않습니다.

### 🔑 사용되는 3개 API (모두 무료 플랜)
마법사가 직접 안내하지만 참고용:
| 서비스 | 용도 | 가입 | 무료 한도 |
|---|---|---|---|
| [Finnhub](https://finnhub.io/register) | 주식 시세 (주력) | 이메일만 | 분당 60 호출 |
| [Alpha Vantage](https://www.alphavantage.co/support/#api-key) | 환율 | 이메일만 | 하루 25 호출 |
| [Tavily](https://tavily.com) | 뉴스·매크로 리서치 | 이메일만 | 월 1,000 credits |

### 🧑‍💻 개발/테스트용 로컬 설치 (기여자 전용)

플러그인 자체를 수정하거나 테스트하려면:
```bash
git clone https://github.com/today8934/overnight-market-report-plugin.git
cd overnight-market-report-plugin
claude --plugin-dir .
```

일반 사용자는 위의 `/plugin marketplace add` 방식을 사용하세요.

---

## 🎯 사용법

### 일간 리포트 (`overnight-market-report`)

Claude Code 세션에서 아래와 같은 자연어 문구 중 아무거나 입력하면 자동 트리거됩니다:

- "미국주식 야간 보고서 뽑아줘"
- "밤사이 증시 어땠어?"
- "간밤 미국 시황 정리해줘"
- "어제 미장 어땠어"
- "밤사이 무슨 일 있었어"
- "미국 증시 브리핑"

Claude는 다음 단계를 순차/병렬로 수행합니다:

1. Inline preflight (메인 세션이 직접 3개 MCP cheap test 병렬 — 캐시 hit 시 skip)
2. 한국 시각 확인 + mode 결정 (initial / intraday_refresh / next_day) → 저장 경로 결정
3. `finnhub get_quote` ×31 (Tier 1) + `alphavantage forex` ×2 + `WebSearch` ×2~3 + `news-harvester` 서브에이전트 ×1 — **한 메시지에서 병렬 실행**
4. news-harvester 응답의 카테고리 키워드를 Tier 2/3 매핑과 매칭 → 조건부 2차 배치 (최대 15 티커)
5. Sanity check → 출처 번호 normalize + YAML sources 기록 → 원본 리포트 작성·저장
6. `readability-pass` 서브에이전트 실행 → 같은 디렉토리에 `-쉬운버전.md` 생성
7. 사용자에게 두 경로와 TL;DR만 짧게 보고

대략 2~3분 내에 두 파일이 완성됩니다.

### 주간/월간 롤업 (`overnight-weekly-digest`)

누적된 일간 리포트를 기간 단위로 합성. 트리거 문구:

- "지난 주 미장 정리"
- "주간 야간 보고서 롤업"
- "monthly digest"
- "이번 달 미국 증시 요약"
- "overnight-weekly-digest"

Claude는 YAML frontmatter(`date_kst`, `session_date`, `mode`, `tl_dr`, `indices`, `rotation_label`, `sources`)만 스캔해 지수 누적·로테이션 분포·반복 테마·핵심 뉴스 도메인 빈도를 합성한 뒤 `digest-YYYY-WW.md` 또는 `digest-YYYY-MM.md`로 저장합니다.

---

## 📄 산출물 예시 구조

### 원본 리포트 (`YYYY-MM-DD.md`)
0. **YAML frontmatter** (date_kst / session_date / generated_at / **mode** / tl_dr / indices / rotation_label / **sources**)
1. 핵심 요약 (TL;DR) — 한줄평 + 3~5줄 bullet 하드캡
2. 미국 3대 지수 (SPY/QQQ/DIA/IWM + VIX 프록시)
3. 섹터 동향 (XL* ETF 11종 일간 변동률 정렬 + **로테이션 라벨**)
4. 주요 종목 — **`|dp| ≥ 1.5%` 또는 트리거 뉴스 있는 티커만, 최대 20행**
5. **테마별 트리거 종목** (Tier 2/3 활성화 시에만: 반도체 장비·방산·우라늄·크립토·배터리·금융 메가캡·중국·헬스케어·리스크 심리 등)
6. 원자재 · 금리 · 환율 (WTI·금·달러인덱스 + 10Y·2Y + USD/KRW·JPY; HYG/TLT는 Tier 2 트리거 시)
7. 글로벌 증시 (ETF 프록시)
8. 국제정세 & 매크로 이벤트 (연준·인플레·지정학·실적 — 각 주장에 [[n]](url) 출처)
9. 오늘 한국장 관전 포인트
10. **Mode별 diff 섹션** — `initial`이면 생략, `intraday_refresh`면 "Intraday 업데이트 내역" (신규 캡처 티커·뉴스 breadth만), `next_day`면 "전일 대비 변화"
11. **📊 전체 티커 시세** (`<details>` 접이식, 주요 종목 표 미포함분 전수)
12. ⚠️ 면책 조항
13. 데이터 품질 & 소스 *(말미)*

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
│  ├─ finnhub get_quote ×31 (Tier 1: 지수/섹터/      │
│  │     Mag7+AI/원자재·달러/글로벌 ETF)               │
│  ├─ alphavantage get_forex_rate ×2                 │
│  ├─ WebSearch ×2~3 (VIX/10Y 보조)                  │
│  └─ Agent → news-harvester subagent ×1 ─┐          │
│     (뉴스 카테고리 매칭 후 Tier 2/3 ≤15 2차 배치)  │
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

### MCP 서버가 연결되지 않거나 401 에러가 날 때
**가장 쉬운 방법**: 아무 트리거("미국주식 야간 보고서")로 호출하면 Setup Wizard가 자동으로 재실행되어 키를 다시 받아 등록합니다.

**수동 확인·재등록**:
```bash
# 상태 확인
claude mcp list

# 특정 서버 재등록 (예: tavily)
claude mcp remove tavily
claude mcp add tavily -e TAVILY_API_KEY=<your-key> -- npx -y tavily-mcp@latest
```

### `tavily_search`, `tavily_extract`가 도구 목록에 없을 때
Tavily MCP가 로드는 됐지만 deferred tool로 등록된 경우입니다. Claude는 `ToolSearch("select:mcp__tavily__tavily_search,mcp__tavily__tavily_extract")`로 자동 로드합니다. 사용자 조치 불필요.

### finnhub 특정 티커에서 "CFD subscription required" 에러
해외 지수 심볼(`^N225`, `^FTSE` 등)은 무료 플랜 제한입니다. 스킬이 **자동으로 ETF 프록시(EWJ, EWU 등)로 우회**하므로 사용자 조치는 불필요.

### Alpha Vantage 25/day 제한 도달
이 플러그인은 환율 2건만 호출하므로 하루 10회 이상 실행해야 도달합니다. 제한에 걸리면 해당 리포트의 USD/JPY 등이 `N/A`로 표시되지만 전체 리포트는 정상 완성됩니다.

### Setup Wizard를 다시 보고 싶을 때 (키 교체 등)
`claude mcp remove finnhub` (또는 `alphavantage` / `tavily`) 로 등록을 먼저 지우고 플러그인을 재호출하면 preflight이 `needs_setup`을 반환하여 Wizard가 다시 실행됩니다.

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
