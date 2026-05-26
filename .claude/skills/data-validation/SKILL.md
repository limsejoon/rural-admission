---
name: data-validation
description: Rural Admission Wiki JSON 데이터 검증 규칙. L3/L4/L5 파일의 스키마 준수, 도메인 규칙 위반, 레이어 간 불일치를 점검하는 방법. data-validator 에이전트가 검증할 때, 또는 기존 데이터 파일에 오류가 있는지 확인할 때 반드시 이 스킬을 사용할 것.
---

## 검증 순서

1. 파일 존재 확인 → 2. 스키마 검증 → 3. 도메인 규칙 검증 → 4. 레이어 간 교차검증

## 1. 파일 존재 확인

대학 ID를 받으면 다음 3개 파일을 읽는다:
```
data/structures/{id}.json
data/yearly/2028/{id}.json
data/evidence/{id}.json
```
없는 파일은 "파일 없음" 오류로 기록하고 해당 레이어 검증은 건너뛴다.

## 2. 스키마 검증

스키마 파일: `schema/structure.schema.json`, `schema/yearly.schema.json`, `schema/evidence.schema.json`

**확인 항목:**
- L3 required: `university_id`, `university`, `tracks`
- L3 tracks[] required: `track_id`, `track_name`, `phase`, `rural_type`, `has_interview`
- L4 required: `university_id`, `year`, `tracks`
- L4 tracks[] required: `track_id`, `track_total_quota`, `rural_quota`, `rural_quota_status`, `evidence_id`
- L5 각 항목 required: `track_id`, `year`, `value`, `source_pdf`, `confidence`
- enum 허용값: `phase` (susi/jeongsi), `rural_type` (standalone/included), `rural_quota_status` (specified/merged/pending), `confidence` (verified/unverified/pending), `csat_group` (ga/na/da/null)

## 3. 도메인 규칙 검증

| 규칙 | 확인 방법 |
|------|----------|
| specified → rural_quota 정수 | `status=="specified"` 이고 `rural_quota`가 null이거나 0 이하면 오류 |
| merged → rural_quota null | `status=="merged"` 이고 `rural_quota`가 숫자면 오류 (추정 금지 위반) |
| rural_quota ≤ track_total_quota | rural_quota 숫자일 때 초과하면 오류 |
| standalone → rural_eligible_groups에 "농어촌" 포함 | L3에서 확인 |
| 정시 전형 → csat_group 존재 | L3 phase=="jeongsi"인 track이 L4에서 csat_group이 없으면 경고 |

## 4. 레이어 간 교차검증

**L3 ↔ L4 매핑:**
- L3 tracks[].track_id 집합 = L4 tracks[].track_id 집합 (1:1 매핑)
- 한쪽에만 있는 track_id → 오류

**L4 ↔ L5 매핑:**
- L4 tracks[].evidence_id 집합 = L5 최상위 키 집합 (1:1 매핑)
- `ev_` 접두어 + track_id 형식 일치 여부

**L4 ↔ L5 수치 일치:**
- L4 `rural_quota` == L5 `value` (null 포함)

## 오류 보고 형식

```
[오류] data/yearly/2028/yonsei.json
  - tracks[0].rural_quota_status: "merged"인데 rural_quota: 45 (추정 금지 위반)
  - 수정 제안: rural_quota를 null로 변경

[경고] data/structures/yonsei.json
  - tracks[1]: 정시 전형이지만 csat_group 필드 없음
  - 수정 제안: csat_group 필드 추가 ("ga"/"na"/"da" 중 해당값)
```

오류(규칙 위반)와 경고(권고사항)를 구분해서 보고한다.
