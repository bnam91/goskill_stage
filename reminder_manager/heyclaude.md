# heyclaude.md — reminder_manager 환경 세팅

> 이 파일은 Claude Code가 읽고 따라하는 안내문이야. 사람이 직접 보지 않아도 OK.

## 트리거 조건

사용자가 "reminder_manager 세팅", "미리알림 스킬 처음 사용", "미리알림 환경 세팅해줘" 같은 말을 하거나, 아래 사전 요구사항 중 하나라도 빠져있으면 이 단계를 진행해. 모든 단계는 **idempotent** — 이미 있으면 skip.

## 사전 요구사항 체크

```bash
echo "=== reminder_manager 환경 점검 ==="
[[ "$(uname)" == "Darwin" ]]                                                          && echo "✓ macOS"             || echo "❌ macOS 전용 스킬 (현재 OS: $(uname))"
[ -f "$HOME/.claude/skills/reminder_manager/scripts/app_reminders_control.py" ]       && echo "✓ app_reminders_control.py" || echo "❌ 스크립트 없음 (goskill 다시 다운로드)"
python3 -c "from EventKit import EKEventStore" 2>/dev/null                            && echo "✓ pyobjc-framework-EventKit" || echo "❌ pyobjc-framework-EventKit 없음"
```

빠진 게 있으면 아래 단계로 보충.

---

## Step 1. macOS 확인

이 스킬은 **macOS 전용**. EventKit / AppleScript 기반.

```bash
if [[ "$(uname)" != "Darwin" ]]; then
  echo "❌ reminder_manager는 macOS 전용 스킬입니다. 현재 OS: $(uname)"
  exit 1
fi
```

---

## Step 2. PyObjC EventKit 설치

```bash
if python3 -c "from EventKit import EKEventStore" 2>/dev/null; then
  echo "✓ pyobjc-framework-EventKit 이미 설치됨"
else
  pip install pyobjc-framework-EventKit
  echo "✅ pyobjc-framework-EventKit 설치 완료"
fi
```

`pip` 명령이 없거나 권한 오류가 나면:
- `pip3 install pyobjc-framework-EventKit` 시도
- 또는 `python3 -m pip install pyobjc-framework-EventKit`

---

## Step 3. 스크립트 확인

스킬과 같이 배포되는 파일들 (`~/.claude/skills/reminder_manager/scripts/`).

```bash
SCRIPTS="$HOME/.claude/skills/reminder_manager/scripts"
for f in app_reminders_control.py complete_reminder.py set_priority.py; do
  if [ -f "$SCRIPTS/$f" ]; then
    echo "✓ $f"
  else
    echo "⚠️ $f 누락"
  fi
done
```

`app_reminders_control.py`만 있으면 핵심 동작은 가능. `complete_reminder.py` / `set_priority.py`는 부속 기능.

---

## Step 4. 첫 실행 + 미리알림 앱 접근 권한 허용

처음 실행하면 macOS가 **"<터미널 앱>이(가) 미리알림에 접근하려고 합니다"** 권한 팝업을 띄움. **"확인"을 눌러야 동작**.

```bash
python3 -c "
import sys, os; sys.path.insert(0, os.path.expanduser('~/.claude/skills/reminder_manager/scripts'))
from app_reminders_control import get_list_names
names = get_list_names()
for i, n in enumerate(names, 1): print(f'{i}. {n}')
"
```

목록 이름이 출력되면 통과.

권한 거부했거나 팝업이 안 뜨면:
- `시스템 설정 → 개인정보 보호 및 보안 → 미리 알림` 에서 사용 중인 터미널/Claude Code 권한 체크
- 또는 `시스템 설정 → 개인정보 보호 및 보안 → 전체 디스크 접근` 권한 부여 (Reminders SQLite DB 접근에 필요할 수 있음)

---

## 검증

위 Step 4 명령이 미리알림 목록을 출력하면 세팅 완료. 이후 사용자가 "미리알림 목록 보여줘", "XX 목록에 YY 추가해줘" 같은 말을 하면 정상 동작.
