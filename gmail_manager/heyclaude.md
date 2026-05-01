# heyclaude.md — gmail_manager 환경 세팅

> 이 파일은 Claude Code가 읽고 따라하는 안내문이야. 사람이 직접 보지 않아도 OK.

## 트리거 조건

사용자가 "gmail_manager 세팅", "Gmail 스킬 처음 사용", "메일 스킬 환경 세팅해줘" 같은 말을 하거나, 아래 사전 요구사항 중 하나라도 빠져있으면 이 단계를 진행해. 모든 단계는 **idempotent** — 이미 있으면 skip.

## 사전 요구사항 체크

```bash
echo "=== gmail_manager 환경 점검 ==="
[ -d "$HOME/Documents/github_cloud/module_auth" ]                                && echo "✓ module_auth"            || echo "❌ module_auth 없음"
[ -f "$HOME/Documents/github_cloud/module_api_key/.env" ]                        && echo "✓ module_api_key/.env"    || echo "❌ module_api_key/.env 없음"
[ -f "$HOME/.claude/skills/gmail_manager/scripts/gmail_manager.py" ]             && echo "✓ gmail_manager.py (스킬에 포함됨)" || echo "❌ gmail_manager.py 없음 (goskill 다운로드 다시 시도)"
[ -f "$HOME/.claude/skills/gmail_manager/scripts/config.json" ]                  && echo "✓ config.json"            || echo "❌ config.json 없음"
python3 -c "import google.auth, google_auth_oauthlib, googleapiclient, dotenv" 2>/dev/null && echo "✓ Google API 패키지 + python-dotenv" || echo "❌ Google API 패키지 또는 python-dotenv 없음"
```

빠진 게 있으면 아래 단계로 보충.

---

## Step 1. Google 인증 모듈 (module_auth) 클론

Gmail, Drive, Sheets 스킬이 공통으로 사용하는 OAuth 모듈.

```bash
if [ ! -d "$HOME/Documents/github_cloud/module_auth" ]; then
  mkdir -p "$HOME/Documents/github_cloud"
  git clone https://github.com/bnam91/module_auth "$HOME/Documents/github_cloud/module_auth"
  echo "✅ module_auth 클론 완료"
else
  git -C "$HOME/Documents/github_cloud/module_auth" pull --ff-only 2>/dev/null
  echo "✓ module_auth 이미 있음 (최신화 시도)"
fi
```

---

## Step 2. API 키 모듈 (module_api_key) 다운로드

`.env` (GOOGLE_CLIENT_ID, GOOGLE_CLIENT_SECRET 포함) 가 들어있는 폴더. **Google Drive에서 수동 다운로드** 필요 (자격증명이라 git에 두지 않음).

```bash
TARGET="$HOME/Documents/github_cloud/module_api_key"
if [ ! -f "$TARGET/.env" ]; then
  mkdir -p "$TARGET"
  echo "⚠️ module_api_key/.env 가 없습니다. 사용자에게 안내:"
  echo ""
  echo "  Google Drive 링크에서 폴더 전체를 다운로드해주세요:"
  echo "  https://drive.google.com/drive/u/0/folders/1opkiTG09SP9_a3pHUy41DcnCVjUOPPK0"
  echo ""
  echo "  다운로드한 파일들을 다음 경로에 배치:"
  echo "  $TARGET/"
  echo ""
  echo "  (회사 관리자로부터 받은 링크여야 함. 폴더 ID가 다를 수 있음.)"
  echo ""
  echo "→ 사용자가 다운로드 완료했다고 알려주면 다음 단계 진행"
else
  echo "✓ module_api_key/.env 이미 있음"
fi
```

---

## Step 3. gmail_manager 스크립트 + config.json 확인

스킬과 같이 배포되는 파일들. 직원의 첫 사용 시 `config.json`은 빈 accounts 상태로 시작.

```bash
SCRIPTS="$HOME/.claude/skills/gmail_manager/scripts"
[ -f "$SCRIPTS/gmail_manager.py" ]    && echo "✓ gmail_manager.py" || echo "❌ gmail_manager.py 누락"
[ -f "$SCRIPTS/config.json" ]         && echo "✓ config.json"      || echo "❌ config.json 누락"
[ -f "$SCRIPTS/mail_organizer.md" ]   && echo "✓ mail_organizer.md (메일 정리 분류 기준)" || echo "ℹ️ mail_organizer.md 없음 (선택 기능)"
```

---

## Step 4. Python 패키지 설치

```bash
if python3 -c "import google.auth, google_auth_oauthlib, googleapiclient, dotenv" 2>/dev/null; then
  echo "✓ Google API 패키지 + python-dotenv 이미 설치됨"
else
  pip install google-auth google-auth-oauthlib google-api-python-client python-dotenv
  echo "✅ Python 패키지 설치 완료"
fi
```

---

## Step 5. 본인 Gmail 계정 추가 + OAuth 첫 로그인

직원 본인이 사용할 Gmail 계정을 별칭과 함께 등록. 처음 등록 시 브라우저에서 Google OAuth 로그인 화면이 뜸 — 로그인하면 토큰이 `tokens/` 폴더에 저장되어 이후 자동 인증.

```bash
# 사용자에게 별칭과 이메일 주소 물어본 뒤 실행
# 예: 별칭=work, 이메일=hong@goyamkt.com
# python3 "$HOME/.claude/skills/gmail_manager/scripts/gmail_manager.py" --add-account --alias <별칭> --email <이메일>
```

→ 사용자에게 본인 별칭/이메일 알려달라고 요청 후 실행.

---

## Step 6. 셀프 테스트

```bash
# 등록된 계정 목록 확인
PYTHONUTF8=1 python3 "$HOME/.claude/skills/gmail_manager/scripts/gmail_manager.py" --list-accounts

# 메일 목록 조회 (방금 등록한 별칭으로)
# PYTHONUTF8=1 python3 "$HOME/.claude/skills/gmail_manager/scripts/gmail_manager.py" --list-mails --account <별칭>
```

목록이 정상 출력되면 통과.

---

## 검증

위 Step 6에서 메일 목록이 정상 출력되면 세팅 완료. 이후 사용자가 "메일 목록 보여줘", "안읽은 메일 보여줘", "메일 정리해줘" 같은 말을 하면 정상 동작.

OAuth 토큰은 `~/.claude/skills/gmail_manager/scripts/tokens/<별칭>.json`에 저장되어 재로그인 없이 사용.
