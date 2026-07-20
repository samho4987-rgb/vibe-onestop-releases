---
name: pipeline
description: 매물 접수문자 처리의 전체 파이프라인을 오케스트레이션한다. "접수문자 처리해줘", "매물 파이프라인 실행", "이 문자 매물장에 넣어줘" 등 매물 접수·검증·콘텐츠 생성 전 과정 요청 시 사용. 연결규약 v1.2의 표준 시퀀스와 사람개입 3지점 원칙을 적용한다.
---

# vibe-onestop 파이프라인 오케스트레이션

매물 접수문자 → 검증완료 → 콘텐츠 패키지까지의 표준 시퀀스를 실행한다.
세부 규약(필드 소유권·스키마·발행 규약)은 `references/protocol.md`(연결규약 v1.2) 참조.

## 필요 MCP (별도 등록, 이 플러그인에 미포함)

vibe-ledger(저장·상태·검증) / vibe-geo(좌표·POI) / vibe-pubdata(실거래·건물검증·보증금안전) /
vibe-news(뉴스 글감) / realty-pipeline(save_package). 도구가 없으면 사용자에게 MCP 등록 상태를 확인시킨다.

## 표준 시퀀스

1. **접수 파싱** — `intake-parsing` skill 규칙으로 원문 → ledger JSON + 확인질문.
   원문은 `internal_memo`에 보존. `save_listing` → status 접수.
2. **공부 팩트** — building-register 조회 → `update_listing(register, address.road)`. 원문값 그대로, 가공 금지.
3. **가격 팩트** — real-estate / vibe-pubdata(`get_market_comps`, `verify_building`, 전월세는 `check_deposit_safety`)
   → `update_listing(market, finance.*.verified)`. claim과 verified를 항상 구분.
4. **입지 팩트** — vibe-geo 좌표·POI → `update_listing(address.coords, location_points)`.
   coords는 최초 1회 저장 후 재사용. location_points는 원시 팩트만(`{poi, distance_m, walk_min, category}`).
5. **셀링포인트** — location_points·market 기반으로 후보 3~4안 생성 → **사람 검수 후 확정분만** `add_selling_point`.
6. **사람 확인** — 임대인/매도인 광고 동의 확인 후 `update_status(검증완료)`.
7. **광고요건** — `check_ad_ready` 누락 0건 확인. 누락 시 항목별 보완 후 재검.
8. **콘텐츠 생성** — 검증완료 매물의 `get_listing`을 읽어 skill 4종 실행:
   `ad-copy`(이실장 문구) → `blog-post`(네이버 원고 md) → `youtube-script` → `sns-card`.
   contacts 그룹은 콘텐츠 입력에서 전체 제외.
9. **패키지 저장** — realty-pipeline `save_package`로 산출물 일괄 저장.
10. **광고 등록** — 이실장 RPA는 사람이 실행(폼 입력 후 등록 클릭은 사람). 완료 시 `update_status(광고중)`.

## 절대 규칙 (요약)

- **사람개입 3지점 자동화 금지**: ①임대인 광고 동의 확인 ②홍보문구 최종 검수 ③광고 등록 최종 실행.
- **ledger 쓰기는 오케스트레이터(Claude)만** `update_listing`/`add_selling_point`로 수행.
  status는 `update_status`로만 변경. 하류(RPA·콘텐츠)는 ledger 읽기 전용.
- **location_points(원시 팩트) ≠ selling_points(검수된 홍보 문장)** — 절대 혼용 금지.
- **개인정보**: 임차인 연락처는 `(비공개)` 표기 저장, 광고·콘텐츠에 절대 미출력.
  임대인 연락처는 사람만 입력(자동 수집 금지).
- **단위**: 금액=원(정수), 면적=㎡, 거리=m, 시간=분. 평·만원 변환은 표시 단계에서만.
- **ID**: `listing_id = YYYYMMDD-NNN`으로만 매물 지칭.

## 블로그 발행 (참고)

원고 md는 `## 제목(안)`·`## 본문` 필수, 매물글은 의무기재 체크리스트 섹션 필수.
발행은 vibe_blogpost.py(임시저장까지) + 사람 검수·발행. 뉴스글 발행 후 `vibe-news.mark_used`.
세부는 `references/protocol.md` 6절.
