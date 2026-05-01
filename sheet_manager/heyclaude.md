# heyclaude.md — sheet_manager 환경 세팅

> 이 파일은 Claude Code가 읽고 따라하는 안내문입니다. 사람이 직접 보지 않아도 OK.

## 트리거 조건

사용자가 "sheet_manager 세팅", "구글시트 스킬 처음 사용", "sheet_manager 환경 세팅해줘" 같은 말을 하거나, 아래 사전 요구사항 중 하나라도 빠져있으면 이 단계를 진행한다. 모든 단계는 **idempotent** — 이미 있으면 skip.

## 사전 요구사항 체크

```bash
echo "=== sheet_manager 환경 점검 ==="
[ -d "$HOME/Documents/github_cloud/module_auth" ]                              && echo "✓ module_auth"            || echo "❌ module_auth 없음"
[ -f "$HOME/Documents/github_cloud/module_api_key/.env" ]                      && echo "✓ module_api_key/.env"    || echo "❌ module_api_key/.env 없음"
[ -f "$HOME/.claude/skills/sheet_manager/scripts/sheet_manager.py" ]           && echo "✓ sheet_manager.py (스킬에 포함됨)" || echo "❌ sheet_manager.py 없음 (goskill 다운로드 다시 시도)"
python3 -c "import google.auth, google_auth_oauthlib, googleapiclient" 2>/dev/null && echo "✓ Google API 패키지" || echo "❌ Google API 패키지 없음"
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

`.env`와 OAuth credentials를 담은 폴더. **Google Drive에서 수동 다운로드** 필요 (자격증명이라 git에 두지 않음).

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

## Step 3. sheet_manager Python 스크립트 확인

스킬과 같이 배포되는 파일 (`~/.claude/skills/sheet_manager/scripts/sheet_manager.py`). goskill로 다운로드 시 자동으로 함께 받음 — 별도 설치 단계 없음.

```bash
SCRIPT="$HOME/.claude/skills/sheet_manager/scripts/sheet_manager.py"
if [ -f "$SCRIPT" ]; then
  echo "✓ sheet_manager.py 정상 (스킬에 포함됨)"
else
  echo "❌ sheet_manager.py 누락 — goskill 앱에서 sheet_manager 다시 다운로드하세요"
fi
```

---

## Step 4. Python 패키지 설치

```bash
if python3 -c "import google.auth, google_auth_oauthlib, googleapiclient" 2>/dev/null; then
  echo "✓ Google API 패키지 이미 설치됨"
else
  pip install google-auth google-auth-oauthlib google-api-python-client
  echo "✅ Python 패키지 설치 완료"
fi
```

---

## Step 5. 첫 실행 + 셀프 테스트 (OAuth 토큰 발급)

처음 sheet_manager 실행 시 브라우저에서 Google 로그인 화면이 뜬다. 로그인하면 토큰이 `module_auth` 폴더에 저장되어 이후 자동 인증.

회사 공용 **테스트 시트**로 동작 확인 (사용자에게 별도로 물어보지 말 것 — 이미 박혀있음):

```bash
TEST_SHEET_ID="1grl5SNwOWmoVnq5Rdxt0gp1vnfrnwl5KJwIq67CJkic"
python3 "$HOME/.claude/skills/sheet_manager/scripts/sheet_manager.py" tabs "$TEST_SHEET_ID"
```

탭 목록이 정상 출력되면 OAuth + 인증 + 스크립트 모두 통과.

권한 오류(`PERMISSION_DENIED`, `not found`)가 나면 회사 관리자에게 해당 시트 읽기 권한 요청 후 다시 시도.

---

## 검증

위 Step 5 명령어가 탭 목록을 출력하면 세팅 완료. 이후 사용자가 "시트 읽어줘", "구글시트에 써줘" 같은 말을 하면 정상적으로 SKILL.md의 명령어들이 동작한다.

테스트 시트 URL (직원이 직접 확인하고 싶으면):
https://docs.google.com/spreadsheets/d/1grl5SNwOWmoVnq5Rdxt0gp1vnfrnwl5KJwIq67CJkic/edit
