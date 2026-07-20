# vibe-onestop 플러그인 릴리즈

부동산 매물접수 업무자동화 파이프라인 — Claude 플러그인 배포 저장소입니다.

## 설치

1. [Releases](../../releases)에서 최신 `vibe-onestop-x.y.z.plugin` 파일을 다운로드합니다.
2. Claude 데스크톱 앱(Cowork)에서 플러그인 파일을 열거나 설정 → 플러그인에서 설치합니다.
3. 끝. 별도의 Python 설치나 MCP 서버 등록이 필요 없습니다.

## 구성 (v1.3)

| 구성요소 | 내용 |
|---|---|
| 스킬 6종 | pipeline(오케스트레이션) · intake-parsing · ad-copy · blog-post · youtube-script · sns-card |
| MCP 서버 4종 (Windows exe 동봉) | vibe-ledger(매물장) · vibe-geo(입지) · vibe-pubdata(공공데이터 검증) · vibe-news(뉴스 글감) |

- **지원 OS**: 동봉 MCP 서버 실행파일은 **Windows x64 전용**입니다. macOS에서는 스킬만 동작하며, MCP 서버는 Python으로 별도 등록이 필요합니다.
- API 키(카카오·공공데이터포털·네이버)는 설치 후 대화에서 `set_api_key` 도구로 등록합니다.
- 매물 데이터는 로컬(`~/.greencore/listings.db`)에만 저장됩니다.

## 사용법

플러그인 설치 후 Claude에게:

> 접수문자 [원문 붙여넣기] 처리해줘

pipeline 스킬이 접수 → 검증 → 콘텐츠 생성 표준 시퀀스를 자동 실행합니다.
(임대인 광고 동의 확인 · 홍보문구 최종 검수 · 광고 등록 실행은 항상 사람이 합니다)
