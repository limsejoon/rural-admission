---
name: add-university
description: Rural Admission Wiki에 새 대학 농어촌 전형 데이터를 추가하는 전체 워크플로우 오케스트레이터. 새 대학 추가, 기존 대학 데이터 수정, 데이터 재추출, 검증, 사이트 동기화 요청 시 이 스킬을 사용할 것. "대학 추가", "JSON 만들어줘", "전형 데이터", "데이터 넣어줘", "다시 추출", "수정해줘", "검증해줘", "사이트에 반영" 등 데이터 파이프라인 관련 작업에 반드시 이 스킬을 사용할 것.
---

## 실행 모드: 서브 에이전트 (파이프라인 패턴)

data-intake → data-validator → site-sync 순서로 순차 실행한다. 각 에이전트 결과가 파일로 저장되어 다음 에이전트로 전달된다.

## Phase 0: 컨텍스트 확인

워크플로우 시작 전 기존 상태를 확인한다:
- `data/structures/{id}.json` 존재 여부
- `data/yearly/2028/{id}.json` 존재 여부
- `data/evidence/{id}.json` 존재 여부

**실행 모드 결정:**
- 3개 파일 없음 → **초기 실행** (Phase 1부터)
- 파일 있음 + 사용자가 특정 수정 요청 → **부분 재실행** (해당 Phase만)
- 파일 있음 + 새 정보 제공 → **갱신 실행** (기존 파일 백업 후 전체 재실행)

## Phase 1: 데이터 추출 및 구조화

`data-intake` 에이전트를 서브 에이전트로 호출한다 (`model: "opus"`).

**전달 정보:**
- 대학 ID, 대학명, 카테고리, 지역
- 사용자가 제공한 전형 정보 전문
- 출처 문서명

**기대 산출물:**
- `data/structures/{id}.json`
- `data/yearly/2028/{id}.json`
- `data/evidence/{id}.json`

## Phase 2: 검증

Phase 1 완료 후 `data-validator` 에이전트를 서브 에이전트로 호출한다 (`model: "opus"`).

**전달 정보:** 대학 ID

**오류 발견 시:**
- 오류 목록을 data-intake에 전달하여 수정 요청
- 수정 완료 후 재검증 (최대 1회 재시도)
- 재시도 후에도 오류 남으면 사용자에게 보고하고 계속 진행 (오류 명시)

## Phase 3: 사이트 동기화

Phase 2 검증 통과 후 `site-sync` 에이전트를 서브 에이전트로 호출한다 (`model: "opus"`).

**전달 정보:** 대학 ID, 대학명, 카테고리, 지역, 출처 문서명, 완성도 상태 (검증 결과 기반)

## 최종 보고

```
✓ {대학명} 추가 완료

생성 파일:
- data/structures/{id}.json — 전형 {N}개
- data/yearly/2028/{id}.json — 수시 {N}명 / 정시 {N}명
- data/evidence/{id}.json — 근거 {N}건 (confidence: unverified)

universities.json — status: {complete/partial}

다음 단계: 검수 콘솔(tools/review-console.html)에서 PDF 대조 후 confidence를 verified로 변경하세요.
```

## 에러 핸들링

- data-intake 실패 → 사유 보고 후 사용자에게 전형 정보 재입력 요청
- data-validator 오류 → 1회 수정 재시도, 재실패 시 오류 목록 보고 후 계속
- site-sync 실패 → 데이터 파일은 정상이므로 universities.json만 수동 수정 안내

## 테스트 시나리오

**정상 흐름:**
- 입력: "연세대학교 추가해줘. 수시 농어촌학생전형 45명(standalone, 면접 있음, 수능최저 없음), 정시 기회균형 농어촌 23명(specified)"
- 기대: 3개 JSON 파일 생성, 검증 통과, universities.json 업데이트

**에러 흐름:**
- 입력: "고려대 추가. 기회균형 100명" (rural_quota_status 불명확)
- 기대: data-intake가 `"pending"` 상태로 생성 + evidence에 미확인 메모 → 사용자에게 추가 정보 요청
