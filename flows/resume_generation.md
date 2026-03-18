# project_to_resume_flow — 이력서 생성

**트리거**: `경력기술서 작성해줘` / `이력서 써줘` / `이력서 최신화해줘`

**필요 정보**: `[CAREER_DIR]/08_Experiences/`에 경험 파일 1개 이상

## 흐름

```
전제조건 가드 체크
  ↓
data_foundation[curate]
  ↓
strategy_positioning[universal]
  ↓
document_writer[base_resume]
  ↓
결과 요약 출력
  ↓ (완료 후)
dashboard_render_flow 자동 실행
```

## 산출물

| 유형 | 경로 |
|------|------|
| 경험 asset | `[CAREER_DIR]/07_Assets/asset_[프로젝트명].md` |
| 전이 역량 | `[CAREER_DIR]/07_Assets/transferable_skills.md` |
| 성장 궤적 | `[CAREER_DIR]/07_Assets/growth_trajectory.md` |
| 범용 포지셔닝 | `[CAREER_DIR]/03_Narrative/one_line_intro.md` |
| 범용 경력기술서 (작업본) | `outputs/resumes/resume_base_YYYYMMDD.md` |
| 범용 경력기술서 (클린본) | `outputs/resumes/resume_base_YYYYMMDD.txt` |

## 갱신 트리거

- 원본 소스(`08_Experiences/`) 수정 시 → 전체 재실행
- 산출물 생성 후 2주 경과 + 활성 지원 진행 중 → 갱신 권장
- 원본 변경 없으면 → "원본 변경 없음, 그래도 갱신할까?" 확인 후 진행
