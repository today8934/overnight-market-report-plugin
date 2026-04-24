# News-harvester Subagent 프롬프트

메인 오케스트레이터가 Step 3에서 `Agent` tool (`subagent_type: general-purpose`)로 호출할 때 **그대로 복사**해 사용하는 프롬프트. 대용량 뉴스 수집을 메인 context에서 격리하는 용도.

## 호출 방법

`Agent` tool에 아래 프롬프트를 전달. `{YYYY-MM-DD}`, `{Month}`, `{D}`, `{YYYY}`, `{전일 NY 정규장 마감 날짜}`는 실제 값으로 치환.

## 프롬프트 (치환 후 그대로 전달)

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
- (핵심 URL 5~10개, 번호 붙여 — 메인이 [[n]](url) inline 링크로 사용. **최대 10개 하드캡**)

중요: 원문을 직접 복붙하지 말고 반드시 요약. 응답 전체가 메인 context에 포함되므로 간결함이 핵심. 불확실한 인과는 "~로 보임"처럼 약하게 표현.
```

## Fault tolerance

- subagent 실패/빈 응답 → 메인이 WebSearch 3~5개 직접 실행. 시세 리포트는 정상 완성 (fault isolation)
- Tavily MCP 미로드 → subagent가 자동으로 WebSearch만으로 폴백 (쿼리 다양성은 `data-sources.md` 참조)
