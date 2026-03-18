# dashboard_render_flow — 대시보드 생성

**트리거**: `대시보드 갱신` / `현황판` / `대시보드 만들어줘`

모든 워크플로우 완료 후 자동 실행. Agent 호출 없이 직접 처리.

## 흐름

```
직접 처리:
  [CAREER_DIR]/05_Applications/*.md    → 지원 건 목록 (회사/포지션/상태/마감일 파싱)
  outputs/jd_reports/*.md              → JD 리포트 목록
  outputs/resumes/*.md                 → 이력서 버전 목록
  outputs/cover_letters/*.md           → 자소서 + 평가 목록 (eval_ 접두사 = 평가)
  outputs/positioning/*.md             → 포지셔닝 문서 목록
  [CAREER_DIR]/04_Upskill/gap_analysis.md       → 스킬 갭 TOP 5 추출
  [CAREER_DIR]/04_Upskill/portfolio_build_plan.md → 업스킬 항목 + 상태
  [CAREER_DIR]/04_Upskill/execution_log.md      → 실행 이력
  [PROJECTS_DIR]/Blog Posting/         → 블로그 후보 스캔
  [CAREER_DIR]/00_Dashboard/weekly_focus.md     → 주간 포커스 임베딩

출력: [CAREER_DIR]/00_Dashboard/dashboard.md 덮어쓰기 (자동 생성 문서)
```

## 대시보드 섹션 구조

```markdown
# Career Operating Dashboard

## 1. 진행 중인 지원 건
| 회사 | 포지션 | 상태 | 마감일 | 산출물 |

## 2. 부족 스킬 TOP 5
| 순위 | 스킬 영역 | 심각도 | 보완 계획 링크 |

## 3. 이력서 / 자소서 버전
| 유형 | 파일 | 생성일 | 비고 |

## 4. JD 분석 리포트
| 리포트 | 날짜 | 포함 내용 |

## 5. 블로그 후보
| 주제 | 출처 | 상태 |

## 6. 업스킬링 진행
| 항목 | 우선순위 | 상태 | 다음 단계 |

## 7. 다음 액션 추천 (최대 5개)
```

## 액션 추천 우선순위 로직

1. **마감 긴급** — D-7 이내 `[긴급]`, D-3 이내 `[긴급!!!]`
2. **미완성 워크플로우** — JD 있으나 포지셔닝 없음, 포지셔닝 있으나 자소서 없음, 자소서 있으나 평가 없음
3. **산출물 노후** — outputs/ 파일 생성일 14일+ 경과 + 활성 지원 존재
4. **갭→액션 매핑** — gap_analysis TOP 항목 vs execution_log 진행 현황
5. **블로그 주기** — 최근 7일 게시물 없으면 스킬갭 연계 주제 제안
6. **주간 포커스 정합** — weekly_focus 목표 vs 실제 산출물 비교
