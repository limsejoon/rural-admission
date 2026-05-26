---
name: site-update
description: Rural Admission Wiki universities.json 업데이트 규칙. 새 대학 추가, 상태 변경, 사이트 동기화 방법. 데이터 파일 추가/수정 후 사이트에 반영하거나, universities.json을 수정할 때 이 스킬을 사용할 것.
---

## universities.json 구조

파일 위치: `universities.json` (프로젝트 루트)

```json
{
  "active_year": 2028,
  "available_years": [2028],
  "categories": { ... },
  "universities": [
    {
      "university_id": "snu",
      "university": "서울대학교",
      "category": "in_seoul",
      "region": "서울",
      "status": "complete",
      "source_doc": "2028 시행계획 (2026.4.)"
    }
  ]
}
```

## status 결정 기준

| 상태 | 조건 |
|------|------|
| `"complete"` | L3/L4/L5 모두 존재, 검증 통과 |
| `"partial"` | 일부 레이어만 완성 또는 confidence가 "unverified"인 상태 |
| `"pending"` | 데이터 파일 없음 |

## 대학 항목 추가 규칙

1. `in_seoul` 카테고리 대학은 기존 in_seoul 대학 마지막 항목 다음에 삽입
2. `regional_national` 카테고리 대학은 마지막에 삽입
3. 중복 추가 금지 — 먼저 `university_id`로 기존 항목 존재 여부 확인
4. 기존 항목 수정 시 `status`와 `source_doc`만 변경하는 경우가 대부분

## 렌더링 확인 포인트

`universities.json` 저장 후 `python3 -m http.server 8000`으로 서버를 띄워야 한다 (file:// CORS 차단). 확인 항목:
- 대학 카드가 목록에 표시되는지
- 배지 색상: 농어촌 단독(`standalone`) → 초록, 포함형(`included`) → 황토색
- 수시/정시 인원 숫자 표시 여부
- "확인되지 않음" 표시(`merged` 상태)가 올바른지
