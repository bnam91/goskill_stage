---
name: reminder_manager
description: 맥 미리알림(Reminders) 앱을 제어하는 스킬이야.
---

맥 미리알림(Reminders) 앱을 제어하는 스킬이야.
스크립트 경로: ~/.claude/skills/reminder_manager/scripts/app_reminders_control.py

> **🚀 처음 사용하는 환경이라면**: 같은 폴더의 `heyclaude.md`를 따라 환경 세팅을 먼저 진행. macOS 전용이며, `pyobjc-framework-EventKit` 패키지 + 미리알림 앱 접근 권한이 필요해. (`scripts/` 안의 파이썬 파일들은 이 스킬 폴더에 같이 들어있어서 별도 설치 불필요.)

## 사용 가능한 기능

### 1. 목록 이름만 빠르게 확인
스크립트에 `--lists` 옵션은 없으므로 아래 Python 한 줄로 대체:

```bash
python3 -c "
import sys, os; sys.path.insert(0, os.path.expanduser('~/.claude/skills/reminder_manager/scripts'))
from app_reminders_control import get_list_names
names = get_list_names()
for i, n in enumerate(names, 1): print(f'{i}. {n}')
"
```

### 2. 전체 목록 확인 (번호 + 이름)
스크립트를 실행하면 목록이 표시됨. 번호 없이 실행 시 대화형 모드이므로, 반드시 번호나 이름으로 직접 지정해야 함.

### 3. 특정 목록의 미리알림 조회
```bash
python3 ~/.claude/skills/reminder_manager/scripts/app_reminders_control.py [번호 또는 목록이름]
```
예시:
- `python3 ... 1` → 1번 목록 조회
- `python3 ... "군자"` → "군자" 포함 목록 조회

### 4. 미리알림 추가 (섹션 지정)
```bash
python3 ~/.claude/skills/reminder_manager/scripts/app_reminders_control.py [목록번호] "섹션이름, 항목제목"
```
예시:
- `python3 ... 4 "1주차, 명함만들기"` → 4번 목록의 "1주차" 섹션에 "명함만들기" 추가

### 5. 깃발 표시 항목 조회 ⚠️ 중요

**EventKit의 `priority == 1`은 깃발이 아니라 우선순위(높음)임. 깃발은 반드시 AppleScript로 조회해야 함.**

```bash
osascript -e '
tell application "Reminders"
    set remindersText to ""
    set foundCount to 0
    repeat with aList in lists
        repeat with aReminder in (reminders of aList)
            try
                if flagged of aReminder is true and completed of aReminder is false then
                    set remindersText to remindersText & (name of aReminder) & "|" & (name of aList) & "\n"
                    set foundCount to foundCount + 1
                end if
            end try
        end repeat
    end repeat
    if foundCount is 0 then return "EMPTY"
    return remindersText
end tell
'
```

특정 목록만 조회하려면 `repeat with aList in lists` 대신 `set aList to list "inbox"` 사용.

### 6. 우선순위 변경
스크립트에 기능 없으므로 EventKit으로 직접 처리:
```python
import sys, os, time
sys.path.insert(0, os.path.expanduser('~/.claude/skills/reminder_manager/scripts'))
from app_reminders_control import _get_event_store, fetch_reminders_sync

event_store = _get_event_store()
calendars = event_store.calendarsForEntityType_(1)
cal = next((c for c in calendars if c.title() == '목록이름'), None)
predicate = event_store.predicateForRemindersInCalendars_([cal])
reminders = fetch_reminders_sync(event_store, predicate)

targets = {'항목1', '항목2'}  # 변경할 항목 제목들
# 우선순위: 1=높음, 5=중간, 9=낮음, 0=없음
for r in reminders:
    if r.title() in targets:
        r.setPriority_(1)
        event_store.saveReminder_commit_error_(r, True, None)
```

### 7. 미리알림 완료 처리
같은 폴더의 `complete_reminder.py` 사용:
```bash
python3 ~/.claude/skills/reminder_manager/scripts/complete_reminder.py "항목제목"
```

### 8. 우선순위 일괄 설정
같은 폴더의 `set_priority.py` 사용 (인자는 스크립트 내부 참고):
```bash
python3 ~/.claude/skills/reminder_manager/scripts/set_priority.py
```

## 실행 방법

사용자의 요청을 분석하여 적절한 명령을 Bash 도구로 실행해줘.

- "미리알림 목록 보여줘" → 목록 이름 조회 실행
- "XX 목록 확인해줘" → 해당 목록의 미리알림 전체 조회
- "XX에 YY 추가해줘" → 목록과 섹션 파악 후 추가 실행
- 목록 이름이 불확실하면 먼저 목록 이름 조회 후 매칭

결과는 한국어로 정리해서 보여줘.
