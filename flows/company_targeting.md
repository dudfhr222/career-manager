# company_targeting_flow — 회사 지원 준비

**트리거**: `[회사명] 지원 준비해줘` / `이 공고로 지원서 써줘`

**필요 정보**: 공고 URL 또는 JD 텍스트, 자소서 항목 + 글자 수 제한

## 흐름

```
전제조건 가드 체크
  ↓
┌─────────────────── 병렬 실행 ───────────────────┐
│ data_foundation[analyze_jds]                     │
│ (JD 미분석 시)                                    │
│                                                   │
│ strategy_positioning[company_full]                │
│ (회사 분석 + 포지셔닝 + 갭 코칭)                   │
└───────────────────────────────────────────────────┘
  ↓ (둘 다 완료 후)
document_writer[full_application]
  → cover_letter_evaluator (자동 호출)
  ↓
결과 요약 출력
  ↓ (완료 후)
dashboard_render_flow 자동 실행
```

## 산출물

| 유형 | 경로 |
|------|------|
| JD 파일 | `[CAREER_DIR]/05_Applications/[회사명]_[포지션명].md` |
| JD 분석 리포트 | `outputs/jd_reports/jd_report_YYYY-MM-DD.md` |
| 회사 프로파일 | `outputs/company_profiles/[회사명]_company_profile_YYYY-MM-DD.md` |
| 포지셔닝 시트 | `outputs/positioning/[회사명]_포지셔닝_YYYY-MM-DD.md` |
| 맞춤 경력기술서 | `outputs/resumes/resume_[회사명]_YYYYMMDD.md` |
| 자소서 | `outputs/cover_letters/[회사명]_자소서_YYYYMMDD.md` |
| 자소서 평가 | `outputs/cover_letters/eval/eval_[회사명]_YYYYMMDD.md` |

## 부분 실행

| 요청 | 실행 범위 |
|------|----------|
| `JD 분석만 해줘` | data_foundation[analyze_jds] |
| `회사 분석만 해줘` | strategy_positioning[company_full] |
| `경력기술서만 써줘` | document_writer[custom_resume] |
| `자소서만 써줘` | document_writer[cover_letter] |

## 전제조건 가드

- `[CAREER_DIR]/07_Assets/asset_*.md` 1개 이상 → 없으면 data_foundation[curate] 자동 선행
- `[CAREER_DIR]/templates/Career_template.md` 비어있지 않음 → 비어있으면 중단 후 안내
