# GooseRelayVPN 전수조사 분석 정리 (한국어)

> 이 문서는 GooseRelayVPN 저장소를 전수조사하며 나눈 대화를 정리한 기록입니다.
> 작성일: 2026-09-20

## 관련 링크

| 구분 | 주소 |
|---|---|
| 이 저장소 (포크) | https://github.com/bmshin94/GooseRelayVPN |
| 원본 저장소 | https://github.com/Kianmhz/GooseRelayVPN |
| 릴리즈 | https://github.com/Kianmhz/GooseRelayVPN/releases |
| 도커 이미지 (GHCR) | `ghcr.io/kianmhz/gooserelayvpn-server:latest` |
| 서버 설치 스크립트 | https://raw.githubusercontent.com/Kianmhz/GooseRelayVPN/main/scripts/goose-server.sh |
| 라이선스 | MIT (Copyright (c) 2026 Kian Haddad) |

---

## 1. 이게 뭐 하는 물건인가

**한 줄 요약:** 구글 앱스스크립트를 중계소로 삼아 raw TCP 트래픽을 내 VPS로 터널링하는 SOCKS5 기반 검열 우회 VPN.

감시자(ISP / 국가 방화벽) 입장에서는 클라이언트가 **구글 IP와 TLS 통신하는 것**으로만 보입니다.

### 데이터 흐름

```
브라우저/앱
  -> SOCKS5 (127.0.0.1:1080)
  -> Zstd 압축 + AES-256-GCM 배치 봉인
  -> HTTPS to 구글 엣지 IP (SNI=www.google.com, Host=script.google.com)
  -> Apps Script doPost()  (평문을 절대 못 보는 단순 포워더)
  -> 내 VPS :8443/tunnel   (복호화, session_id로 demux, 실제 대상에 dial)
  <- 롱폴링으로 역방향 동일 경로
```

### 폴더 전수조사 결과

| 경로 | 역할 | 비고 |
|---|---|---|
| `cmd/client/` | 내 컴퓨터에서 도는 SOCKS5 클라이언트 | main.go 230줄, 프리플라이트 진단 포함 |
| `cmd/server/` | VPS 출구 서버 진입점 | 68줄, 얇은 래퍼 |
| `internal/frame/` | AES-256-GCM 봉인 + Zstd 압축 + 배치 패커 | 암호화 심장부 |
| `internal/carrier/` | 롱폴링 루프 + 도메인 프론팅 HTTPS 클라이언트 | 932줄, 최대 규모 |
| `internal/exit/` | 복호화 → 세션 demux → 실제 `net.Dial` | 951줄 |
| `internal/session/` | 세션 상태머신, seq 카운터, 재조립, 백프레셔 | 550줄 |
| `internal/socks/` | SOCKS5 서버 + VirtualConn, no-op 리졸버(DNS 누출 차단) | |
| `internal/protocol/` | 양쪽 공통 와이어 상수 | 프레임 256KB, 배치 48/144 |
| `internal/config/` | JSON 설정 로더 | |
| `apps_script/Code.gs` | 구글 쪽 ~30줄 단순 포워더 | 평문/키를 절대 보지 않음 |
| `bench/` | E2E 벤치마크 하네스 + 베이스라인 3개 | 10% 회귀 시 CI 실패 |
| `scripts/` | systemd 유닛 + 원클릭 설치 스크립트 | SHA256 검증 포함 |
| `.github/workflows/` | CI(race + staticcheck) / GHCR 도커 / 8플랫폼 릴리즈 | |

전체 규모: 파일 73개, Go 코드 약 10,654줄.

### 4중 방어 구조

1. **하드코딩된 구글 엣지 IP** (`216.239.38.120`) — 로컬 DNS를 아예 안 씀 → DNS 하이재킹 무력화
2. **TLS 인증서 검증** — 구글 개인키 없이는 중간자 공격 불가, 실패 시 fail-closed
3. **AES-256-GCM 종단간 암호화** — 구글조차 평문을 못 봄. 16바이트 GCM 태그가 변조도 탐지
4. **no-op SOCKS5 리졸버** — 목적지 호스트명이 VPS에서 해석됨 → 무엇을 보는지 ISP가 모름

### 언제 쓰나

- 국가 단위 검열 환경 (원 프로젝트 타겟은 이란 — README_FA.md가 56KB 페르시아어)
- WireGuard / OpenVPN / V2Ray 등 일반 VPN 프로토콜이 DPI로 전부 차단된 곳
- 지역 차단 우회 및 IP 마스킹 (목적지 사이트는 VPS의 IP만 봄)
- HTTP 전용 프록시와 달리 **SSH, IMAP, 게임 등 모든 TCP** 통과 가능

### 한국 사용자에게 주는 실질적 가치

1. **최고급 Go 교과서** — ARCHITECTURE.md(25KB)에 "왜 이렇게 설계했는가"가 전부 문서화되어 있음
2. **인프라 패턴 보물창고** — 롱폴링 풀듀플렉스, 헬스 기반 지수 백오프 블랙리스트, 동일-폴 페일오버
3. **벤치마크 문화** — 베이스라인 커밋 + 회귀 시 CI 실패 구조를 자기 프로젝트에 이식 가능
4. **실사용** — 해외 서비스가 한국 IP를 막을 때 월 $4 VPS로 IP 세탁

### 한계

| 항목 | 내용 |
|---|---|
| 속도 | 구글 경유로 왕복 300~800ms 추가. 유튜브는 480p 권장 |
| 일일 한도 | 구글 **계정당** 약 20,000 UrlFetch/일 (배포 단위 아님). 텔레그램/X는 몇 시간이면 소진 |
| VPS 필수 | 실제 `net.Dial`이 일어날 서버가 반드시 필요 (월 $4면 충분) |
| 클라우드플레어 | 데이터센터 IP라 캡차 빈발 (버그가 아니라 정상) |
| 쿼터 리셋 | 태평양 자정 기준 |

---

## 2. 쉬운 비유 — "택배 세탁"

1. **문제** — 동네 우체국(ISP)이 해외 택배를 전부 열어보고 압수함. 단, "구글 본사행"만은 프리패스.
2. **짐 싸기** — 내 요청을 진공 압축(Zstd, 최대 65% 감소)하고 금고에 넣어 잠근 뒤(AES-256), 겉포장에 "구글 본사행"이라고 적음(도메인 프론팅).
3. **통과** — 우체국은 "구글 가는 거네, 통과" 하고 보내줌.
4. **구글 알바생** — 구글에 올려둔 30줄 스크립트가 금고를 받지만 열쇠가 없어 열지 못하고, 적힌 주소(내 VPS)로 그대로 전달.
5. **내 VPS** — 유일하게 열쇠를 가진 곳. 금고를 열어 진짜 목적지에 접속하고, 응답을 다시 금고에 넣어 역순 배송.

**왜 하필 구글인가:** 구글을 차단하면 나라 전체 인터넷이 마비되므로 차단할 수 없음. 그 틈을 파고드는 구조.

---

## 3. 질문 7개 정리

### Q1. 설치 및 사용법

**준비물:** 내 컴퓨터 + 해외 VPS(월 $4) + 구글 계정

**(A) VPS 서버 — 리눅스면 원클릭**
```bash
bash <(curl -Ls https://raw.githubusercontent.com/Kianmhz/GooseRelayVPN/main/scripts/goose-server.sh)
```
다운로드 → SHA256 검증 → 설정 → 키 생성 → systemd 등록 → 방화벽까지 한 번에 처리.
재실행하면 `install` / `update` / `uninstall` 메뉴 제공.

수동 설치:
```bash
openssl rand -hex 32                 # 1. 64자 AES-256 키 생성 (클라/서버 동일값)
# 2. server_config.json 작성
#    { "server_host": "0.0.0.0", "server_port": 8443, "tunnel_key": "생성한_64자" }
sudo ufw allow 8443/tcp              # 3. 방화벽 (클라우드 콘솔의 Security Group도 필수)
./goose-server -config server_config.json
curl http://내VPS아이피:8443/healthz # {"ok":true,...} 가 나오면 성공
```

도커: `server_config.json`을 **먼저** 만든 뒤
```bash
docker run -d --name goose-server --restart unless-stopped -p 8443:8443 \
  -v $(pwd)/server_config.json:/app/server_config.json:ro \
  ghcr.io/kianmhz/gooserelayvpn-server:latest
```

**(B) 구글 앱스스크립트**
1. https://script.google.com → **새 프로젝트** (반드시 새 프로젝트에 `Code.gs` 하나만)
2. `apps_script/Code.gs` 전체 붙여넣기
3. `RELAY_URLS`를 `['http://내VPS아이피:8443/tunnel']`로 수정
4. 배포 → 새 배포 → 웹 앱 / 실행: 나 / 액세스: 모든 사용자
5. 출력된 **배포 ID** 복사

> 코드를 수정할 때마다 반드시 **"새 배포"**를 만들어야 반영됨. 저장만으로는 적용되지 않음.

**(C) 클라이언트**
```json
{
  "socks_host": "127.0.0.1",
  "socks_port": 1080,
  "google_host": "216.239.38.120",
  "sni": "www.google.com",
  "script_keys": [{ "id": "배포ID", "account": "acct-a" }],
  "tunnel_key": "서버와_동일한_64자"
}
```
```bash
./goose-client -config client_config.json
# macOS: xattr -d com.apple.quarantine goose-client
# Windows: .\goose-client.exe  (./ 형태는 cmd.exe에서 동작하지 않음)
```
성공 시 로그:
```
CLIENT  INFO  pre-flight OK: relay healthy, AES key matches end-to-end
CLIENT  INFO  ready: local SOCKS5 is listening on 127.0.0.1:1080
```
브라우저 설정: SOCKS5 `127.0.0.1:1080` + **"SOCKS v5 사용 시 프록시 DNS" 반드시 체크** (미체크 시 DNS 누출).

**용량 늘리기:** 서로 다른 구글 계정으로 배포를 여러 개 만들고 `script_keys`에 `account` 라벨을 붙이면 계정 수만큼 일일 쿼터가 배가됨. 배포 1개당 폴 워커 3개, 계정 버킷당 idle 슬롯 기본 2개. 실용적 상한은 2~3계정.

### Q2. 플러그인 / 스킬 / MCP 중 무엇인가
**셋 다 아님.** 완전히 독립적인 Go 네트워크 애플리케이션입니다.

| 구분 | 여부 |
|---|---|
| 플러그인 | 아니오 |
| Claude Skill | 아니오 |
| MCP (Model Context Protocol) | 아니오 |
| **실체** | **단독 실행 CLI 바이너리 2개 + 구글 앱스스크립트 1개** |

유일하게 "플러그인스러운" 요소는 `apps_script/Code.gs`가 구글 앱스스크립트 플랫폼 위에서 실행되는 30줄짜리 서버리스 함수라는 점뿐입니다.
(저장소의 `CLAUDE.md`는 원본에 없는 파일로, 포크에서 추가된 페르소나 설정이며 프로젝트 기능과 무관합니다.)

### Q3. API 토큰이 필요한가
**전혀 필요 없습니다.**

| 항목 | 필요 여부 |
|---|---|
| 구글 API 토큰 / OAuth | 불필요 |
| GCP 프로젝트 / 결제 계정 | 불필요 |
| 서비스 계정 JSON | 불필요 |
| 인증서 / 비밀번호 | 불필요 |

필요한 건 두 가지뿐입니다.
1. `tunnel_key` — `openssl rand -hex 32`로 직접 만든 64자 AES-256 키
2. 앱스스크립트 배포 ID — 배포 시 발급되는 공개 URL의 일부

인증 방식이 영리합니다. **AES-GCM 인증 태그 자체가 인증**이어서, 키 없이 만든 데이터는 `Open()`에서 실패하고 조용히 폐기됩니다. 별도 비밀번호 체계가 필요 없는 구조입니다.

> 반대로 비밀번호도, 레이트 리밋도, 사용자별 계정도 없습니다. `tunnel_key` 유출은 곧 VPS 전체 공개를 의미하므로 서버 root 비밀번호처럼 취급해야 합니다.

### Q4. 왜 깃허브에서 유명한가
(정확한 스타 수는 이 환경에서 GitHub API 403으로 확인하지 못했습니다. 아래는 코드베이스 전수조사에 근거한 분석입니다.)

1. **절박한 수요 + 유일한 해법** — 일반 VPN이 전부 막힌 환경에서 "구글은 막을 수 없다"는 틈을 정확히 공략. 56KB 분량의 페르시아어 README가 타겟층을 명확히 보여줌
2. **기존 앱스스크립트 프록시와의 차별점** — 기존 것들은 HTTP만 지원, 이건 **raw TCP** 전체를 터널링
3. **문서 품질** — README 42KB, ARCHITECTURE.md 25KB. 상수 하나하나의 선택 이유까지 설명. v1.7.0 base64 버그, v1.6 동시성 회귀 등 실패 이력도 코드 주석에 정직하게 기록
4. **정직한 위협 모델** — "방어하지 못하는 것" 5가지(VPS 침해, 클라이언트 침해, 구글 계정 압수, DPI 패턴 핑거프린팅, 쿼터 고갈 DoS)를 별도 섹션으로 명시
5. **즉시 사용 가능** — 8개 플랫폼 릴리즈 바이너리, 원클릭 설치 스크립트, 도커 이미지, SHA256 검증
6. **엔지니어링 성숙도** — CI에 race detector + staticcheck, 벤치마크 회귀 방지 시스템, MIT 라이선스

### Q5. 로컬 에이전트 구축에 도움이 되는가
**직접적인 AI 에이전트 기능은 전혀 없습니다.** 다만 에이전트 인프라 설계에 그대로 이식할 수 있는 패턴이 매우 많습니다.

| # | 패턴 | 위치 | 에이전트 응용 |
|---|---|---|---|
| 1 | 헬스 기반 지수 백오프 블랙리스트 (3s→6s→12s→48s) | `internal/carrier/client_blacklist_test.go` | LLM API 키 다중 로테이션, 429 시 자동 제외 |
| 2 | 동일-폴 페일오버 (같은 사이클 내 재시도, 유실 0) | `internal/carrier/client.go` | 모델 폴백을 유저 모르게 처리 |
| 3 | 계정별 버킷 세마포어 (`idle_slots_per_bucket`) | `internal/carrier/` | 워커 수와 동시성 상한의 분리 — 레이트 리밋 회피 |
| 4 | 적응형 코얼레싱 (`coalesce_step_ms`, 25ms 윈도우) | `internal/exit/exit.go` | 툴콜/임베딩 요청 배칭 → API 비용 절감 |
| 5 | 롤백 가능한 트랜잭셔널 drain | `internal/session/session.go` | 작업 큐의 exactly-once 보장 |
| 6 | 프리플라이트 진단 (시작 시 전 구간 검증) | `cmd/client/diagnostics.go` | 에이전트 부팅 시 MCP/API키/DB 연결 일괄 점검 |

추가로 실용적 용도 하나 — 에이전트가 **지역 차단된 API**를 호출해야 할 때, 에이전트 코드를 한 줄도 고치지 않고 `ALL_PROXY=socks5h://127.0.0.1:1080` 환경변수만으로 우회할 수 있습니다.

### Q6. 수익화 아이디어가 있는가
있습니다. 다만 **"터널 자체"가 아니라 "터널 주변"**에서 찾아야 합니다. 상세 내용은 아래 4장 참조.

### Q7. React나 PHP로 만들 수 있는가

| 컴포넌트 | React | PHP | 판정 |
|---|---|---|---|
| SOCKS5 클라이언트 | 불가능 | 가능하나 최악 | Go 유지 |
| 앱스스크립트 중계기 | — | — | 이미 JS |
| VPS 출구 서버 | Node면 가능 | 가능(비추천) | Go 유지 |
| **관리 대시보드 (웹 UI)** | **최적** | **적합** | **여기를 만들자** |

**React가 안 되는 이유:** 브라우저 샌드박스에서는 `net.Listen()`으로 TCP 포트를 열 수 없고, raw TCP 소켓이 없으며(WebSocket만 존재), 임의 SNI 지정이 불가능합니다. SNI 조작은 도메인 프론팅의 핵심이므로 치명적입니다.
단, **Electron / Tauri** 환경이라면 UI는 React로 만들고 기존 Go 바이너리를 그대로 구동하는 구성이 가능하며, 이는 매우 좋은 선택입니다.

**PHP가 힘든 이유:** `stream_socket_server` / Swoole / ReactPHP로 기술적으로는 가능하지만, 수백 개 동시 세션에서 Go 고루틴 대비 성능 격차가 크고, Go의 단일 바이너리 크로스컴파일(안드로이드 Termux 포함 8플랫폼) 이점을 잃으며, AES-GCM + Zstd 배치 처리가 PHP에서는 성능 병목이 됩니다.

**권장 구성:**
```
[ React 대시보드 ] <-> [ PHP 또는 Node API ] <-> [ Go 바이너리 (그대로) ]
```

React로 만들 것: 실시간 쿼터 게이지, 배포 헬스 대시보드, 처리량/레이턴시 차트, 설정 마법사(폼 → `client_config.json` + `Code.gs` 자동 생성), 원클릭 연결 토글
PHP로 만들 것: `/healthz` 폴링 및 적재, 멀티 VPS 통합 관리 API, 사용자/키 발급 관리, 설치 스크립트 생성기

핵심은 **Go 코어는 건드리지 않고 그 위에 웹 레이어를 얹는 것**입니다.

---

## 4. 수익화 아이디어 상세

### 먼저 — 하면 안 되는 것

| 금지 | 이유 |
|---|---|
| 이 코드로 유료 VPN 서비스 운영 | 구글 앱스스크립트 ToS 위반. 무료 쿼터의 상업적 중계 이용은 명시적으로 금지 |
| 구독형 VPN 장사 | 계정당 20,000/일 한도로 **구조적으로 스케일 불가** |
| 검열 국가 대상 판매 | 국가별 VPN 규제 위반. 법적 리스크가 사업 리스크보다 큼 |
| 원본 무단 재판매 | MIT라 법적으론 가능하나 저작자 크레딧 필수, 평판 리스크 |

### 아이디어 1 — "Goose Control" 셀프호스팅 관리 대시보드 (추천 1순위)
난이도 中 / 수익성 高 / 합법성 문제없음

현 프로젝트 최대 약점은 **UI가 전혀 없다는 것**(터미널 로그뿐)인데, 타겟 유저 상당수는 비개발자입니다.

- React + Tauri 데스크톱 앱 (Go 바이너리 번들)
- 설정 마법사: 폼 입력 → `client_config.json` + `Code.gs` 자동 생성
- 계정별 쿼터 게이지 + 태평양 자정 리셋 카운트다운
- 배포 헬스/블랙리스트 실시간 모니터링, 원클릭 시작·정지

| 티어 | 가격 | 내용 |
|---|---|---|
| Free | $0 | 배포 1개, 기본 UI |
| Pro | $15 (평생) | 무제한 배포, 쿼터 분석, 자동 업데이트 |
| Team | $49 | 멀티 VPS 관리, 통계 내보내기 |

터널이 아니라 **관리 도구**를 파는 구조여서 합법적입니다. Gumroad / Lemon Squeezy로 즉시 판매 가능.

### 아이디어 2 — VPS 셋업 대행 서비스
난이도 低 / 수익성 中 / 합법성 문제없음

"VPS 구매 → SSH → 스크립트 실행 → 앱스스크립트 배포" 과정에서 대부분 이탈합니다.
- 서비스: 고객 VPS에 전체 세팅 + 검증 + 사용법 안내
- 가격: 건당 $20~30, 또는 월 $5 유지보수(업데이트/모니터링)
- 채널: Fiverr / 크몽 / 숨고에 "VPN 서버 구축 대행" 등록

> 주의: **구축 대행이지 서비스 운영이 아니어야 합니다.** VPS와 앱스스크립트 계정 모두 고객 소유여야 하며, 이 선을 넘으면 ToS·법적 문제가 발생합니다.

### 아이디어 3 — 기술 교육 콘텐츠
난이도 低 / 수익성 中 / 합법성 문제없음

- 유튜브 시리즈: "Go로 만드는 프로덕션급 네트워크 터널" (10편)
- 유료 이북 ($25): 도메인 프론팅, AES-GCM 배치 설계, 롱폴링 풀듀플렉스
- 인프런 / 유데미 강의 ($60): 10,654줄 코드 해부

한국어 "실전 네트워크 프로그래밍" 콘텐츠가 희소하다는 점이 강점입니다. 베스트셀러급이면 월 $500~2,000 수준.

### 아이디어 4 — 패턴을 SaaS 부품으로 재조립 (추천 2순위)
난이도 中高 / 수익성 高 / 합법성 문제없음

터널과 무관하게 **패턴만 추출**해 별도 제품을 만드는 접근입니다.

**제품 A — "KeyRotator": LLM API 키 로테이션 프록시**
- 블랙리스트 + 라운드로빈 + 동일-폴 페일오버 패턴 이식
- OpenAI / Anthropic / Gemini 키 여러 개를 단일 엔드포인트로 통합
- 429 발생 시 자동 백오프 + 다른 키로 재시도 (호출자는 인지하지 못함)
- 월 $9~29 SaaS

**제품 B — "BatchGate": 요청 코얼레싱 게이트웨이**
- `coalesce_step_ms` 아이디어를 임베딩/툴콜 API에 적용
- API 호출 횟수 30~50% 감소 → 직접적 비용 절감
- 절감액의 10% 과금 또는 월 $19

구글 ToS와 무관하고 법적 리스크가 없으며 시장이 성장 중입니다.

### 아이디어 5 — 기업용 "제한 환경 접속 툴킷"
난이도 高 / 수익성 高 / 계약 및 법률 검토 필수

합법적 수요처: 학술 연구자(검열 국가 현장 조사), 언론사 특파원, NGO, 보안 회사(지역별 접근성 테스트).
- 상품: 온프레미스 라이선스 + 컨설팅 + 교육 + SLA
- 가격: 연 $3,000~15,000
- 전제: 구글 대신 **자체 프론팅 인프라**로 전환하여 ToS 문제를 제거해야 함

### 아이디어 6 — 웹/모바일 프론트엔드 생태계
난이도 中 / 수익성 中 / React·PHP 역량 직결

- **안드로이드 앱**: 현재는 Termux CLI로만 동작하며 정식 앱이 없음 (VpnService API + Go 바이너리 번들)
- **Tauri 데스크톱 앱**: React UI + Go 코어
- **설정 생성기 웹사이트**: 폼 입력 → config 2개 + `Code.gs` 다운로드 (무료 배포로 트래픽 확보 후 Pro 전환)

수익: 앱스토어 유료앱 $4.99 또는 무료 + Pro 인앱결제

### 아이디어 7 — 오픈소스 후원 모델
난이도 低 / 수익성 低~中 / 합법성 문제없음

원본도 이미 암호화폐 후원(TRX / BNB / TON)을 받고 있습니다.
포크에 **한국어 문서 + 관리 UI + 안드로이드 앱**을 추가해 별도 프로젝트로 성장시키고 GitHub Sponsors / Open Collective로 후원을 받는 방식. 당장의 수익은 작지만 아이디어 1·6의 유입 깔때기 역할을 합니다.

### 권장 로드맵

```
1개월차  : 아이디어 3 (교육 콘텐츠)   -> 이해도 확보 + 인지도 + 첫 수익
2~3개월  : 아이디어 1 (관리 대시보드) -> React 강점 활용, $15 평생 라이선스
4~6개월  : 아이디어 4 (KeyRotator)    -> 실제로 스케일되는 SaaS
병행     : 아이디어 2 (셋업 대행)     -> 즉시 현금흐름 + 고객 인터뷰 소스
```

### 핵심 원칙

1. **"터널 판매"는 하지 않는다** — ToS 위반 및 법적 리스크
2. **"도구 / 지식 / 패턴"을 판다** — 합법적이고 마진이 높음
3. **기존 강점(React / PHP)이 살아나는 레이어를 공략한다** — Go 코어는 그대로 두고 그 위에 얹기

---

## 부록 — 저장소 기본 정보

| 항목 | 값 |
|---|---|
| 언어 | Go 1.22 |
| 주요 의존성 | `github.com/things-go/go-socks5`, `github.com/klauspost/compress` (Zstd), `golang.org/x/net` |
| 파일 수 | 73개 |
| Go 코드 | 약 10,654줄 |
| 라이선스 | MIT |
| 지원 플랫폼 | linux (amd64/arm64/armv7), windows (amd64/arm64), darwin (amd64/arm64), android (arm64) |
| 와이어 포맷 | `session_id(16) \|\| seq(u64 BE) \|\| flags(u8) \|\| target_len(u8) \|\| target \|\| payload_len(u32 BE) \|\| payload` |
| 배치 봉인 | `nonce(12) \|\| AES-GCM(u16 frame_count \|\| [u32 frame_len \|\| frame_bytes]...)` — 배치당 1회 봉인 |
| HTTP 바디 | `base64(nonce \|\| ciphertext+tag)` (RawStdEncoding, 패딩 없음) |
