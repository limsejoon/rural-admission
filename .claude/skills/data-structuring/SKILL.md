---
name: data-structuring
description: Rural Admission Wiki의 5-Layer JSON 구조화 규칙. 농어촌 입시 데이터를 L3(structure)/L4(yearly)/L5(evidence) 형식으로 올바르게 만드는 방법. data-intake 에이전트가 JSON을 생성할 때, 또는 기존 파일을 수정할 때 반드시 이 스킬을 사용할 것.
---

## 핵심 원칙: 추정 금지

PDF/공시 자료에 없는 숫자를 만들지 않는다. 불확실한 값은 `null`이나 `"pending"`으로 표현하고, 이유를 `_note` 필드에 기록한다.

## L3 — Structure (data/structures/{id}.json)

전형 구조를 저장한다. 전형명, 수시/정시 구분, 면접 여부, 수능최저 여부 등 연도가 바뀌어도 잘 안 변하는 정보다.

```json
{
  "university_id": "yonsei",
  "university": "연세대학교",
  "region": "서울",
  "category": "in_seoul",
  "source_doc": "2028학년도 입학전형 시행계획 (2026.4.)",
  "tracks": [
    {
      "track_id": "yonsei_susi_rural",
      "track_name": "기회균형전형(농어촌학생)",
      "phase": "susi",
      "admission_basis": "학생부종합",
      "rural_type": "standalone",
      "rural_eligible_groups": ["농어촌"],
      "has_interview": true,
      "csat_minimum": false,
      "description": "농어촌 자격자만 단독 선발..."
    }
  ]
}
```

**rural_type 선택 기준:**
- `"standalone"` — 농어촌만 별도 선발하는 전형 (전형명에 "농어촌" 포함이 일반적)
- `"included"` — 기회균형/사회통합 등 다수 대상 중 농어촌이 포함된 전형

## L4 — Yearly (data/yearly/2028/{id}.json)

모집인원을 저장한다. 매년 바뀌므로 연도 폴더로 분리된다.

```json
{
  "university_id": "yonsei",
  "year": 2028,
  "tracks": [
    {
      "track_id": "yonsei_susi_rural",
      "track_total_quota": 45,
      "rural_quota": 45,
      "rural_quota_status": "specified",
      "rural_quota_note": "농어촌 단독 전형이므로 전체가 농어촌 인원",
      "csat_group": null,
      "evidence_id": "ev_yonsei_susi_rural"
    }
  ]
}
```

**rural_quota_status 결정 규칙:**
| 상황 | status | rural_quota |
|------|--------|-------------|
| PDF에 농어촌 인원 별도 명시 | `"specified"` | 명시된 숫자 |
| 여러 대상 합산, 농어촌 단독 미상 | `"merged"` | `null` |
| 아직 추출 안 함 | `"pending"` | `null` |

`"merged"`일 때 `track_total_quota`는 전체 합산 인원, `rural_quota`는 반드시 `null`.

## L5 — Evidence (data/evidence/{id}.json)

각 숫자의 근거를 저장한다. 객체의 키가 evidence_id다.

```json
{
  "ev_yonsei_susi_rural": {
    "track_id": "yonsei_susi_rural",
    "year": 2028,
    "value": 45,
    "source_pdf": "data/raw_data/2028/yonsei.pdf",
    "page": 7,
    "method": "모집인원 표 '농어촌학생' 열 합계에서 직접 추출",
    "note": "학과별 합산 검증 완료",
    "confidence": "unverified"
  }
}
```

**confidence 값:**
- `"verified"` — 사람이 검수 콘솔에서 PDF와 대조 확인
- `"unverified"` — AI 추출, 미검수
- `"pending"` — 아직 추출 전

## 레이어 간 연결 규칙

- `track_id`: L3 tracks[].track_id = L4 tracks[].track_id (동일 값)
- `evidence_id`: L4 tracks[].evidence_id = L5 최상위 키 (동일 값)
- L5 `value` = L4 `rural_quota` (일치 필수)

## 상세 참조

복잡한 전형 구조(포함형 처리 패턴, 단과대학 합산 방법 등)는 `references/schema-guide.md` 참조.
