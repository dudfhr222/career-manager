# jd_to_gap_analysis_flow — JD 분석

**트리거**: `이 공고 분석해줘: [URL]` / `JD 분석해줘` + JD 텍스트 / `채용 트렌드 알려줘`

**필요 정보**: 공고 URL 또는 JD 텍스트

> JS 렌더링 사이트(원티드, 링크드인 등)는 URL 크롤링이 안 될 수 있음. JD 텍스트를 직접 붙여넣기.

## 흐름

```
JD 소스 확인 (파일 / 사용자 제공 / 웹검색)
  ↓
data_foundation[analyze_jds]
  ↓
리포트 요약 출력
  ↓ (고심각도 갭 발견 시, 선택)
strategy_positioning[gap_coaching]
  ↓ (완료 후)
dashboard_render_flow 자동 실행
```

## 산출물

| 유형 | 경로 |
|------|------|
| JD 개별 파일 | `[CAREER_DIR]/05_Applications/[회사명]_[포지션명].md` |
| JD 종합 분석 리포트 | `outputs/jd_reports/jd_report_YYYY-MM-DD.md` |
| 갭 액션 플랜 (선택) | `outputs/upskill_plans/*.md` |

## 리포트 포함 내용

- 공통 요구 역량 Top 10 (빈도 기준)
- 필수/우대 역량 분리
- 역량 갭 현황 (심각도 분류)
- 회사별 특이사항
- 시장 트렌드

## 부분 실행

| 요청 | 실행 범위 |
|------|----------|
| `JD만 저장해줘` | 파싱·저장만 (갭 분석 생략) |
| `갭 분석까지 해줘` | 분석 + 갭 도출 |
| `학습 계획도 짜줘` | gap_coaching까지 전체 |
| `채용 트렌드 알려줘` | 웹 검색 기반 트렌드 분석 |

## JD 필터링 기준

`inputs/user_config.md` 설정 기준으로 아래 공고는 자동 제외:
- 마감 공고
- 접속 불가 URL
- 게시일 6개월 초과
- 희망 지역 불일치 (user_config.md의 지역 설정 기준)
