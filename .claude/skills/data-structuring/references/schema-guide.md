# 스키마 상세 가이드

## 목차
1. [포함형(included) 전형 처리 패턴](#포함형-전형-처리-패턴)
2. [단독형(standalone) 전형 처리 패턴](#단독형-전형-처리-패턴)
3. [단과대학 합산 교차검증](#단과대학-합산-교차검증)
4. [track_id 명명 규칙](#track_id-명명-규칙)
5. [흔한 실수 패턴](#흔한-실수-패턴)

---

## 포함형(included) 전형 처리 패턴

기회균형/사회통합 등 다수 대상이 묶인 전형에서 농어촌이 일부인 경우.

**서울대 수시 사례:**
- 기회균형특별전형(사회통합): 5개 대상(농어촌·저소득·보훈·서해5도·자립) 합산 183명
- PDF에 농어촌 단독 인원 미명시
- 처리: `rural_type: "included"`, `rural_quota_status: "merged"`, `rural_quota: null`, `track_total_quota: 183`

**서울대 정시 사례:**
- 기회균형특별전형(농어촌·저소득): 농어촌 87명, 저소득 93명 별도 명시
- 처리: `rural_type: "included"`, `rural_quota_status: "specified"`, `rural_quota: 87`, `track_total_quota: 180`

핵심: 같은 "포함형"이라도 PDF에 농어촌 별도 숫자가 있으면 `specified`, 합산만 있으면 `merged`.

---

## 단독형(standalone) 전형 처리 패턴

농어촌 자격자만 선발하는 별도 전형. 전형명에 "농어촌"이 명시되는 경우가 많다.

- `rural_type: "standalone"`
- `rural_eligible_groups: ["농어촌"]`
- 전체 인원 = 농어촌 인원 → `rural_quota_status: "specified"`, `rural_quota = track_total_quota`

---

## 단과대학 합산 교차검증

대학들이 단과대학별로 표를 제시하고 합계 행을 별도로 표기한다. 검증 방법:
1. 단과대학별 인원을 합산
2. 표의 합계 행과 일치하는지 확인
3. 불일치 시 evidence의 `note` 필드에 기록

서울대 사례:
- 단과대학 17개 값 합산 = 183 ✓
- 단과대학 17개 값 합산 = 87 ✓

교차검증 완료 시 evidence `method` 필드에 "단과대학별 N개 값 합산 검증 완료(=X)" 형식으로 기록.

---

## track_id 명명 규칙

```
{university_id}_{phase}_{유형}
```

- phase: `susi` 또는 `jeongsi`
- 유형: `rural` (단독), `social` (사회통합), `opportunity` (기회균형), `equal` (고른기회) 등 전형 특성을 짧게 표현
- 같은 대학에 동일 phase + 유형의 전형이 2개 이상이면 숫자 추가: `_rural1`, `_rural2`

예시:
- `snu_susi_social` — 서울대 수시 사회통합
- `snu_jeongsi_rural` — 서울대 정시 농어촌
- `yonsei_susi_rural` — 연세대 수시 농어촌(단독)

---

## 흔한 실수 패턴

**실수 1: merged인데 rural_quota에 숫자 기입**
```json
// 잘못
"rural_quota_status": "merged",
"rural_quota": 45  // ← 이 숫자는 어디서 왔는가?
```
`merged`는 농어촌 단독 미상을 의미한다. 전체 인원의 일부라고 추정해서 기입하는 것은 "추정 금지" 원칙 위반.

**실수 2: evidence_id와 L5 키 불일치**
```json
// L4
"evidence_id": "ev_yonsei_susi_rural"

// L5 (잘못)
{
  "ev_yonsei_rural_susi": { ... }  // 순서 다름
}
```
`evidence_id`는 `ev_{track_id}` 형식을 정확히 따른다.

**실수 3: L5 value가 L4 rural_quota와 다름**
L5는 L4 숫자의 근거다. `value`는 `rural_quota`와 반드시 일치해야 한다.
`rural_quota: null`이면 L5 `value: null`.

**실수 4: confidence를 verified로 기입**
AI가 추출한 데이터는 반드시 `"unverified"`. 검수 콘솔에서 사람이 PDF와 대조 확인한 경우에만 `"verified"`.
