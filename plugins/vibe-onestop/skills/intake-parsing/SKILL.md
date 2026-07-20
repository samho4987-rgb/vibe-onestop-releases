---
name: intake-parsing
description: 매물 접수문자·카톡 원문을 vibe-ledger 스키마 JSON으로 파싱하고 확인질문을 생성한다. 접수 문자를 받으면 이 규칙을 최우선 적용. 원문은 internal_memo에 보존하고 save_listing 후 확인질문을 사용자에게 제시한다.
---

# 매물접수 파싱 skill (vibe-onestop 4단계 ①)

접수 원문 → ledger JSON + 확인질문. 연결규약 v1.1 3·4절 준수.

## 절차

1. 원문 전체를 `internal_memo`에 그대로 보존 (수정 금지, 접수 경로 표기: 문자/카톡/전화메모)
2. 아래 매핑 규칙으로 JSON 생성 → `save_listing` (status: 접수)
3. 누락·모호 항목마다 확인질문 생성 → 사용자에게 제시 (추측으로 채우지 않는다)
4. 후속: verify_building → get_market_comps → get_location_points 순 (각 결과는 Claude가 update_listing으로 저장)

## 필드 매핑 규칙

| 원문 표현 | 필드 | 규칙 |
|---|---|---|
| 주소(도로명/지번) | `address.road` 또는 `address.jibun` | 한쪽만 있으면 geocode로 상호 변환. 산번지는 `산` 유지 |
| 건물명·호수·층 | `unit.building / ho / floor` | "복층" 언급 시 `unit.duplex: true` |
| 매매가·종매도가 | `deal.price` | **원 단위 정수** (1억8천 → 180000000) |
| 보증금/월세 | `deal.deposit / rent` | 매매+임차승계면 `tenant.현임차_보증금/현임차_월세`로 분리 — deal과 혼용 금지 |
| 전세 | `deal.type: "전세"`, rent 0 | |
| 관리비 | `deal.mgmt_fee` | 원 단위. "없음"은 0, 미언급은 확인질문 |
| 융자·대출 | `deal.은행융자금` | |
| 면적 | `register.전용_sqm` 우선 | 평 표기는 참고값으로 `전용_평` 병기. **광고 면적은 대장 조회 후 대장값 기준** |
| 추천업종·옵션 | `features.*` | 원문 표현 유지 |
| 연락처 | `contacts.*` | 의뢰인 번호만. 임차인 연락처는 `(비공개)` 표기. **광고동의 여부는 반드시 확인질문** |
| 수익률·투자수치 주장 | `tenant.수익률_주장` 등 claim 필드 | 검증값은 `finance.*.verified`에 별도 저장 — claim/verified 절대 혼용 금지 |

## 확인질문 생성 규칙

다음이 원문에 없으면 반드시 질문 목록에 포함:

- 거래유형 확정 (매매/전세/월세 — "매매/임대 택1"이면 우선순위)
- 광고 동의 (매매→매도인, 임대→임대인) — **사람 확인 전 광고중 전환 불가**
- 관리비, 주차, 입주(명도) 시기, 욕실수·방향 (이실장 필수 입력값)
- 임차 승계 여부와 만기일 (매매 + 현임차 존재 시)
- 사진 보유 여부

## 금지

- 원문에 없는 값 추측 입력 금지
- location_points·selling_points 이 단계 생성 금지 (후속 모듈 소관)
- 임차인 개인정보 저장 시 `(비공개)` 누락 금지
