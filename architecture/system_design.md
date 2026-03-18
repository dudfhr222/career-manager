# System Design — Career Manager

## 전체 아키텍처

```
┌─────────────────────────────────────────────────────────┐
│                   Claude Code Session                    │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │              CLAUD.md (Orchestrator)             │   │
│  │  - 세션 초기화 (파이프라인 상태 스캔)             │   │
│  │  - 요청 분류 (판단 트리)                          │   │
│  │  - 워크플로우 디스패치                            │   │
│  │  - 전제조건 가드                                  │   │
│  │  - 직접 처리 (상태 조회, 대시보드)                │   │
│  └─────────────┬───────────────────────────────────┘   │
│                │ Agent tool 호출                         │
│  ┌─────────────┼───────────────────────────────────┐   │
│  │             ↓           ↓           ↓            │   │
│  │  [data_foundation] [strategy_positioning] [...]   │   │
│  │  각 agent는 self-contained prompt으로 실행         │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
         │                              │
         ↓                              ↓
  [CAREER_DIR]/                    outputs/
  (원본 데이터, 수정 금지)          (파생 산출물)
```

---

## 5개 Agent 역할 분담

| Agent | 통합 역할 | 주요 책임 |
|-------|----------|----------|
| `data_foundation` | experience_curator + market_analyst | 경험 원본 → asset 변환, JD 수집·분석, 역량 갭 도출 |
| `strategy_positioning` | positioning_strategist + company_analyst + skill_gap_coach | 포지셔닝 설계, 회사 프로파일, 갭 액션 플랜 |
| `document_writer` | resume_writer + cover_letter_writer | 경력기술서·자소서 생성, 이력서-자소서 교차 검증 |
| `cover_letter_evaluator` | 평가 전문 | 탈락 패턴 분석, 8대 항목 등급 평가, 수정 지시서 |
| `interview_prep` | interview_simulator | 예상 Q&A, STAR 답변 설계, 압박 질문 시뮬레이션 |

---

## 데이터 흐름

```
[CAREER_DIR]/08_Experiences/*.md  ─────────────────────┐
[CAREER_DIR]/templates/Career_template.md               │
                                                        ↓
                                            data_foundation[curate]
                                                        │
                                                        ↓
                                            [CAREER_DIR]/07_Assets/asset_*.md
                                                        │
                          JD 데이터 ──┐                  │
                                      ↓                  ↓
                            data_foundation[analyze_jds]
                                      │
                                      ↓
                    outputs/jd_reports/jd_report_*.md
                    [CAREER_DIR]/05_Applications/*.md
                                      │
                                      ↓
                          strategy_positioning[company_full]
                                      │
                              ┌───────┴────────┐
                              ↓                ↓
                  outputs/company_profiles/  outputs/positioning/
                                              │
                                              ↓
                                   document_writer[full_application]
                                              │
                                    ┌─────────┴──────────┐
                                    ↓                     ↓
                            outputs/resumes/     outputs/cover_letters/
                                                          │
                                                          ↓ (자동)
                                             cover_letter_evaluator
                                                          │
                                                          ↓
                                          outputs/cover_letters/eval/
```

---

## 병렬/순차 오케스트레이션 패턴

### 병렬 실행 가능 조합

| 조합 | 조건 | 사용 워크플로우 |
|------|------|--------------|
| `data_foundation[analyze_jds]` + `strategy_positioning[company_full]` | JD와 회사 정보가 각각 독립적 | company_targeting_flow |
| 복수 회사 `data_foundation[analyze_jds]` | 회사별 JD 독립 존재 | 비교 분석 |
| 복수 회사 `strategy_positioning[company_full]` | data_foundation 완료 후 각 회사 독립 | 복수 회사 준비 |

### 순차 강제 조합

| 조합 | 이유 |
|------|------|
| `data_foundation` → `strategy_positioning` | positioning이 JD 분석 결과에 의존 |
| `strategy_positioning` → `document_writer` | 문서 작성이 포지셔닝 시트에 의존 |
| `document_writer[resume]` → `document_writer[cover_letter]` | 자소서가 이력서 내용 참조 |
| `document_writer` → `cover_letter_evaluator` | 평가 대상 자소서 필요 |

---

## 오케스트레이터 판단 트리

```
세션 시작
  └─ STEP 1: 컨텍스트 로드 (Career_template, 경험 현황, 주간 포커스)
  └─ STEP 2: 파이프라인 상태 스캔 (회사별 7단계 진행 테이블)
  └─ STEP 3: 마감일 긴급도 스캔 (D-day 계산 + 복합 경고)
  └─ STEP 4: 산출물 버전 중복 감지
  └─ STEP 5: 세션 시작 요약 출력 → dashboard_render_flow

사용자 요청 수신
  ├─ 분석 계열 → data_foundation / strategy_positioning
  ├─ 산출물 계열 → 해당 agent 체인
  ├─ 지원 계열 → company_targeting_flow
  ├─ 면접 계열 → interview_prep_flow
  ├─ 상태 확인 → 직접 처리
  └─ 미인식 → 유사 의도 추론 또는 가능 작업 목록 제시
```

---

## 전제조건 가드

각 agent 호출 전 Glob으로 선행 조건을 체크한다. 미충족 시 선행 agent 자동 실행.

| 호출 대상 | 전제조건 | 미충족 시 |
|---|---|---|
| `document_writer (resume)` | `[CAREER_DIR]/07_Assets/asset_*.md` 존재 | data_foundation[curate] 자동 실행 |
| `strategy_positioning (회사별)` | JD report 또는 JD 파일 존재 | data_foundation[analyze_jds] 자동 실행 |
| `interview_prep` | positioning sheet 존재 | strategy_positioning 자동 실행 |
| `document_writer (cover_letter)` | positioning sheet + 자소서 항목 | strategy_positioning + 항목 요청 |
| 모든 agent | Career_template.md 비어있음 | 작성 안내 후 중단 |

---

## 산출물 구조

```
outputs/
├── jd_reports/          # JD 종합 분석 리포트
├── company_profiles/    # 회사 프로파일 (인재상·근황·문화)
├── positioning/         # 포지셔닝 시트 (어필 축·맞춤 한 줄·경쟁 차별화)
├── resumes/             # 경력기술서 (.md 작업본 + .txt 클린본)
├── cover_letters/       # 자소서 (.md 작업본 + .txt 클린본)
│   └── eval/            # 자소서 평가 리포트
├── interview_packs/     # 면접 준비 팩
├── upskill_plans/       # 갭 기반 업스킬 액션 플랜
└── archive/             # 버전 정리 후 이전 파일 보관
```

---

## Agent 호출 인터페이스

오케스트레이터가 agent를 호출하는 방식:

```
1. agent prompt 파일을 Read
2. Read한 내용을 Agent tool의 prompt로 전달
3. 실행 지시(mode, 파라미터, 오늘 날짜, 작업 디렉토리)를 prompt 하단에 추가

예시:
  Agent tool:
    prompt: |
      [agents/data_foundation.md 전체 내용]

      ---
      ## 실행 지시
      mode: curate
      오늘 날짜: YYYY-MM-DD
      작업 디렉토리: [PROJECT_DIR]
```

---

## 설계 결정 기록

### 왜 Self-contained Agent Prompt인가?

Claude Code의 Agent tool은 각 subagent가 독립적 컨텍스트에서 실행된다. 외부 rules 파일을 참조하면 매 호출마다 추가 Read가 필요하고 컨텍스트 오염 위험이 있다. 모든 규칙을 agent 파일 자체에 임베딩하면 단일 파일로 완결되는 실행 단위가 된다.

### 왜 5개 Agent인가?

초기에는 8개 agent였으나 역할 경계가 모호한 agent들(experience_curator + market_analyst → data_foundation, positioning_strategist + company_analyst + skill_gap_coach → strategy_positioning 등)을 통합. 의존 관계가 명확한 5개로 정리하면 오케스트레이터의 디스패치 로직이 단순해지고 병렬화 판단이 쉬워진다.

### 왜 cover_letter_evaluator를 별도 agent로 분리했는가?

자소서 작성(document_writer)과 평가(cover_letter_evaluator)는 서로 다른 관점을 요구한다. 작성은 "어떻게 잘 쓸 것인가", 평가는 "왜 탈락할 수 있는가". 같은 agent에서 두 관점을 동시에 갖게 하면 평가가 관대해진다. 분리함으로써 평가 agent는 항상 비판적 관점을 유지한다.
