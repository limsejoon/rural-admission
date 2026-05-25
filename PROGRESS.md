# PROGRESS — 작업 정리

마지막 갱신: 설계 합의 + 서울대 1교 완성 + 검수 콘솔 + 웹 렌더링까지.
이 문서는 VS Code/CLI로 이어갈 때 "어디까지 했고 무엇이 정해졌는지"를 담는다.

---

## 1. 확정된 설계 결정

| # | 항목 | 결정 |
|---|---|---|
| 1 | 대학 범위 | 인서울 약 25개교 + 지방 거점/국립대 약 10개교 (≈35) |
| 2 | 기준 연도 | 2028학년도 (내신 5등급제·통합형 수능 전환 첫 해 — 과거 데이터 비교 불가, 근거 저장이 더 중요) |
| 3 | 추출 방식 | 관리자가 로컬/Actions에서 변환 → 결과 JSON만 GitHub Pages에 올림 (정적 호스팅) |
| 4 | 파서 | native PDF는 PyMuPDF4LLM, 표 깨지면 MinerU / Marker(--use_llm) 2단 |
| 5 | granularity | 웹=전형 총원, 학과별은 L2(extracted)에 담아 AI가 응답. 없으면 "확인되지 않음" |
| 6 | 포함형 인원 | `rural_quota_status`로 2분기: `specified`(농어촌 별도 명시 → 그 숫자) / `merged`(합산되어 미상 → 전체만, 농어촌 null) / `pending` |
| 7 | 핵심 철학 | **추정 금지** — PDF에 없는 숫자는 만들지 않는다 |

## 2. 5개 레이어

```
L1 raw_data/<year>/<id>.pdf   원본 PDF              (AI Pack)
L2 extracted/<id>.json        단과대학·학과별 분해   (AI Pack 전용)
L3 structures/<id>.json       잘 안 바뀌는 구조       (웹)
L4 yearly/<year>/<id>.json    매년 바뀌는 숫자        (웹)
L5 evidence/<id>.json         근거(페이지·산출·검수)  (웹/AI)
```
`track_id`가 L3·L4·L5를 잇는 키. 내년엔 `yearly/2029/`만 추가, structure 재사용.

## 3. 완료된 산출물

- **데이터 스키마** — `schema/{structure,yearly,evidence}.schema.json`
- **서울대 1교 완성** — 5개 레이어 모두 실제 PDF에서 추출·교차검증
  - 수시 기회균형(사회통합) **183명** · `merged` (5개 대상 합산, 농어촌 미상) · 면접 O · 수능최저 X
  - 정시 '나'군 기회균형(농어촌·저소득) · 농어촌 **87명** · `specified` · 수능최저 O(3개 영역 3등급)
  - 단과대학 합산으로 87·183 교차검증 통과
- **검수 콘솔** — `tools/review-console.html`
  - 좌: PDF 대조 / 우: 전형별 필드 편집 / 상단 `AI 추출`(스키마·규칙대로 자동 추출, 해마다 재사용)
  - 단과대학 합 vs 농어촌 인원 실시간 검산, 4개 레이어로 분리 내보내기
  - AI 추출은 Claude 아티팩트 환경에서 동작 / 로컬 단독 사용 시 MinerU 결과를 JSON 불러오기로 이어감
- **웹 렌더링** — `site/index.html`
  - 랜딩(그냥 보기 / AI와 함께 보기) → 카드 → 상세 → AI Pack 안내
  - 배지가 의미를 구분: `농어촌 별도 명시`(초록) vs `전체·농어촌 미상`(황토)
  - 가/나/다군 숨김, 출처(evidence) 노출

## 4. 남은 일 (TODO)

- [ ] 웹을 인라인 DATA 대신 `data/` 레이어에서 **fetch**하도록 연결 (`universities.json` → 각 레이어 로드)
- [ ] 나머지 34개 대학 PDF 추출 (콘솔로 1교씩)
- [ ] **AI Pack 빌더** — ZIP 구성(JSON 레이어 + 원문표 + 질문 가이드) + 프롬프트 가이드(추정 금지 최우선)
- [ ] GitHub Pages 배포 (Actions 또는 수동)
- [ ] PDF 원문/AI Pack 다운로드 버튼을 실제 파일로 연결

## 5. 알려진 환경 제약

- GitHub Pages = 정적. 서버 로직 없음 → 추출은 사전 변환.
- 콘솔의 `AI 추출` fetch(api.anthropic.com)는 Claude 아티팩트 런타임에서만 키가 주입됨. 로컬 배포본에서는 비활성(편집·검산·내보내기는 정상).
- 웹의 `fetch('data/...')`는 `file://`에서 CORS로 막힐 수 있음 → 로컬은 `python3 -m http.server`로 띄워 확인.
