# weekly_operating_cycle — 주간 루틴

**트리거**: `이번 주 뭐 할까?` / `주간 리뷰` / `진행상황 알려줘`

Agent 호출 없이 직접 처리.

## 흐름

```
직접 스캔:
  [CAREER_DIR]/07_Assets/         → asset 파일 수/목록
  [CAREER_DIR]/03_Narrative/      → positioning 완성 여부
  [CAREER_DIR]/05_Applications/   → JD 수집 현황
  outputs/ 하위 디렉토리별        → 산출물 존재 여부 + 최종 수정일
  [CAREER_DIR]/00_Dashboard/weekly_focus.md → 이번 주 포커스

출력: 상태 테이블 + 추천 다음 액션
  ↓ (완료 후)
dashboard_render_flow 자동 실행
```

## 출력 형식

```
| 영역 | 상태 | 최종 갱신일 | 비고 |
|------|------|------------|------|
| 경험 asset | N개 완성 | YYYY-MM-DD | — |
| 포지셔닝 | 완성/미완성 | YYYY-MM-DD | — |
| JD 수집 | N개 | YYYY-MM-DD | 분석 필요 |
| 경력기술서 | 완성/미완성 | YYYY-MM-DD | — |
| 면접팩 | 없음/있음 | YYYY-MM-DD | [회사명] D-N 🔴 |

추천 다음 액션 (최대 5개):
1. [긴급/주의] ...
2. ...
```

## 권장 주간 루틴 패턴

1. 주초: `이번 주 뭐 할까?` → 상태 확인 + 우선순위 결정
2. 주중: 지원 준비 / JD 분석 / 경험 정리
3. 주말: `주간 리뷰` → 완료 항목 확인 + 다음 주 포커스 설정

## 산출물 신선도 기준

- 원본 소스 수정일 > 파생 산출물 생성일 → 갱신 권장
- 산출물 생성 후 2주 경과 + 활성 지원 진행 중 → 갱신 권장
- 갱신 우선순위: asset → positioning → resume → interview_pack
