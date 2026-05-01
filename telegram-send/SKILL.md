---
name: telegram-send
description: goagent_bot으로 현빈에게 텔레그램 메시지를 보내는 스킬이야. 사용자가 "텔레그램 보내줘", "현빈한테 텔레그램 보내줘", "텔레그램 메시지 전송해줘", "현빈한테 알려줘" 등의 요청을 할 때 실행해.
version: 1.0.0
---

# 텔레그램 메시지 전송 (goagent_bot → 현빈)

goagent_bot(`gogo_agent_bot`)을 통해 현빈에게 텔레그램 메시지를 보내는 스킬.

> **🚀 처음 사용하는 환경이라면**: 같은 폴더의 `heyclaude.md` 참고. `~/Documents/github_cloud/module_telegram/`이 그 환경에 미리 있어야 동작해 (본인 다른 맥에서 복사해서 옮기는 식).

## 설정값

- **봇**: `gogo_agent_bot` (goagent_bot)
- **현빈 chat_id**: `6942656480`
- **클린 모듈 경로**: `~/Documents/github_cloud/module_telegram/module/index.js`

## 실행 방법

아래 node.js 코드를 Bash 툴로 실행:

```bash
node -e "
const { sendToHyunbin } = require(require('os').homedir() + '/Documents/github_cloud/module_telegram/module/index.js');
sendToHyunbin(\`{메시지 내용}\`)
  .then(r => console.log('전송 완료:', r.message_id))
  .catch(e => console.error('전송 실패:', e.message));
"
```

- `{메시지 내용}` 자리에 사용자가 요청한 메시지를 넣어서 실행
- 성공 시 `전송 완료: {message_id}` 출력
- 실패 시 오류 메시지 출력 후 사용자에게 알림

## 주의사항

- 메시지 내용에 백틱(`` ` ``) 포함 시 이스케이프 처리 필요
- 긴 메시지(4096자 초과) 시 잘라서 여러 번 전송

---

## 다른 Claude Code에서 직접 모듈로 사용하기

프로젝트 완료 후 현빈에게 보고할 때, 아래 모듈을 직접 `require`해서 사용하면 돼.

### 모듈 경로

```
~/Documents/github_cloud/module_telegram/module/index.js
```

### 사용 코드 (복붙용)

```javascript
const path = require('path');
const { sendToHyunbin } = require(path.join(require('os').homedir(), 'Documents/github_cloud/module_telegram/module/index.js'));

await sendToHyunbin('메시지 내용');
```

### Bash 한 줄로 실행

```bash
node -e "
const { sendToHyunbin } = require(require('os').homedir() + '/Documents/github_cloud/module_telegram/module/index.js');
sendToHyunbin('여기에 메시지')
  .then(r => console.log('전송 완료:', r.message_id))
  .catch(e => console.error('전송 실패:', e.message));
"
```

### 요약

| 항목 | 값 |
|------|-----|
| 봇 이름 | `gogo_agent_bot` |
| 현빈 chat_id | `6942656480` |
| 함수 | `sendToHyunbin(text) => Promise` |
| (선택) low-level | `sendMessage(botToken, chatId, text) => Promise` |
