---
name: overnight-weekly-digest
description: overnight-market-report 플러그인이 쌓아둔 일간 리포트들을 주간 또는 월간 롤업 보고서로 합성합니다. vault 내 YAML frontmatter(date_kst, session_date, mode, tl_dr, indices, rotation_label, sources)를 파싱해 지수·로테이션·반복 테마·핵심 이벤트 타임라인을 요약합니다. 사용자가 "주간 미장 정리", "지난 주 미국 증시 요약", "monthly digest", "overnight-weekly-digest", "이번 달 야간 보고서 롤업" 같은 표현을 쓸 때 트리거하세요. 단일 일간 리포트를 원하는 경우에는 overnight-market-report 스킬을 사용하고, 이 스킬은 **기간 집계**에만 사용합니다.
---

# Overnight Weekly Digest

overnight-market-report 플러그인의 일간 산출물을 기간 단위로 롤업. 매일의 TL;DR·지수·섹터 로테이션·언급 테마를 합쳐 주간/월간 추세를 드러냅니다.

## 왜 필요한가

매일 리포트가 쌓이는데 지난 5~20일의 흐름을 역추적하려면 파일을 하나씩 열어야 합니다. YAML frontmatter가 구조화되어 있으니 **파싱 기반 자동 요약**이 자연스럽습니다. 단순 복붙이 아니라 지수 변화폭 누적, 로테이션 라벨 분포, 반복 등장 테마(Tier 2/3 트리거), 핵심 출처 빈도를 뽑습니다.

## 출력 위치

일간 리포트와 같은 디렉토리(기본 `~/workspace/overnight-market-report/`, 또는 `~/.claude/data/overnight-market-report/config.json`의 `output_dir`).

- 주간: `digest-YYYY-WW.md` (ISO 주차)
- 월간: `digest-YYYY-MM.md`

같은 이름 파일이 있으면 `-HHMM.md` 시간 suffix.

## 실행 순서

1. **기간 결정** — 사용자 자연어 파싱:
   - "지난 주" / "주간" → 지난 월요일~일요일 (KST 기준 ISO week)
   - "이번 주" → 이번 주 월요일~오늘
   - "지난 달" / "월간" → 이전 달 1일~말일
   - 명시적 날짜 범위 ("4/15~4/22") → 그대로
2. **리포트 파일 스캔** — 출력 디렉토리에서 `YYYY-MM-DD*.md` 패턴 + 파일명에 `digest-` 접두사는 제외. `session_date` 기준으로 기간 필터.
3. **중복 제거** — 같은 `session_date`에 여러 파일(initial + intraday_refresh)이 있으면 **가장 최신 generated_at** 1건만 채택.
4. **YAML 파싱** — 각 파일 상단 YAML frontmatter만 Read(offset/limit로 40~60줄). 본문은 필요할 때만 추가 Read.
5. **집계 계산** (아래 "집계 지표" 참조).
6. **요약 작성** (아래 "Digest 템플릿" 참조).
7. **저장** — 경로 규칙대로 저장. 본문은 사용자에게 붙여넣지 말고 저장 경로 + 핵심 bullet 3~5개만 보고.

## 집계 지표

- **지수 누적 변화**: `indices.SPY/QQQ/DIA/IWM`의 기간 내 일간 `dp` 합 및 최고/최저일
- **로테이션 라벨 분포**: `rotation_label` count (risk-on/off/리플레이션/스태그/혼조)
- **반복 테마**: "테마별 트리거 종목" 섹션을 스캔해 등장한 Tier 2/3 테마 이름 count — 빈도 상위 5
- **핵심 뉴스 출처**: `sources` 배열의 URL을 도메인 단위로 count — 상위 10 (reuters.com, cnbc.com 등)
- **모드 분포**: initial / intraday_refresh / next_day count (intraday_refresh 비율이 높으면 사용자가 장중 갱신에 의존)
- **핵심 한줄평 타임라인**: 각 날짜별 `tl_dr`를 연대기순 bullet로 나열 (최대 10개, 기간이 길면 3일 간격 샘플링)
- **이벤트 타임라인**: tl_dr + 본문 "국제정세 & 매크로 이벤트" 헤드라인에서 FOMC/CPI/PCE/실적/지정학 키워드 추출

## Digest 템플릿

```markdown
---
digest_type: {weekly|monthly|custom}
period_start: {YYYY-MM-DD}
period_end: {YYYY-MM-DD}
generated_at: {ISO8601 KST}
reports_count: {N}
rotation_distribution:
  risk-on: {n}
  risk-off: {n}
  리플레이션: {n}
  스태그플레이션 우려: {n}
  혼조: {n}
---

# 야간 미국 증시 {period_start} ~ {period_end} 롤업

> 기간 내 {N}개 리포트 집계 (중복 제거 후).

## 핵심 흐름
- (3~5줄, 기간 전체의 서사를 1문단으로 압축)

## 지수 누적
| 지수 | 누적 변동률 합 | 최고일 | 최저일 |
|---|---|---|---|
| SPY |  |  |  |
| QQQ |  |  |  |
| DIA |  |  |  |
| IWM |  |  |  |

## 섹터 로테이션 분포
| 라벨 | 일수 |
|---|---|
| risk-on |  |
| risk-off |  |
| ... |  |

**해석**: (어떤 라벨이 지배했고 왜)

## 반복 테마 Top 5
1. {테마명} — {N회 등장}, 대표 날짜 {MM-DD, MM-DD}
2. ...

## 한줄평 타임라인
- {MM-DD}: {tl_dr}
- ...

## 핵심 출처 도메인 Top 10
- reuters.com ({N}회)
- cnbc.com ({N}회)
- ...

## 다음 기간 관전 포인트
- (마지막 리포트의 "오늘 한국장 관전 포인트" 기반, 아직 해소 안 된 테일리스크)
```

## 주의사항

- **본문 전수 Read 금지**: YAML frontmatter만 파싱하는 게 원칙. 본문은 "반복 테마" / "이벤트 타임라인" 뽑을 때만 선택적 Read (grep으로 헤드라인 몇 줄).
- **숫자 진실성**: 누적 변동률은 리포트에 기록된 `dp` 값을 그대로 합산. 재계산/보정 금지.
- **쉬운버전 파일 제외**: 스캔 시 `-쉬운버전.md` suffix 파일은 스킵 (원본과 동일 데이터).
- **기간 미명시**: 사용자 요청에 기간이 불명확하면 "지난 주 (월~일) 기준으로 뽑을까요, 아니면 기간 지정하시겠어요?" 한 번 질문.
- **출력 디렉토리 자동 생성**: 원본 플러그인 config와 동일한 규칙. `~/.claude/data/overnight-market-report/config.json`의 `output_dir`을 참조.
- **보고 길이**: 사용자 채팅창에는 저장 경로 + 핵심 흐름 3~5줄만. 본문 전체 재출력 금지.
