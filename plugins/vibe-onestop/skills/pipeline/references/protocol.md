# vibe-onestop 연결규약

버전: 1.2 (2026-07-20) · 매물접수 업무자동화 파이프라인 모듈 간 인터페이스 규약

---

## 1. 핵심 원칙

1. **단일 진실 저장소**: 매물 데이터의 원본은 vibe-ledger(`~/.greencore/listings.db`) 하나뿐이다.
   다른 모듈은 자체 매물 DB를 갖지 않는다 (캐시는 허용, 원본 아님).
2. **조회기는 무상태(stateless)**: vibe-geo, building-register, real-estate 등 조회 모듈은
   "입력 → 결과 반환"만 한다. ledger에 직접 쓰지 않는다.
   **ledger에 쓰는 주체는 항상 오케스트레이터(Claude)다.**
3. **MCP와 skill의 분리**: 데이터 조회·저장·실행 = MCP / 규칙·템플릿·절차 = skill.
   법령·양식이 바뀌면 skill 문서만 고친다. 코드는 건드리지 않는다.
4. **사람 개입 3지점은 자동화 금지**: ①임대인 광고 동의 확인 ②홍보문구 최종 검수
   ③광고 등록 최종 실행. 이 지점을 우회하는 기능은 어떤 모듈에도 만들지 않는다.

## 2. 모듈 정의

| 모듈 | 유형 | 역할 (한 줄) | 상태 |
|---|---|---|---|
| **vibe-ledger** | MCP | 매물 저장·상태머신·의무기재 검증 — 파이프라인의 중심 저장소 | ✅ 완성 |
| **building-register** | MCP | 건축물대장 조회 (공부 팩트 공급) | ✅ 보유 |
| **real-estate** | MCP | 실거래 시세·공시가격 조회 (가격 팩트 공급) | ✅ 보유 |
| **vibe-geo** | MCP | 주소→좌표, 역·POI 거리 조회 (입지 팩트 공급) | ✅ 완성 |
| **이실장 RPA** | 스크립트 | 매물 JSON → 이실장 등록 폼 자동 입력 (등록 클릭은 사람) | 🔜 3단계 |
| **realty-pipeline** | MCP | 블로그·스크립트·SNS 산출물 패키지 저장 | ✅ 보유 |
| **콘텐츠 skill 세트** | skill | 접수 파싱 규칙, 홍보문구 의무기재·톤, 블로그·유튜브·SNS 템플릿 (`skills/` 5종) | ✅ 완성 |
| **vibe-pubdata** | MCP | 실거래 비교사례·건물검증·보증금 안전 (검증 팩트 공급) | ✅ 완성 |
| **vibe-news** | MCP | 뉴스 수집·글감 선정·사용 관리 (~/.vibe-news/news.db) | ✅ 완성 |
| **vibe-blogpost** | 스크립트 | 원고 md+이미지 → 네이버 블로그 임시저장 (발행은 사람) | ✅ v0.3 |
| **vibe-cardnews** | 스크립트 | 매물 JSON → 카드뉴스 PNG 세트 | 🔜 개발 예정 |

## 3. 스키마 필드 소유권

"소유"란 해당 필드에 값을 공급할 책임. 실제 쓰기는 모두 Claude가
`update_listing`/`add_selling_point`로 수행한다.

| 필드 그룹 | 공급자 | 비고 |
|---|---|---|
| `source`, `internal_memo` | 접수 파싱 (skill) | 원문 보존 필수, 수정 금지 |
| `address.*` (jibun/코드) | 접수 파싱 | |
| `address.road`, `register.*` | building-register | 공부 원문값 그대로. 가공 금지 |
| `address.coords` (위경도) | **vibe-geo** | 최초 1회 저장 후 재사용 (재조회 방지) |
| `location_points[]` | **vibe-geo** | 원시 팩트만: `{poi, distance_m, walk_min, category}` |
| `market.*` (비교사례) | real-estate | `{comps[], median, comment}` — 신설 그룹 |
| `finance.*.verified` | real-estate(공시가) + 산식 skill | claim(주장)과 verified(검증)를 항상 구분 |
| `deal.*`, `features.*` | 접수 파싱 + 사람 확인 | |
| `selling_points[]` | Claude 생성 + **사람 검수 후 확정** | location_points를 가공한 홍보 문장 |
| `contacts.임대인` | 사람만 입력 | 자동 수집 금지 |
| `status` | vibe-ledger 상태머신 | `update_status`로만 변경. 직접 수정 금지 |

**location_points(원시 팩트) vs selling_points(홍보 가공)는 절대 혼용하지 않는다.**
전자는 기계가 재검증 가능해야 하고, 후자는 사람 검수를 거친 문장이다.

## 4. 데이터 흐름 (표준 시퀀스)

```
접수 문자
 → [파싱 skill] JSON + 확인질문 → ledger.save_listing        (status: 접수)
 → [building-register] 조회 → ledger.update_listing(register)
 → [real-estate] 시세·공시가 → ledger.update_listing(market, finance.verified)
 → [vibe-geo] 좌표·POI → ledger.update_listing(coords, location_points)
 → [Claude] 셀링포인트 생성 → 사람 검수 → ledger.add_selling_point
 → 사람: 임대인 동의 확인 → ledger.update_status(검증완료)
 → ledger.check_ad_ready 통과 → update_status(광고중)
 → [이실장 RPA] ledger.get_listing 읽기 전용 → 폼 입력 → 사람이 등록 클릭
 → [콘텐츠 skill] get_listing 읽기 → 블로그·스크립트·SNS 생성
 → [realty-pipeline] save_package
```

규칙: **하류 모듈(RPA·콘텐츠)은 ledger를 읽기만 한다.** 광고 등록 결과·발행 URL 등
기록이 필요하면 Claude가 `internal_memo` 또는 `history`(memo)로 남긴다.

## 5. 공통 규약

- **ID**: `listing_id = YYYYMMDD-NNN`. 모든 모듈은 이 ID로만 매물을 지칭한다.
- **단위**: 금액 = 원(정수), 면적 = ㎡(소수), 거리 = m, 시간 = 분. 평·만원 변환은 표시 단계에서만.
- **에러**: 모든 MCP 도구는 실패 시 `{"error": "..."}` 반환 (예외 던지지 않음).
  정부 API 호출부는 500 응답 대비 재시도 1~2회 내장 (건축물대장 게이트웨이 이슈).
- **개인정보**:
  - 임차인 연락처는 `contacts.임차인`에 `(비공개)` 표기와 함께 저장, 광고·콘텐츠 산출물에 절대 미출력
  - 콘텐츠 생성 skill은 `contacts` 그룹 전체를 입력에서 제외
  - 테스트 코드·레포에는 가짜 데이터만 사용 (010-0000-0000, 가상 주소)
- **네이밍**: 모듈 폴더 = `vibe-*` 소문자 하이픈. 도구명 = snake_case 동사형.

## 6. 콘텐츠 발행 규약 (v1.1 신설)

### 6.1 사무소 프로필 (공용 파일)

- 경로: `~/.vibe/office.json` — vibe 도구군 공용. 필드: `name, reg_no, owner, address, phone`
- 입력 주체: 사람만 (`vibe_blogpost.py --setup` 대화형 마법사. 향후 vibe-onestop GUI에서 동일 파일 편집)
- 소비자: vibe-blogpost(매물글 의무기재 블록 자동 삽입), 향후 vibe-cardnews·이실장 RPA
- 프로필 존재 시 의무기재 게이트에서 '중개사무소' 항목 자동 충족 처리

### 6.2 원고 md 규격 (Claude ↔ vibe-blogpost 계약)

- `## 제목(안)` 과 `## 본문` 섹션 필수 — vibe-blogpost는 이 두 섹션만 파싱한다
- **매물(광고) 글**: `## … 의무기재 …` 헤딩 아래 체크리스트(`- [ ]`/`- [x]`) 섹션 필수.
  미완료 항목이 있으면 vibe-blogpost가 실행을 차단한다 (--skip-check는 검토용만)
- **뉴스(논평) 글**: 의무기재 섹션 없음. 구조는 "사실 도입 → 세 줄 요약 → 현장 논평(+데이터 근거) → 출처 링크 → 해시태그". 기사 전재 금지
- 문체: 연결 폴더의 `블로그_스타일가이드.md` 최우선 적용 (daily-news-draft 예약 작업 포함)

### 6.3 발행 흐름

```
원고 md (+cardnews PNG) → vibe_blogpost.py → 게이트(의무기재) → 스마트에디터 입력 → 임시저장
 → 사람: 검수 → 발행 → (뉴스글이면) vibe-news.mark_used(news_id, post_url)
```

- 발행 자동화 금지 (사람개입 3지점 원칙의 연장). 하루 발행 1~2건 권장 (계정 보호)
- 네이버 UI 변경 대응: 셀렉터는 `~/.vibe-blogpost/selectors.json` 오버라이드로만 수정

## 7. 규약 변경 절차

스키마 필드 추가는 자유 (JSON 저장이므로 마이그레이션 불필요).
단, **필드 소유권 변경·상태머신 변경·사람개입 3지점 관련 변경**은
이 문서를 먼저 수정하고 버전을 올린 뒤 코드에 반영한다.

| 버전 | 일자 | 변경 |
|---|---|---|
| 1.0 | 2026-07-16 | 최초 작성 (ledger 완성 시점) |
| 1.1 | 2026-07-19 | 콘텐츠 발행 규약 신설 (office.json·원고 md 규격·발행 흐름), 모듈 표 갱신 (geo 완성, pubdata·news·blogpost·cardnews 추가) |
| 1.2 | 2026-07-20 | 콘텐츠 skill 세트 완성 처리 (`skills/` 5종: intake-parsing·ad-copy·blog-post·youtube-script·sns-card) |
