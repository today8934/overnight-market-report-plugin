# 티커 유니버스 (3-Tier)

한국 투자자의 실전 해석에 필요한 커버리지를 3개 tier로 나눕니다. 고정 40~50개를 매일 조회하는 대신, 뉴스 키워드·달력 이벤트에 따라 동적으로 확장합니다.

## 🥇 Tier 1 — Always-on (매일 고정, 33개)

메인 orchestrator의 1차 병렬 배치에서 **무조건 수집**하는 티커. 한국장 해석의 최소 공통분모입니다. v0.4.0에서 **도메인 개별주(SMH/SOXX/MU/JPM/BAC/XOM/LLY/UNH/FXI/KWEB/BABA/HYG/TLT)를 Tier 2로 강등**하여 무이슈 날의 호출량을 축소했습니다.

### 지수 ETF 프록시 (5개)
| 지수 | ETF | 비고 |
|---|---|---|
| S&P 500 | **SPY** | 미국 대형주 벤치마크 |
| Nasdaq 100 | **QQQ** | 빅테크 비중 높음 |
| Dow Jones | **DIA** | 전통 산업 30대 |
| Russell 2000 | **IWM** | 미국 중소형주, 경기 민감도 ↑ |
| VIX 프록시 | **VIXY** | VIX 대체 (finnhub `^VIX`는 CFD 제한) |

### 섹터 ETF (11개 전체)
XLK(Tech), XLF(Financials), XLE(Energy), XLV(Healthcare), XLY(Consumer Discretionary), XLP(Staples), XLI(Industrials), XLU(Utilities), XLRE(Real Estate), XLB(Materials), XLC(Communication)

→ 수집 후 `dp`(변동률)로 정렬 → 상위 3 / 하위 3 식별.

### Mag7 + AI 파급 (7 + 2 = 9개)
**Mag7**: AAPL, MSFT, NVDA, GOOGL, AMZN, META, TSLA
**AI 파급**: AMD, AVGO

이들은 한국 투자자에게 거의 매일 해석 소재가 되므로 Tier 1에 고정. NVDA/AVGO는 반도체 시그널도 겸함.

### 원자재·달러 (3개)
| 자산 | ETF |
|---|---|
| WTI 유가 | **USO** |
| 금 | **GLD** |
| 달러인덱스 | **UUP** |

### 글로벌 ETF 프록시 (5개)
| 시장 | ETF |
|---|---|
| 일본 Nikkei | **EWJ** |
| 영국 FTSE | **EWU** |
| 독일 DAX | **EWG** |
| 홍콩 HSI | **EWH** |
| 한국 KOSPI | **EWY** |

매일 "글로벌 증시" 섹션에 필요하므로 Tier 1 유지.

### 환율 (2개, alphavantage)
- USD/KRW, USD/JPY

**Tier 1 합계**: 시세 ETF/주식 **31개** + 환율 2건 = **33 호출**. 한 메시지에서 finnhub 31 + alphavantage 2 병렬 호출하면 1차 수집 완료 (finnhub 무료 60 calls/min 한도 내).

---

## 🎯 Tier 2 — 테마 트리거 (news-harvester 결과 기반)

news-harvester subagent가 반환한 "카테고리별 임팩트" 내 키워드를 감지하면 **2차 병렬 배치**로 아래 티커를 추가 수집합니다. 감지가 없으면 호출하지 않음.

| 트리거 키워드 (뉴스 카테고리 내) | 추가 수집 티커 | 해석 목적 |
|---|---|---|
| "middle east", "iran", "israel", "hormuz", "red sea" | **ITA, LMT, RTX** + USO 심화 | 방산 섹터 수혜, 유가 프리미엄 반영 |
| "ukraine", "russia", "nato", "belarus" | **URA, ITA, LMT** | 우라늄·방산 — 한국 원전 수혜 |
| "ai data center", "power demand", "grid", "nuclear" | **NLR, URA, VST, CEG** | AI 전력 테마 — 두산에너빌리티/한전기술 해석 |
| "tariff", "china trade", "trump tariff", "export control" | **FXI, KWEB, BABA, CPER, MCHI** | 중국·구리 — KOSPI 수출주 파급 |
| "bitcoin", "crypto", "ethereum", "spot etf" | **IBIT, COIN, MSTR** | 크립토 센티먼트 — 한국 소매 투자자 심리 |
| "ev", "battery", "lithium", "solid state", "byd" | **LIT, ALB, RIVN** + TSLA 심화 | 2차전지 — LG엔솔/삼성SDI/에코프로 |
| "shipping", "bulk carrier", "tanker", "red sea shipping" | **BDRY, STNG, DAC** | 조선·해운 — HD현대중공업/팬오션 |
| "semiconductor earnings", "wafer", "lithography", "asml", "hynix", "samsung", "HBM", "korea chip", "memory" | **SMH, SOXX, MU, AMAT, LRCX, KLAC, ASML** | 반도체·반도체 장비 — 삼성·하이닉스 capex, 한국 메모리 밸류체인 |
| "gold rally", "silver", "precious metals" | **SLV, GDX** | 귀금속 대체자산 (GLD 외 채굴주) |
| "biotech", "fda approval", "clinical trial" | **IBB, XBI** | 바이오 ETF |
| "bank earnings", "jpmorgan", "wells fargo", "goldman sachs", "NIM" | **JPM, BAC** | 금융 메가캡 — 실적·금리 바로미터 |
| "oil earnings", "exxon", "chevron", "petroleum companies" | **XOM** | 에너지 개별주 — XLE 내부 리더십 |
| "pharma earnings", "weight loss drug", "healthcare policy", "eli lilly", "unitedhealth", "obesity drug" | **LLY, UNH** | 헬스케어 지배 종목 |
| "credit spread", "yield curve", "safe haven", "flight to quality", "bond rally" | **HYG, TLT** | 리스크 심리·장기 채권 |

### 활성화 절차

1. news-harvester subagent의 반환에서 `## 카테고리별 임팩트` 블록을 파싱
2. 각 카테고리 bullet 텍스트를 소문자화해 위 키워드 테이블과 substring 매칭
3. 매칭된 테마의 티커 union set을 **2차 finnhub `get_quote` 병렬 배치**로 수집
4. 최대 **15개**까지만 (과수집 방지). 초과 시 뉴스에서 가장 자주 언급된 2개 테마 우선
5. 결과는 보고서 "테마별 트리거 종목" 신규 섹션에 정리

---

## 📅 Tier 3 — 이벤트 트리거 (달력 기반)

실행 날짜가 특정 이벤트와 겹치면 추가 수집.

| 이벤트 | 감지 방법 | 추가 수집 |
|---|---|---|
| **FOMC 당일/익일** | news-harvester 결과에 "FOMC", "rate decision" 포함 | `SHY` (2년물 프록시) + `TLT` 심화 + WebSearch `"CME FedWatch probability {Month} {YYYY}"` |
| **CPI / PCE 발표일** | news-harvester 결과에 "CPI release", "PCE release" 포함 | `TIP` (물가연동채 ETF) + `DXY` 보조 |
| **어닝 시즌 피크** (1·4·7·10월 둘째~셋째 주) | 날짜로 직접 판단 | 해당 주 보고 예정 메가캡 (news-harvester에게 "today earnings" 쿼리 위임) |
| **중국 지표 발표일** | news-harvester 결과에 "China PMI", "China CPI", "China GDP" 포함 | `FXI` 심화 + `CPER` (중복 제거) |
| **OPEC+ 회의** | news-harvester 결과에 "OPEC", "oil production cut" 포함 | `USO` 심화 + `XOM` 심화 + `XLE` 재해석 |

---

## ⚖️ 운영 원칙

- **Tier 1은 항상**, Tier 2/3은 **news-harvester 응답 수령 후** 조건부 추가
- **하드 캡**: 1 + 2 + 3 합쳐 **finnhub 호출 55개 이하** (분당 60 한도 여유). Tier 1 축소(43→31)로 Tier 2/3 예산 여유 확대
- **중복 제거**: Tier 2/3에서 이미 Tier 1에 있는 티커는 스킵 (예: TSLA는 Tier 1이므로 "ev" 트리거 시 LIT/ALB/RIVN만 추가)
- **실패 허용**: Tier 2/3 호출 실패는 보고서 완성도에 치명적이지 않음 — 실패 내역은 "데이터 품질"에 한 줄만 명시하고 계속 진행
- **TL;DR 과부하 방지**: Tier 2/3로 수집한 티커는 "테마별 트리거 종목" 별도 섹션에만 두고 TL;DR 한줄평에는 가장 강한 신호 1~2개만 반영

### 강등 티커 백업 보호

Tier 1에서 Tier 2로 강등된 티커(SMH/SOXX/MU/JPM/BAC/XOM/LLY/UNH/FXI/KWEB/BABA/HYG/TLT)도 실제로는 트리거 키워드가 **상당히 자주** 발생합니다. 매주 최소 2~3일은 활성화될 것으로 예상. 만약 한 주 내내 특정 티커가 한 번도 활성화 안 되면 해당 테마 자체가 조용했다는 뜻 — 굳이 수집할 필요 없음이 확인된 셈.

**safety valve**: news-harvester의 카테고리 파싱이 실패해 Tier 2 매칭이 전혀 안 되는 drill-down 날, 메인 orchestrator는 **"Top 10 헤드라인" 블록도 substring 매칭**해 보조 트리거로 사용. 이중 안전망.
