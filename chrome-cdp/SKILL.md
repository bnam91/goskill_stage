---
name: chrome-cdp
description: Chrome CDP(Remote Debugging) 포트를 열고 MCP chrome-devtools로 브라우저를 제어하는 스킬이야. 사용자가 "CDP 열어줘", "크롬 디버깅 포트 열어줘", "CDP 연결해줘", "chrome-devtools 써줘" 등의 요청을 할 때 실행해.
---

# Chrome CDP 디버깅 포트 관리

MCP `chrome-devtools` 툴을 사용하기 위해 Chrome을 CDP 모드로 실행하는 스킬.

## 기본 설정

- **기본 포트**: `9225`
- **user-data-dir 기본값**: `/tmp/chrome-debug` (격리된 임시 프로필)

## 프로필 별칭 (인수)

`/chrome-cdp` 뒤에 `-별칭`을 붙이면 해당 계정 세션으로 CDP를 실행한다.

본인 환경에 맞게 아래 표를 채워서 사용:

| 포트 | 인수 | 계정 | user-data-dir | 비고 |
|------|------|------|---------------|------|
| 9225 | (없음) | 격리/임시 | `/tmp/chrome-debug` | 로그인 없는 클린 상태 |
| 9225 | `-myaccount` | 본인 계정 | `~/chrome_profiles/myaccount` | 예시 (본인 환경에 맞게 수정) |

- 기본 포트는 `9225` (격리된 클린 작업용). 여러 프로필을 동시에 열거나 본인 일반 Chrome과 분리하고 싶으면 다른 포트(9290, 9291, 9341 등) 사용
- **참고**: 포트 `9222`는 본인의 일반 Chrome(로그인된 프로필) 용도로 비워두는 컨벤션. CDP 기본은 `9225`로 분리
- user_data 권장 경로: `~/chrome_profiles/` 또는 `~/Documents/github_cloud/user_data/`

> **주의**: Chrome 쿠키는 macOS Keychain 키로 암호화되어 있어 프로필 복사 방식은 Google 세션이 작동하지 않음. 각 프로필 디렉토리에서 직접 로그인한 세션만 사용 가능.

### 프로필 선택 로직

```bash
# 인수 파싱 예시: /chrome-cdp -myaccount
ALIAS="myaccount"  # 인수에서 추출

case "$ALIAS" in
  myaccount)
    USER_DATA_DIR="$HOME/chrome_profiles/myaccount"
    ;;
  # 별칭 추가 시 여기에 case 추가
  *)
    USER_DATA_DIR="/tmp/chrome-debug"
    ;;
esac
```

## ⚠️ 사전 체크: 아키텍처 일치 (Apple Silicon 필수)

**Apple Silicon Mac(M1/M2/M3/M4)에서 Intel용 Chrome이 깔려 있으면 Rosetta 번역으로 매우 느려진다.** 반드시 네이티브 ARM64 빌드 사용.

```bash
SYS_ARCH=$(uname -m)              # arm64 (Apple Silicon) or x86_64 (Intel)
CHROME_PATH="/Applications/Google Chrome.app/Contents/MacOS/Google Chrome"
CHROME_ARCH=$(file "$CHROME_PATH" 2>/dev/null | grep -oE "arm64|x86_64" | head -1)

if [ "$SYS_ARCH" = "arm64" ] && [ "$CHROME_ARCH" = "x86_64" ]; then
  echo "⚠️  Apple Silicon Mac에 Intel용 Chrome이 설치됨 → Rosetta로 동작해서 매우 느려짐"
  echo "    https://www.google.com/chrome/ 에서 'Mac용 (Apple silicon)' 다시 다운로드 권장"
fi
```

이후 Chrome 실행 시 `arch -arm64`를 prefix로 붙여서 강제로 네이티브 모드로 실행한다.

## 실행 순서

### 1단계: CDP 상태 및 프로필 확인

포트가 열려 있어도 **현재 실행 중인 프로필이 요청한 것과 다르면 재시작**해야 한다.

**먼저 non-CDP Chrome 확인** — 같은 프로필로 일반 Chrome이 실행 중이면 프로필 잠금 충돌이 발생한다. kill 전에 반드시 확인하고 사용자에게 알려야 한다.

```bash
TARGET_DIR=$(eval echo "$USER_DATA_DIR")
ME=$(whoami)

# 1-A. non-CDP Chrome이 같은 프로필로 실행 중인지 확인
NON_CDP=$(ps aux | grep "Google Chrome" | grep -v grep | grep -v "remote-debugging-port" | grep "$TARGET_DIR" | head -1)
if [ -n "$NON_CDP" ]; then
  echo "⚠️ Chrome이 non-CDP로 실행 중 ($TARGET_DIR, port 없음)"
  echo "→ kill 후 Preferences 초기화 → CDP 재시작합니다."
  pkill -u "$ME" -f "Google Chrome" 2>/dev/null
  sleep 1
fi

# 1-B. CDP Chrome 상태 확인
CURRENT_PROFILE=$(ps aux | grep "remote-debugging-port=9225" | grep -v grep | grep -o "\-\-user-data-dir=[^ ]*" | cut -d= -f2)

if curl -s http://localhost:9225/json/version >/dev/null 2>&1; then
  if [ "$CURRENT_PROFILE" = "$TARGET_DIR" ]; then
    echo "✅ 이미 올바른 프로필로 CDP 실행 중 → 3단계로"
    # SKIP_RESTART=true
  else
    echo "⚠️ 다른 프로필로 실행 중 (현재: $CURRENT_PROFILE)"
    echo "→ $TARGET_DIR 로 재시작"
    # SKIP_RESTART=false → 2단계 실행
  fi
else
  echo "CDP 미실행 → 2단계 실행"
fi
```

- non-CDP Chrome 실행 중 → **사용자에게 알리고** kill 후 2단계
- 동일 프로필 CDP 실행 중 → **3단계로 바로 이동**
- 다른 프로필 or 미실행 → **2단계 실행**

### 2단계: Chrome CDP 모드로 실행

> **⚠️ kill은 충돌 시에만** — 1단계에서 이미 non-CDP 충돌이나 다른 프로필인 경우만 여기 도달.
> MCP가 연결 중인 상태에서 kill하면 WebSocket이 끊겨 MCP 서버가 종료됨 → CC 재시작 필요.
> 따라서 **1단계에서 "동일 프로필 CDP 실행 중"이면 절대 kill하지 않고 3단계로 바로 이동.**

```bash
ME=$(whoami)

# kill이 필요한 경우만 실행 (non-CDP 충돌 or 다른 프로필)
# SIGTERM 먼저 시도
pkill -u "$ME" -f "remote-debugging-port=9225" 2>/dev/null
sleep 1
# 포트 여전히 점유 중이면 kill -9 (이 경우 MCP 서버도 죽으므로 CC 재시작 필요)
if lsof -ti :9225 >/dev/null 2>&1; then
  lsof -ti :9225 | xargs kill -9 2>/dev/null
  sleep 1
  echo "⚠️ kill -9 사용됨 → CDP 작업 후 CC 재시작 필요"
fi

# 크래시 복원 팝업 방지: 프로필 종료 상태를 "정상 종료"로 초기화
PREF_FILE="${USER_DATA_DIR:-/tmp/chrome-debug}/Default/Preferences"
if [ -f "$PREF_FILE" ]; then
  python3 -c "
import json
with open('$PREF_FILE', 'r') as f:
    prefs = json.load(f)
prefs.setdefault('profile', {})['exit_type'] = 'Normal'
prefs.setdefault('profile', {})['exited_cleanly'] = True
with open('$PREF_FILE', 'w') as f:
    json.dump(prefs, f)
" 2>/dev/null
fi

# CDP 모드로 재시작 (arch -arm64 prefix로 Apple Silicon 네이티브 강제)
arch -arm64 "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
  --remote-debugging-port=9225 \
  --remote-allow-origins='*' \
  --user-data-dir="${USER_DATA_DIR:-/tmp/chrome-debug}" \
  --no-first-run \
  --no-default-browser-check \
  --hide-crash-restore-bubble \
  --disable-blink-features=AutomationControlled \
  > /tmp/chrome_cdp.log 2>&1 &

# 포트 열릴 때까지 대기 (최대 10초)
python3 - << 'EOF'
import time, urllib.request, json

for i in range(10):
    try:
        with urllib.request.urlopen('http://localhost:9225/json/version', timeout=2) as r:
            data = json.loads(r.read())
            print(f"✅ CDP 준비 완료: {data.get('webSocketDebuggerUrl')}")
            break
    except:
        print(f"  대기 중... ({i+1}/10)")
        time.sleep(1)
else:
    print("❌ CDP 포트 열기 실패. /tmp/chrome_cdp.log 확인")
EOF
```

> **Intel Mac에서 사용 시**: `arch -arm64`를 빼고 그냥 `"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome"` 로 실행.

### 3단계: MCP chrome-devtools로 제어

포트가 열리면 MCP 툴 사용:

```python
# 열린 탭 목록 확인
mcp__chrome-devtools__list_pages()

# URL 이동
mcp__chrome-devtools__navigate_page(type="url", url="https://www.naver.com")

# 스크린샷
mcp__chrome-devtools__take_screenshot()

# JS 실행
mcp__chrome-devtools__evaluate_script(script="document.title")
```

## 주의사항 (프로필 관리)

- 각 프로필은 해당 디렉토리에서 직접 로그인한 세션을 그대로 사용
- 세션 만료 시 CDP Chrome 창에서 직접 재로그인 필요
- 단, Google 계정은 CDP 모드에서 신규 OAuth 로그인 차단됨 → 기존 세션 유지 중에만 사용 가능

## MCP vs WebSocket

MCP `chrome-devtools`는 기본적으로 자체 Chrome을 띄움. 이 스킬이 띄운 9225 포트 Chrome에 연결하려면 두 가지 방법:

**방법 1**: 명시적 포트 변종 사용
- `mcp__chrome-devtools-9225__*` 툴 호출 (포트 9225 전용 MCP 변종이 등록되어 있으면)

**방법 2**: 기본 MCP를 9225에 연결
- `~/.claude.json`의 chrome-devtools 설정에 `--browserUrl http://127.0.0.1:9225` 인수 필요
- 설정 변경 후 **Claude Code 재시작** 필요

재시작 없이 즉시 사용하려면 Python WebSocket으로 직접 제어:
```python
import websocket, json, urllib.request

with urllib.request.urlopen('http://localhost:9225/json/version') as r:
    ws_url = json.loads(r.read())['webSocketDebuggerUrl']

ws = websocket.create_connection(ws_url)
# Target.createTarget으로 새 탭 열기
ws.send(json.dumps({"id":1,"method":"Target.createTarget","params":{"url":"https://example.com"}}))
```

## 캡챠 처리

페이지 이동 후 캡챠가 감지되면 **사용자에게 묻지 말고 직접 해결 시도**:

1. `mcp__chrome-devtools__take_screenshot()` 으로 캡챠 화면 캡처
2. 이미지에서 질문 텍스트 읽기 (예: "가게 전화번호의 뒤에서 1번째 숫자는?")
3. 이미지 내 정답 확인 후 `mcp__chrome-devtools__fill()` 또는 `mcp__chrome-devtools__click()` 으로 입력
4. 확인 버튼 클릭 후 `mcp__chrome-devtools__take_screenshot()` 으로 결과 확인
5. 캡챠 통과 실패 시에만 사용자에게 안내

## 주의사항

- `pkill`은 **본인 유저** (`whoami` 결과)만 대상. 다른 유저의 Chrome은 건드리지 않음
- `/tmp/chrome-debug`는 Google 로그인 없는 클린 상태
- Chrome 재시작 시 기존 탭 세션 초기화됨

## ⚠️ 일렉트론 앱 포트 보호

**본인이 개발 중인 일렉트론 앱이 사용하는 포트는 절대 건드리지 않는다.**

예시 (사용 중인 앱이 있다면 본인 환경에 맞게 추가):
- 포트 9333, 9334 → 본인의 web-editor 앱 (있다면)
- 그 외 9300번대 포트들 → 다른 일렉트론 앱

규칙:
- `pkill` 등 프로세스 종료 시 이 스킬의 기본 포트 `remote-debugging-port=9225` 대상만 종료
- 일렉트론 앱 포트 프로세스는 어떤 경우에도 kill 금지
- chrome-devtools MCP가 일렉트론 앱 탭을 반환해도 navigate/click 등 조작 금지
- 본인 일반 Chrome(9222 등)은 별도 포트로 분리되어 있어 이 스킬이 건드리지 않음
