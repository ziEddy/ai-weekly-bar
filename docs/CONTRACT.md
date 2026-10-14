# AI Weekly Bar 구현 계약 — macOS · Windows

- 상태: **초안 v0.1** (2026-10-14). 소스 저장소가 생기면 그쪽 `docs/contract/`로 옮긴다.
- 근거: 1.3.1 베타(빌드 9) 실행 파일의 문자열·링크 프레임워크, `README.md`·`INSTALL.md`·`PRIVACY.md`·`RELEASE_STATUS.md`,
  캐시 소비자인 Project Monitor `modules/ai_usage.py`. **소스를 읽고 쓴 문서가 아니다.** 소스로 확인해야 할 항목은 ⚠로 표시했다.
- 이 문서가 정하는 것은 **두 구현이 같아야 하는 동작**이다. 코드는 공유하지 않는다(macOS는 Swift, Windows는 별도 구현).
  UI 모양·프레임워크·언어는 각 플랫폼이 정하고, 여기 적힌 입력·출력·규칙만 같으면 된다.

## 0. 한 줄 요약

사용자의 공식 클라이언트에서 **잔여 한도**를 읽어 상주 아이콘 한 칸에 보이고, 클릭하면 서비스별 상세·작업 바로가기·시스템 수치를 보인다.
모델에 메시지를 보내지 않고, 토큰을 추출하지 않고, 개발자 서버가 없다.

## 1. 용어

| 용어 | 뜻 |
|---|---|
| service | `codex` · `claude` · `antigravity`. 식별자는 이 소문자 문자열로 고정한다 |
| quota (창) | 한 서비스 안의 한도 하나. `id` · `name` · `window` · `used` · `reset`를 가진다 |
| remaining | **남은 %**. `floor(clamp(100 − used, 0, 100))`. 쓴 %가 아니다 — 캐시 소비자가 남은 %로 읽는다 |
| reset | 창이 초기화되는 시각. Unix epoch 초(실수 허용) |
| lastSuccessfulRefresh | 마지막으로 **성공한** 조회 시각. 실패한 조회는 이 값을 바꾸지 않는다 |
| stale (오래된 값) | §3.3의 규칙에 걸린 값. 숫자는 남기되 색을 칠하지 않고 「갱신 안내」를 붙인다 |
| 대표 창 | 상주 아이콘에 찍는 창. 서비스별로 §4에 정한다 |

## 2. 화면 구조 (양쪽 동일)

- 상주 아이콘 한 칸: **선택한 서비스의 대표 창 remaining**과 서비스 아이콘. 값이 없으면 `—%`.
  - macOS: 메뉴바 `NSStatusItem` 텍스트. Windows: 알림 영역 아이콘에 **숫자를 그려 넣는다**(작업표시줄 텍스트 API는 Windows 11에 없다).
- 팝업: 상단 화면 전환 **AI 사용량 / 시스템**. 첫 실행은 시스템, 그 뒤는 마지막 선택을 기억한다.
- AI 사용량 화면: 서비스 탭 `Codex / Claude / Antigravity`(순서 고정), 창 목록(remaining 막대·초기화까지 남은 시간·날짜), 작업 목록(§5).
- 시스템 화면: §6의 수치. 「활성 상태 보기」(macOS) / 「작업 관리자」(Windows) 열기 버튼.
- 하단 전원 버튼으로 종료.
- 단축키: 서비스 전환 `⌘1/⌘2/⌘3` ↔ `Ctrl+1/2/3`, 새로고침 `⌘R` ↔ `Ctrl+R`. 새로고침은 **선택한 서비스만** 조회한다.
- 기억하는 설정: 선택 서비스 · 선택 화면 · 서비스별 작업 목록 탭 · 서비스별 선택 프로젝트. 그 외는 저장하지 않는다.

## 3. 공통 규칙

### 3.1 갱신

| 때 | 동작 |
|---|---|
| 실행 · 잠자기 해제 | 세 서비스 모두 조회 |
| 주기 | **서비스별 독립 10분**. 한 서비스의 실패·지연이 다른 서비스를 막지 않는다 |
| 수동 새로고침 | 선택한 서비스만 |
| 오류 뒤 | 재시도 간격 `1 → 2 → 4 → 8 → 16 → 30분`(상한 30). 성공하면 10분으로 복귀 ⚠(1.0.2 규칙 — 1.3.1에서 유지되는지 소스 확인) |
| 인증 오류 | 그 서비스의 자동 조회를 **멈춘다**. 사용자가 로그인 후 수동 새로고침해야 재개 |
| 시스템 수치 | 시스템 화면이 **보일 때만** 2초 샘플링, 저장 공간은 30초. 화면을 닫거나 AI 사용량으로 전환하면 즉시 중단 |

### 3.2 오류 범주 (고정 집합)

원시 stderr·응답 본문은 **화면에도 파일에도 내지 않는다.** 메모리에 최대 8KB만 들고 아래 범주로 바꾼다.

| 범주 | 뜻 | 자동 조회 |
|---|---|---|
| `notInstalled` | 클라이언트 실행 파일을 못 찾음 | 계속(백오프) |
| `notRunning` | Antigravity 앱이 안 떠 있음 | 계속(백오프) |
| `auth` | 로그인 안 됨 · 토큰 만료 | **중단** |
| `rateLimited` | 클라이언트가 요청 제한을 돌려줌 | 계속(백오프) |
| `protocol` | 응답 형식이 계약과 다름(CLI 업데이트 의심) | 계속(백오프) |
| `accountSelection` | 서버·계정·조직이 둘 이상 발견 — 임의로 합치지 않음 | 중단 · 안내 |
| `timeout` | 응답 없음 | 계속(백오프) |
| `unknown` | 그 외 | 계속(백오프) |

### 3.3 신선도

값은 다음 중 하나면 stale이다. stale이면 숫자는 남기고 색은 없애고 「갱신 안내」를 붙인다.

1. `reset`이 현재 시각 이하 — 초기화가 지났는데 새 값이 아직 없다.
2. `lastSuccessfulRefresh`가 없다.
3. `now − lastSuccessfulRefresh > 720초`(12분 = 주기 10분 + 여유). 캐시 소비자(Project Monitor)와 같은 값이다.

### 3.4 표시 형식

- remaining 색: `< 10` 멈춤(빨강 계열), `< 25` 주의(노랑 계열), 그 외 서비스 틴트. 정확한 색은 플랫폼 자유.
- 남은 시간(`reset − now`):

| 남은 초 | 표시 |
|---|---|
| `≤ 0` | `갱신 중` |
| `≥ 86400` | `N일 N시간` |
| `< 60` | `1분 미만` |
| `< 3600` | `N분` |
| 그 외 | `N시간 N분` |

- 날짜는 사용자 로캘의 짧은 형식 + 요일. 계산은 **기기 시계** 기준이다.
- 모델·그룹별 창은 대표 창 아래 행으로 붙이고, 초기화 시각이 대표 창과 다를 때만 남은 시간을 따로 적는다.

### 3.5 개인정보 (양쪽 동일, PRIVACY.md와 일치해야 한다)

- 토큰·쿠키·API 키·OAuth 파일을 읽지 않는다. Antigravity CSRF 값은 메모리에서만 쓰고 저장·출력하지 않는다.
- 작업 제목·프로젝트 경로·Wi-Fi 이름은 메모리에서만 다룬다. 상태 기록(§7)에 넣지 않는다.
- 외부 서버 없음. 네트워크는 공식 클라이언트가 하고, 앱이 직접 여는 연결은 **localhost(Antigravity)뿐**이다.
- 클라이언트 자식 프로세스 환경: `DISABLE_TELEMETRY=1` · `DISABLE_ERROR_REPORTING=1` · `DISABLE_AUTOUPDATER=1`을 넣는다.
  **`CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC`은 넣지 않는다** — 사용량 조회까지 막는다(1.0.3 결함, Claude Code 2.1.251~2.1.269 재현).

## 4. 서비스 계약

### 4.1 Codex

| 항목 | 계약 |
|---|---|
| 원천 | `codex app-server`(stdio JSON-RPC). 메서드 `account/rateLimits/read` |
| 실행 파일 탐색 순서 | macOS: `Codex.app` · `ChatGPT.app` · `ChatGPT 2.app` 번들 안 `Contents/Resources/codex` → `/opt/homebrew/bin/codex` → `/usr/local/bin/codex` → `PATH`. Windows ⚠: Codex 앱 설치 폴더 → npm 전역(`codex.cmd`) → `PATH` |
| 응답 | `rateLimitsByLimitId` 아래 `limit_id`별 항목. 각 항목은 `primary`/`secondary` 창, 창마다 `used_percent` · `window_minutes` · `resets_at`(epoch 초), 항목에 `rate_limit_reached_type` |
| 대표 창 | `window_minutes == 10080`(주간). 없으면 `—%` |
| 추가 창 | `limit_id`가 `codex`가 아닌 항목(모델·기능별)은 이름 붙여 행으로 |
| 로그인 안내 | 공식 앱 또는 CLI 로그인 |

### 4.2 Claude

| 항목 | 계약 |
|---|---|
| 원천 | 설치된 **공식 CLI**의 실험적 내부 인터페이스 `get_usage` ⚠(정확한 호출 인자는 소스 확인 — 바이너리에 `--print` · `--output-format`가 있다). 문서화된 공개 인터페이스가 아니라 CLI 업데이트로 깨질 수 있다는 고지를 양쪽 모두 유지 |
| 실행 파일 탐색 순서 | macOS: `~/.local/bin/claude` → `/opt/homebrew/bin/claude` → `/usr/local/bin/claude` → `PATH`. Windows ⚠: `%USERPROFILE%\.local\bin\claude.exe` → npm 전역 → `PATH` |
| 창 | 전체 모델 주간(`seven_day`), 5시간(`five_hour`), 모델별 주간(`seven_day_<model>`, 예: `seven_day_opus`). 계정에 없는 창은 행을 만들지 않는다 |
| 대표 창 | `seven_day`(전체 모델 주간) |
| 계정·조직 둘 이상 | `accountSelection` |
| 로그인 안내 | 「Claude Code 로그인」 버튼이 **터미널에서 공식 로그인 명령**을 연다. 데스크톱 앱 로그인과 별개라고 적는다. macOS는 로컬 캐시 폴더에 여는 파일이 생길 수 있고, Windows는 같은 용도로 `.cmd` 하나 — 둘 다 토큰을 담지 않는다 |
| 작업 폴더 | CLI 호출 때 임시 작업 폴더를 **앱 전용 캐시 아래**에 둔다(사용자 프로젝트 폴더에서 돌리지 않는다) |

### 4.3 Antigravity

| 항목 | 계약 |
|---|---|
| 전제 | 로그인된 **공식 앱이 실행 중**이어야 한다. 아니면 `notRunning` |
| 탐지 | 실행 중인 `language_server` 프로세스를 찾는다. macOS 경로 `…/Antigravity.app/Contents/Resources/bin/language_server`(`/Applications` 또는 `/Volumes/Antigravity`). Windows ⚠: Antigravity 설치 폴더 아래 `language_server*.exe` |
| 포트·토큰 | 프로세스 명령줄의 `--csrf_token <값>` 또는 `--csrf_token=<값>`을 읽는다. 포트는 그 프로세스가 듣는 localhost TCP 포트 ⚠(선택 규칙은 소스 확인). macOS는 프로세스 목록·`lsof` 계열, Windows는 `Win32_Process.CommandLine` + `GetExtendedTcpTable` |
| 호출 | `POST http://127.0.0.1:<port>/exa.language_server_pb.LanguageServerService/RetrieveUserQuotaSummary`, 헤더 `X-Codeium-Csrf-Token: <값>` ⚠(본문 인코딩 — Connect/gRPC-web/JSON — 소스 확인) |
| 창 | 그룹별(`Gemini`, `Claude·GPT` 등) **주간**과 **5시간**. 구형 서버는 모델별 창으로 대체. 꺼진·알 수 없는 창은 숨긴다 |
| 대표 창 | Gemini 그룹 주간 ⚠. 캐시에는 `quotaLabel`로 **어느 창인지 이름을 같이** 쓴다(예: `Gemini 5시간`) |
| 서버 둘 이상 | `accountSelection`. 임의로 합치지 않는다 |
| 신뢰 | localhost만. 다른 호스트로 절대 가지 않는다 |

## 5. 작업 목록 (Codex · Claude)

- 탭: `고정 / 프로젝트 / 최근`. 서비스별로 마지막 탭과 선택 프로젝트를 기억한다.
- Codex: 공식 앱 로컬 SQLite를 **읽기 전용**으로 연다(`sqlite3_open_v2` 읽기 전용 + busy timeout). 고정은 `archived = 0 AND (is_pinned = 1 OR thread_section_id IN (Pinned 섹션))`,
  프로젝트는 `projects ORDER BY position, created_at_ms`. 내부·보관 항목은 숨긴다. 열기: ChatGPT 고정 대화는 웹, 그 외는 공식 앱.
  Windows ⚠: 같은 DB가 `%APPDATA%` 또는 `%LOCALAPPDATA%` 아래 어디에 있는지, 스키마가 같은지 실측.
- Claude: 데스크톱 **Code / Cowork 로컬 작업**(웹 Projects가 아니다). 폴더별 그룹, 계정 격리, 중복 제거, 최근은 정렬·개수 제한.
  열기: 데스크톱 앱 딥링크. Windows ⚠: 데스크톱 앱 로컬 메타데이터 경로·형식 실측.
- 제목·경로는 화면 표시용 메모리에만 둔다(§3.5).

## 6. 시스템 화면

| 수치 | 정의 | macOS 원천 | Windows 원천 |
|---|---|---|---|
| CPU 사용률 | 두 샘플 차분. 첫 2초는 「측정 중」 | `host_statistics` | `GetSystemTimes` 차분 |
| CPU 그래프 | 최근 샘플 고정 개수(메모리만) | 〃 | 〃 |
| 메모리 사용량 | 사용/전체 | `host_statistics64` | `GlobalMemoryStatusEx` |
| 메모리 압력 | macOS 3단계(정상·주의·위험) | `sysctl kern.memorystatus_vm_pressure_level` | 같은 개념 없음 → 가용 메모리 비율로 3단계 재정의(예: `> 25%` 정상, `10~25%` 주의, `< 10%` 위험) ⚠ 임계값은 PoC에서 정함 |
| 저장 공간 | 시작 볼륨 가용/전체, 30초마다 | `volumeAvailableCapacityForImportantUsage` | `GetDiskFreeSpaceEx`(시스템 드라이브) |
| 배터리 | 잔량·충전 중·최대 용량(완전 충전/설계, 추정) | IOKit `IOPSCopyPowerSourcesInfo` · `AppleSmartBattery` | `GetSystemPowerStatus` + WMI `root\wmi` `BatteryFullChargedCapacity`/`BatteryStaticData`. 배터리 없으면 행 자체를 숨긴다 |
| 네트워크 속도 | 기본 연결 인터페이스의 송·수신 바이트 차분 | `SCDynamicStore` 기본 경로 + 인터페이스 카운터 | `GetBestInterface`/`GetBestRoute2` + `GetIfEntry2` 카운터 |
| 연결 종류·이름 | Wi-Fi SSID / Ethernet·VPN 서비스 이름 | `SCNetworkInterface` 종류, `CoreWLAN` SSID | `GetIfEntry2.Type`, `WlanQueryInterface` SSID |
| 이름 표시 권한 | SSID는 **사용자가 버튼을 눌렀을 때만** 위치 권한을 요청. 좌표는 수집·저장하지 않는다 | `CLLocationManager` | Windows 11 24H2+도 SSID에 위치 권한이 필요 — 같은 버튼·같은 문구 |
| 제공 안 됨 | 센서·API가 값을 안 주면 「제공 안 됨」. 0으로 채우지 않는다 | | |

## 7. 상태 기록 `status.json` (선택 기능)

배포 빌드는 **자동으로 쓰지 않는다**(1.3.1). 개발·진단 옵션을 명시했을 때만 쓴다. 쓸 때의 형식은 아래로 고정한다 — Project Monitor `ai_usage.py`가 이 형식을 읽는다.

| 플랫폼 | 경로 |
|---|---|
| macOS | `~/Library/Caches/<bundle-id>/status.json` |
| Windows | `%LOCALAPPDATA%\AIWeeklyBar\status.json` |

```json
{
  "schemaVersion": 1,
  "writtenAt": 1760428800,
  "services": {
    "codex":       { "percent": 63, "lastSuccessfulRefresh": 1760428790, "resetAt": 1760900000, "error": null, "quotaLabel": null },
    "claude":      { "percent": 42, "lastSuccessfulRefresh": 1760428700, "resetAt": 1760700000, "error": null, "quotaLabel": null },
    "antigravity": { "percent": 80, "lastSuccessfulRefresh": 1760428600, "resetAt": 1760440000, "error": null, "quotaLabel": "Gemini 5시간" }
  }
}
```

- `percent`: 대표 창의 **남은 %**. 정수 또는 `"42%"` 문자열 모두 허용(소비자가 둘 다 읽는다). 값이 없으면 `null`.
- `lastSuccessfulRefresh` · `resetAt`: epoch 초. 없으면 `null`.
- `error`: §3.2 범주 문자열 또는 `null`. 원시 메시지를 넣지 않는다.
- `quotaLabel`: Antigravity만. 대표 창의 이름. 다른 서비스는 `null`.
- **원자적으로 쓴다**: 임시 파일에 쓰고 rename. 소비자가 쓰는 도중에 읽으면 마지막 값을 쓰도록 돼 있지만 그것에 기대지 않는다.
- 작업 제목·경로·SSID·CSRF·토큰은 **절대** 넣지 않는다.
- `schemaVersion` · `writtenAt`은 v0.1에서 추가 제안 — 현재 소비자는 없어도 읽는다 ⚠.

## 8. 플랫폼 매핑

| 항목 | macOS | Windows |
|---|---|---|
| 상주 위치 | 메뉴바(상단) `NSStatusItem` + `NSPopover` | 알림 영역(하단) 아이콘 + 작업표시줄 위 플라이아웃 창 |
| 아이콘 텍스트 | 문자열 그대로 | 숫자를 비트맵으로 렌더링(16·24·32px, DPI별) |
| 숨김 | 항상 보임 | 기본은 `^` 넘침 영역 — 설치 안내에 「고정」 절차를 적는다 |
| 설정 저장 | `UserDefaults` | `%APPDATA%\AIWeeklyBar\settings.json` |
| 자동 실행 | 로그인 항목(사용자가 추가) | 시작프로그램 바로가기(사용자가 추가). 레지스트리 Run 키를 앱이 스스로 쓰지 않는다 |
| 종료 | 팝업 전원 버튼 | 동일 + 트레이 우클릭 「종료」 |
| 서명 | Developer ID 없음 → Gatekeeper 경고 안내 | Authenticode 없음 → SmartScreen 경고 안내 |
| 배포 | ZIP + `SHA256.txt` + 설치 안내 | 동일 형식. 설치 프로그램 없이 ZIP |
| 최소 OS | macOS 13 | Windows 10 1809 이상 ⚠(Claude Code 요구와 맞춤), SSID 권한 동작은 11 24H2에서 검증 |
| 임시·캐시 | `~/Library/Caches/<bundle-id>/` | `%LOCALAPPDATA%\AIWeeklyBar\` |

## 9. 공유 테스트 벡터

두 구현이 같은 입력에 같은 출력을 내야 하는 순수 함수. 소스 저장소의 `docs/contract/fixtures/*.json`으로 만들고 양쪽 테스트가 읽는다.
1.3.1 자체 검사가 이미 다루는 영역(바이너리 PASS 문자열에서 확인)을 그대로 벡터 이름으로 쓴다.

| 벡터 | 입력 → 기대 |
|---|---|
| `countdown` | 초 → §3.4 문자열. 경계: `0`, `59`, `60`, `3599`, `3600`, `86399`, `86400`, `90000` |
| `remaining` | `used_percent` → `floor(clamp(100−used, 0, 100))`. 경계: `-5`, `0`, `37.6`, `100`, `120`, `null` |
| `weekly-selection` | 창 목록 → 대표 창 선택(Codex `window_minutes == 10080`, Claude `seven_day`) |
| `missing-values` | 창 없음 · 값 없음 → `—%`, 행 생략 |
| `freshness` | `(now, lastSuccessfulRefresh, resetAt)` → stale 여부(§3.3 세 조건) |
| `expired-quota` | reset 지난 값 → stale, 「갱신 중」 |
| `backoff` | 연속 실패 n회 → 다음 간격(1·2·4·8·16·30·30…) |
| `auth-pause` | `auth` 오류 → 자동 조회 중단, 수동 새로고침 성공 → 재개 |
| `antigravity-groups` | 응답 → 주간·5시간 그룹 창, 꺼진·알 수 없는 창 제외, 구형 서버 모델별 대체 |
| `csrf-parse` | 명령줄 문자열 → 토큰(`--csrf_token x` · `--csrf_token=x` · 없음) |
| `pinned-order` | 고정 목록 → 정렬·중복 제거, 잘못된 ID 제외 |
| `recent-limit` | 최근 목록 → 정렬·개수 제한, 내부·보관 제외 |
| `status-json` | 내부 상태 → §7 JSON(민감 값 0건) |

## 10. 실측으로 닫아야 하는 항목 (⚠ 모음)

1. Claude `get_usage` 정확한 호출 인자와 Windows 네이티브 CLI에서 같은 응답인지.
2. Codex `app-server`의 Windows 동작(stdio·경로), 앱 설치 폴더.
3. Antigravity Windows: `language_server` 실행 파일 이름·명령줄 형식·포트 선택 규칙·요청 본문 인코딩.
4. Codex·Claude 작업 메타데이터의 Windows 경로와 스키마 동일 여부.
5. 1.3.1에서 백오프 규칙(1.0.2)이 유지되는지.
6. 메모리 압력 3단계의 Windows 임계값.
7. `status.json`에 `schemaVersion` · `writtenAt`을 넣을지(소비자 수정 없이 가능).

## 11. 변경 규칙

- 이 문서를 바꾸는 커밋은 **두 구현 중 적어도 한쪽의 변경과 같이** 간다. 문서만 앞서 가지 않는다.
- 플랫폼이 하나만 지원하는 수치·기능은 §6·§8 표에 「제공 안 됨」으로 적고 다른 쪽에서 흉내 내지 않는다.
- 비공식 인터페이스(Claude `get_usage`, Antigravity 로컬 서버)가 바뀌면 §4와 공개 고지(README·INSTALL)를 같은 커밋에서 고친다.
