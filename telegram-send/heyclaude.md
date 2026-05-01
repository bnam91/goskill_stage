# heyclaude.md — telegram-send 환경 세팅

> 이 파일은 Claude Code가 읽고 따라하는 안내문이야. 사람이 직접 보지 않아도 OK.
> ⚠️ **이 스킬은 본인 멀티맥 동기화용**. 새 직원에게 그냥 주면 동작 안 함 — `module_telegram`이 git repo가 아니라 별도 배포 메커니즘이 없음.

## 트리거 조건

사용자가 "telegram-send 세팅", "텔레그램 스킬 처음 사용" 같은 말을 하거나, 아래 사전 요구사항 중 하나라도 빠져있으면 진행. 모든 단계 **idempotent**.

## 사전 요구사항 체크

```bash
echo "=== telegram-send 환경 점검 ==="
[ -f "$HOME/Documents/github_cloud/module_telegram/module/index.js" ] && echo "✓ module_telegram 모듈 있음" || echo "❌ module_telegram 누락 (Step 1)"
command -v node >/dev/null 2>&1                                       && echo "✓ Node.js ($(node --version 2>/dev/null))" || echo "❌ Node.js 없음 (Step 2)"
```

빠진 게 있으면 아래 단계로 보충.

---

## Step 1. module_telegram 모듈 확인

이 스킬은 `~/Documents/github_cloud/module_telegram/module/index.js`에 정의된 `sendToHyunbin()` 함수를 호출함. 모듈 파일에 봇 토큰과 chat_id가 박혀있어서 **별도 배포는 안 함**.

```bash
MODULE="$HOME/Documents/github_cloud/module_telegram/module/index.js"
if [ -f "$MODULE" ]; then
  echo "✓ module_telegram 정상"
else
  echo "❌ module_telegram 없음. 사용자에게 안내:"
  echo ""
  echo "  본인이 사용 중인 다른 맥에서 다음 폴더를 통째로 복사해 오세요:"
  echo "    ~/Documents/github_cloud/module_telegram/"
  echo ""
  echo "  (이 폴더에는 봇 토큰이 박혀있어서 git/공개 레포에는 두지 않음)"
  echo ""
  echo "→ 사용자가 복사 완료했다고 알려주면 다음 단계 진행"
  exit 1
fi
```

---

## Step 2. Node.js 확인

```bash
if command -v node >/dev/null 2>&1; then
  echo "✓ Node.js $(node --version)"
else
  echo "❌ Node.js 없음. brew install node 또는 https://nodejs.org/"
fi
```

---

## 검증

```bash
node -e "
const { sendToHyunbin } = require(require('os').homedir() + '/Documents/github_cloud/module_telegram/module/index.js');
sendToHyunbin('telegram-send 세팅 완료 테스트').then(r => console.log('전송 완료:', r.message_id)).catch(e => console.error('실패:', e.message));
"
```

텔레그램에 테스트 메시지가 도착하면 세팅 완료.
