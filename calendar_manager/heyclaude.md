# heyclaude.md — calendar_manager 환경 세팅

> 이 파일은 Claude Code가 읽고 따라하는 안내문이야. 사람이 직접 보지 않아도 OK.

## 트리거 조건

사용자가 "calendar_manager 세팅", "캘린더 스킬 처음 사용", "캘린더 환경 세팅해줘" 같은 말을 하거나, 아래 사전 요구사항 중 하나라도 빠져있으면 이 단계를 진행해. 모든 단계는 **idempotent** — 이미 있으면 skip.

## 사전 요구사항 체크

```bash
echo "=== calendar_manager 환경 점검 ==="
[[ "$(uname)" == "Darwin" ]]                                                      && echo "✓ macOS"             || echo "❌ macOS 전용 스킬 (현재 OS: $(uname))"
[ -f "$HOME/.claude/skills/calendar_manager/scripts/app_calendar_control.py" ]    && echo "✓ app_calendar_control.py (스킬에 포함됨)" || echo "❌ 스크립트 없음 (goskill 다시 다운로드)"
command -v osascript >/dev/null 2>&1                                              && echo "✓ osascript"         || echo "❌ osascript 없음 (macOS 기본이라 정상이라면 있어야 함)"
python3 -c "import subprocess, argparse" 2>/dev/null                              && echo "✓ Python 표준 라이브러리" || echo "❌ Python3 없음"
```

빠진 게 있으면 아래 단계로 보충.

---

## Step 1. macOS 확인

이 스킬은 **macOS 전용**. AppleScript(`osascript`)로 캘린더 앱을 제어해서 Windows/Linux에서는 동작 안 함.

```bash
if [[ "$(uname)" != "Darwin" ]]; then
  echo "❌ calendar_manager는 macOS 전용 스킬입니다. 현재 OS: $(uname)"
  echo "→ 사용자에게 안내 후 중단"
  exit 1
fi
```

---

## Step 2. 스크립트 확인

스킬과 같이 배포되는 파일 (`~/.claude/skills/calendar_manager/scripts/app_calendar_control.py`). goskill로 다운로드 시 자동으로 함께 받음 — 별도 설치 단계 없음.

```bash
SCRIPT="$HOME/.claude/skills/calendar_manager/scripts/app_calendar_control.py"
if [ -f "$SCRIPT" ]; then
  echo "✓ app_calendar_control.py 정상"
else
  echo "❌ 스크립트 누락 — goskill 앱에서 calendar_manager 다시 다운로드"
  exit 1
fi
```

---

## Step 3. 첫 실행 + 캘린더 앱 접근 권한 허용

처음 실행하면 macOS가 **"<터미널 앱>이(가) 캘린더에 접근하려고 합니다"** 권한 팝업을 띄움. **"확인"을 눌러야 동작**.

```bash
python3 "$HOME/.claude/skills/calendar_manager/scripts/app_calendar_control.py" --list-cals
```

캘린더 목록이 출력되면 통과.

권한 거부했거나 팝업이 안 뜨면:
- `시스템 설정 → 개인정보 보호 및 보안 → 캘린더` 에서 사용 중인 터미널/Claude Code 권한 체크
- 또는 `시스템 설정 → 개인정보 보호 및 보안 → 자동화` 에서 "캘린더" 권한 부여

---

## 검증

위 Step 3에서 본인 캘린더 목록이 출력되면 세팅 완료. 이후 사용자가 "오늘 일정 보여줘", "이번주 일정", "XX 일정 추가해줘" 같은 말을 하면 정상 동작.

> 참고: `DEFAULT_CALS = []` 로 비워둔 상태라 `--query`는 사용 가능한 모든 캘린더를 자동으로 조회해. 특정 캘린더만 보고 싶으면 사용자에게 캘린더 이름 받아서 `--cals "이름1,이름2"` 옵션으로 지정.
