# vibe-onestop 플러그인 v1.4.1

부동산 매물접수 업무자동화 파이프라인. 연결규약 v1.2 기준.
**MCP 서버 4종이 단일 Windows 실행파일로 동봉** — 플러그인 설치 한 번으로 끝.
v1.4에서 서버 4개를 하나의 exe로 통합해 용량을 105MB → 32MB로 줄였습니다.

## 스킬 6종

| 스킬 | 역할 |
|---|---|
| pipeline | 접수→검증→콘텐츠→패키지 표준 시퀀스 오케스트레이션 (연결규약 내장) |
| intake-parsing | 접수문자·카톡 → ledger JSON + 확인질문 |
| ad-copy | 이실장 등록 홍보문구 (setMemo 40자 / setDetailMemo 1000자) |
| blog-post | 네이버 블로그 원고 md (vibe-blogpost 입력 규격) |
| youtube-script | 숏폼 60초·롱폼 스크립트 |
| sns-card | 인스타 문구 + Canva 카드뉴스 지시서 |

## 동봉 MCP 서버 4종 (servers/vibe-servers.exe 단일 파일, Windows x64 전용)

| 서버 | 역할 | 데이터 위치 |
|---|---|---|
| vibe-ledger | 매물장 저장·상태머신·광고요건 검증 | `~/.greencore/listings.db` |
| vibe-geo | 좌표·역세권·POI (카카오) | 키·캐시 `~/.greencore/` |
| vibe-pubdata | 건축물대장·실거래 비교·보증금 안전 (data.go.kr) | 키 `~/.greencore/` |
| vibe-news | 부동산 뉴스 수집·글감 (네이버 검색 API) | `~/.vibe-news/` |

`.mcp.json`이 `${CLAUDE_PLUGIN_ROOT}/servers/vibe-servers.exe`를 서버명 인자(ledger/geo/pubdata/news)와 함께 stdio로 실행합니다.
API 키는 설치 후 대화에서 `set_api_key` 도구로 등록 (카카오 REST / data.go.kr Decoding / 네이버).

- **macOS**: 동봉 exe는 실행 불가. 스킬만 사용하고 MCP 서버는 기존처럼 Python으로 별도 등록.
- building-register, real-estate, realty-pipeline MCP는 이 플러그인에 포함되지 않음 (별도 등록).

## 사용법

접수문자를 붙여넣고 "접수문자 처리해줘"라고 하면 pipeline skill이 표준 시퀀스를 실행합니다.
폴더 연결·HANDOFF 읽기 없이 동작합니다.

## 갱신 절차

- 규약·스킬 개정: 개발 폴더 skills/ 수정 → 재패키징·재설치
- 서버 코드 수정: 개발 폴더에서 수정 → Parallels Windows에서 build-win/RUN_BUILD_ALL.cmd 재빌드 → 재패키징
