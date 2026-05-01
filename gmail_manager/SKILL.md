---
name: gmail_manager
description: Gmail 계정을 관리하는 스킬이야. 여러 계정을 config로 관리하고 메일 읽기/목록 조회를 할 수 있어.
---

Gmail 계정을 관리하는 스킬이야. 여러 계정을 config로 관리하고 메일 읽기/목록 조회를 할 수 있어.
스크립트 경로: ~/.claude/skills/gmail_manager/scripts/gmail_manager.py

> **🚀 처음 사용하는 환경이라면**: 같은 폴더의 `heyclaude.md`를 따라 환경 세팅을 먼저 진행. 사전 요구사항(module_auth, module_api_key/.env, Google API 패키지) 중 하나라도 빠져있으면 자동으로 거기로 이동. (`scripts/gmail_manager.py`는 이 스킬 폴더에 같이 들어있어서 별도 설치 불필요.)

## OS별 명령어 차이

- **Python 명령어**: Windows는 `python`, Mac/Linux는 `python3`
- **인코딩**: Windows는 모든 명령어 앞에 `PYTHONUTF8=1` 접두사 필요
- **브라우저 열기**: Windows는 `start ""`, Mac은 `open`

현재 OS를 확인 후 적절한 명령어를 사용해:
- Windows: `PYTHONUTF8=1 python gmail_manager.py ...`
- Mac/Linux: `python3 gmail_manager.py ...`

## 구현 예정 기능 (추후 업데이트)

### 개발 예정
- **답장 (Reply)** - 메일 ID 기반 스레드 유지 회신
- **첨부파일 다운로드** - 메일 첨부파일 로컬 저장
- **첨부파일 포함 발송** - 파일 첨부하여 메일 발송
- **스레드 조회** - 같은 대화 흐름 전체 보기

### 검토 예정
- **전달 (Forward)** - 받은 메일 다른 주소로 포워딩
- **라벨 관리** - 중요 표시, 읽음 처리 등
- **임시저장 목록** - Draft 목록 조회 및 발송

## 계정 관련

### 계정 목록 보기
```bash
PYTHONUTF8=1 python3 ~/.claude/skills/gmail_manager/scripts/gmail_manager.py --list-accounts
```

### 계정 추가
```bash
PYTHONUTF8=1 python3 ~/.claude/skills/gmail_manager/scripts/gmail_manager.py --add-account --alias 별칭 --email 이메일주소
```
처음 추가 시 브라우저 OAuth 로그인 필요. 이후는 자동 인증.

### 계정 제거
```bash
PYTHONUTF8=1 python3 ~/.claude/skills/gmail_manager/scripts/gmail_manager.py --remove-account --alias 별칭
```

## 메일 관련

### 메일 목록 조회
```bash
# 기본 (받은편지함 20개)
PYTHONUTF8=1 python3 ~/.claude/skills/gmail_manager/scripts/gmail_manager.py --list-mails --account 별칭

# 한국 시간 기준 오늘 메일
PYTHONUTF8=1 python3 ~/.claude/skills/gmail_manager/scripts/gmail_manager.py --list-mails --account 별칭 --today

# 오늘 보낸 메일
PYTHONUTF8=1 python3 ~/.claude/skills/gmail_manager/scripts/gmail_manager.py --list-mails --account 별칭 --today --query "in:sent"

# 검색 쿼리 포함
PYTHONUTF8=1 python3 ~/.claude/skills/gmail_manager/scripts/gmail_manager.py --list-mails --account 별칭 --query "is:unread"

# 개수 지정
PYTHONUTF8=1 python3 ~/.claude/skills/gmail_manager/scripts/gmail_manager.py --list-mails --account 별칭 --max 50
```

### 메일 읽기
```bash
PYTHONUTF8=1 python3 ~/.claude/skills/gmail_manager/scripts/gmail_manager.py --read-mail --account 별칭 --id 메일ID
```
메일 ID는 목록 조회 시 [] 안에 표시되는 8자리 앞부분. 전체 ID를 사용해야 함.

### 휴지통 이동
```bash
# 단일
PYTHONUTF8=1 python3 ~/.claude/skills/gmail_manager/scripts/gmail_manager.py --trash-mail --account 별칭 --id 메일ID

# 여러 개
PYTHONUTF8=1 python3 ~/.claude/skills/gmail_manager/scripts/gmail_manager.py --trash-mail --account 별칭 --ids ID1,ID2,ID3
```

## 검색 쿼리 예시
- `is:unread` - 안읽은 메일
- `is:inbox` - 받은편지함
- `from:someone@gmail.com` - 특정 발신자
- `subject:회의` - 제목에 '회의' 포함
- `has:attachment` - 첨부파일 있는 메일
- `after:2026/01/01` - 특정 날짜 이후

## 메일 정리

사용자가 "메일 정리해줘" 라고 하면:
1. 오늘 받은 메일 목록 조회
2. `~/.claude/skills/gmail_manager/scripts/mail_organizer.md` 의 분류 기준 참고
3. 각 메일을 🔴삭제 / 🟡보류 / 🟢보존 으로 분류해서 표로 보여주기
4. 사용자 확인 후 삭제 대상 일괄 휴지통 이동

## 사용자 요청 처리

- "계정 추가해줘" → --add-account 안내 및 실행
- "계정 목록 보여줘" → --list-accounts 실행
- "메일 목록 보여줘" → 어떤 계정인지 확인 후 --list-mails 실행
- "안읽은 메일 보여줘" → --list-mails --query "is:unread" 실행
- "이 메일 읽어줘" / "전문 보여줘" → --read-mail 실행 후 본문을 터미널 출력이 아닌 텍스트로 직접 보여줘
- "설정 방법 알려줘" → heyclaude.md 안내

## 메일 브라우저로 열기

각 계정의 Gmail 브라우저 URL은 사용자가 로그인한 순서에 따라 `u/0`, `u/1`, `u/2` 등으로 다름. 사용자에게 어떤 계정이 몇 번째인지 확인하고 사용:

```bash
# Windows
start "" "https://mail.google.com/mail/u/<번호>/#all/메일ID"

# Mac/Linux
open "https://mail.google.com/mail/u/<번호>/#all/메일ID"
```

## 출력 형식 규칙

- **마크다운 테이블 사용 금지** - 메일 목록, 계정 목록 등 모든 출력에서 `|` 테이블 사용하지 말 것
- 목록은 번호 리스트(`1. 2. 3.`) 또는 불릿(`-`) 사용

## 메일 전문 표시 방법

--read-mail 결과를 터미널 출력 그대로 붙이지 말고, 아래 형식으로 텍스트 직접 출력:

**제목:**
**발신:**
**수신:**
**날짜:**

---

(본문 전체)

---

(원본 메일이 있으면 인용 포함)

결과는 한국어로 정리해서 보여줘.
