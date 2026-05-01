# heyclaude.md — ads-manager-app 환경 세팅

> 이 파일은 Claude Code가 읽고 따라하는 안내문이야. 사람이 직접 보지 않아도 OK.

## 트리거 조건

사용자가 "ads-manager-app 세팅", "광고 계산기 처음 사용" 같은 말을 하거나, 아래 사전 요구사항 중 하나라도 빠져있으면 진행. 모든 단계 **idempotent**.

## 사전 요구사항 체크

```bash
echo "=== ads-manager-app 환경 점검 ==="
[[ "$(uname)" == "Darwin" ]]                                                   && echo "✓ macOS"             || echo "⚠️ macOS 외 OS는 'open' 명령 다를 수 있음 (Linux: xdg-open / Windows: start)"
[ -f "$HOME/.claude/skills/ads-manager-app/scripts/ad-calculator.html" ]       && echo "✓ ad-calculator.html (스킬에 포함됨)" || echo "❌ HTML 누락 (goskill 다시 다운로드)"
```

## Step 1. HTML 파일 확인

```bash
HTML="$HOME/.claude/skills/ads-manager-app/scripts/ad-calculator.html"
if [ -f "$HTML" ]; then
  echo "✓ ad-calculator.html 정상"
else
  echo "❌ HTML 누락 — goskill 앱에서 ads-manager-app 다시 다운로드"
  exit 1
fi
```

## 검증

```bash
open "$HOME/.claude/skills/ads-manager-app/scripts/ad-calculator.html"
```

브라우저에 광고 계산기가 뜨면 완료.
