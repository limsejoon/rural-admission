# Rural Admission Wiki — 데이터 레포

농어촌 자격자가 지원 가능한 대학 지원 경로를 구조화하는 데이터 시스템.
대상: 인서울 약 25개교 + 지방 거점/국립대 약 10개교. 기준 연도: 2028학년도.

> 이 프로젝트는 모집인원 숫자를 모으는 게 아니라, **그 숫자가 왜 나왔는지(근거)와 대학별 전형 구조**를 저장하는 프로젝트다.

---

## 데이터 흐름

```
PDF (모집요강/시행계획)
  → MinerU 등으로 추출
  → 도메인 규칙 적용해 JSON 구조화 + 근거 기록
  → 검수 콘솔로 사람이 보정
  → 확정 JSON을 git push → GitHub Pages 배포
  → 웹(총원만) + AI Pack(원본 분해·근거까지) 동시 생성
```

GitHub Pages는 정적 호스팅이라 변환은 **로컬/Actions에서 미리 돌려** 결과 JSON만 올린다.

---

## 5개 레이어

| 레이어 | 폴더 | 내용 | 변경 빈도 | 노출 |
|---|---|---|---|---|
| L1 Raw | `data/raw_data/<year>/` | 원본 PDF | 매년 | AI Pack |
| L2 Extracted | `data/extracted/` | PDF에서 뽑은 원본 분해표(단과대학·학과별) | 매년 | AI Pack 전용 |
| L3 Structure | `data/structures/` | 잘 안 바뀌는 구조(전형명·면접·수능최저·농어촌 유형) | 드묾 | 웹 |
| L4 Yearly | `data/yearly/<year>/` | 매년 바뀌는 숫자(모집인원·군) | 매년 | 웹 |
| L5 Evidence | `data/evidence/` | 숫자의 근거(PDF 페이지·산출 방식·검수 상태) | 매년 | 웹/AI |

`track_id`가 L3·L4·L5를 잇는 키. 내년엔 `yearly/2029/`만 새로 만들고 structure는 재사용.

---

## 핵심 규칙

**1. 추정 금지.** PDF에 없는 숫자는 만들지 않는다. `rural_quota_status`로 표현:
- `specified` — 농어촌 인원이 별도 명시됨 (그 숫자를 그대로 사용)
- `merged` — 다른 대상과 합산되어 농어촌 단독 인원 미상 (전체 숫자만 저장, 농어촌은 "확인되지 않음")
- `pending` — 아직 추출 안 함

**2. 배지는 의미를 바꾼다.**
- `[농어촌 단독]` / `specified` → "이 숫자가 농어촌 몫"
- `[농어촌 포함]` / `merged` → "전체 숫자이고, 농어촌이 그중 몇 명인지는 모름"

**3. 군(가/나/다)은 저장하되 웹에선 숨김.** 사용자가 질문할 때만 제공.

**4. 학과 단위는 L2까지만.** 웹은 전형 총원, 학과 질문은 AI가 L2를 뒤져서 답하되 없으면 "확인되지 않음".

---

## 서울대 워크드 예시 (2028)

서울대에는 PRD가 가정한 "농어촌 183명" 같은 깔끔한 단일 숫자가 **없다.** 실제 구조:

| 경로 | 전형 | 전체 인원 | 농어촌 인원 | 상태 |
|---|---|---|---|---|
| 수시 | 기회균형(사회통합) | 183 (+농생명 4 정원외) | 미상 | `merged` |
| 정시 '나'군 | 기회균형(농어촌·저소득) | 180 | **87** | `specified` |

- **수시 사회통합 183명**: 농어촌·저소득·보훈·서해5도·자립 5개 대상 합산. 면접 있음, 수능최저 미적용. 농어촌 단독 인원은 표에 없음.
- **정시 농어촌 87명**: 저소득(93)과 별도 책정되어 농어촌 숫자가 명시됨. 수능 60%+교과역량 40%, 수능최저(4개 중 3개 영역 각 3등급 이내) 적용. 단과대학 단위까지만 명시(공과대학 17명 등), 학과별은 정원 10% 상한 규칙만 있어 미명시.

모든 숫자는 단과대학별 합산으로 교차검증(87, 183 일치).

---

## 폴더 구조

```
rural-admission-wiki/
├── README.md
├── universities.json            # 전체 대학 인덱스
├── schema/                      # 각 레이어 JSON Schema
│   ├── structure.schema.json
│   ├── yearly.schema.json
│   └── evidence.schema.json
└── data/
    ├── raw_data/2028/snu.pdf     # L1
    ├── extracted/snu.json        # L2
    ├── structures/snu.json       # L3
    ├── yearly/2028/snu.json      # L4
    └── evidence/snu.json         # L5
```

## 다음 단계
- [ ] 검수 콘솔 (추출값 편집 + 근거 PDF 페이지 대조)
- [ ] 웹 렌더링 (카드/상세, 군 숨김)
- [ ] AI Pack 빌드 + 프롬프트 가이드 (추정 금지 규칙 최우선)
- [ ] 나머지 대학 PDF 추출

---

## CLI 시작하기 (VS Code)

```bash
# 1) 레포 초기화 & 첫 커밋
cd rural-admission-wiki
git init
git add .
git commit -m "init: 5-layer schema + SNU + review console + web"

# 2) 웹 로컬 확인 (file:// 는 fetch가 막히므로 서버로)
python3 -m http.server 8000
#  → http://localhost:8000/site/         웹 렌더링
#  → http://localhost:8000/tools/review-console.html   검수 콘솔

# 3) 새 대학 추가 워크플로우
#  - PDF를 data/raw_data/2028/<id>.pdf 에 저장
#  - 콘솔에서 PDF 불러오기 → AI 추출 → 보정 → 검수완료 → 4개 레이어 내려받기
#  - 받은 파일명의 __ 를 폴더 구분으로 바꿔 배치:
#      structures__<id>.json   → structures/<id>.json
#      yearly__2028__<id>.json  → yearly/2028/<id>.json
#      evidence__<id>.json      → evidence/<id>.json
#      extracted__<id>.json     → extracted/<id>.json
#  - universities.json 에 대학 1줄 추가
```

### 폴더
```
rural-admission-wiki/
├── PRD.md            제품 요구사항 (v1.1)
├── PROGRESS.md       작업 로그 · 결정사항 · TODO
├── README.md         아키텍처 + 이 가이드
├── universities.json 대학 인덱스
├── schema/           JSON Schema 3종
├── data/             L1~L5 레이어
├── tools/            검수 콘솔
└── site/             웹 렌더링
```
