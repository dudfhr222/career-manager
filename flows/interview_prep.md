# interview_prep_flow — 면접 준비

**트리거**: `[회사명] 면접 준비해줘` / `모의 면접 해줘` / `예상 질문 뽑아줘`

**필요 정보**: 회사명, (선택) 면접 단계(1차/2차/임원)·날짜

## 흐름

```
면접 정보 확인 (회사, 단계, 날짜)
  ↓
strategy_positioning[interview_positioning]
  ↓ (포지셔닝 시트 없으면 자동 생성)
interview_prep
  ↓
결과 요약 출력
  ↓ (완료 후)
dashboard_render_flow 자동 실행
```

## 산출물

| 유형 | 경로 |
|------|------|
| 면접용 포지셔닝 | `outputs/positioning/[회사명]_interview_positioning_YYYY-MM-DD.md` |
| 면접 준비 팩 | `outputs/interview_packs/[회사명]_interview_pack_YYYYMMDD.md` |

## 면접 준비 팩 구성

- 면접 유형 (WebSearch 기반, 미확인 태그)
- 경험 기반 질문 + STAR 답변 + 발화 시간
- 기술 심화 질문
- 포지셔닝 검증 질문
- 약점/압박 질문 + 대응 논리
- 역질문 3개
- 함정 질문 대응
- 암기 카드 (경험별 핵심 팩트 3개)

## 면접 단계별 질문 비중

| 면접 단계 | 경험 기반 | 기술 심화 | 포지셔닝 | 약점/압박 |
|-----------|----------|----------|----------|----------|
| 1차 (인성/컬처핏) | 40% | 10% | 30% | 20% |
| 2차 (기술 심화) | 30% | 50% | 10% | 10% |
| 임원 | 30% | 10% | 40% | 20% |

## 전제조건

- `[CAREER_DIR]/07_Assets/asset_*.md` 존재
- `outputs/positioning/[회사명]_*.md` 존재 (없으면 strategy_positioning 자동 선행)
