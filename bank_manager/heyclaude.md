# heyclaude.md — bank_manager 환경 세팅

> 이 파일은 Claude Code가 읽고 따라하는 안내문이야. 사람이 직접 보지 않아도 OK.

## 트리거 조건

사용자가 "bank_manager 세팅", "입금요청 스킬 처음 사용", "bank 환경 세팅해줘" 같은 말을 하거나, 아래 사전 요구사항 중 하나라도 빠져있으면 이 단계를 진행해. 모든 단계는 **idempotent** — 이미 있으면 skip.

## 사전 요구사항 체크

```bash
echo "=== bank_manager 환경 점검 ==="
[ -d "$HOME/Documents/github_cloud/module_auth" ]                                && echo "✓ module_auth"            || echo "❌ module_auth 없음"
[ -f "$HOME/Documents/github_cloud/module_api_key/.env" ]                        && echo "✓ module_api_key/.env"    || echo "❌ module_api_key/.env 없음"
[ -f "$HOME/.claude/skills/bank_manager/scripts/payment_request.py" ]            && echo "✓ payment_request.py (스킬에 포함됨)" || echo "❌ payment_request.py 없음"
[ -f "$HOME/.claude/skills/bank_manager/scripts/favorites.json" ]                && echo "✓ favorites.json"         || echo "❌ favorites.json 없음"
python3 -c "import google.auth, google_auth_oauthlib, googleapiclient" 2>/dev/null && echo "✓ Google API 패키지" || echo "❌ Google API 패키지 없음"
grep -q '^SPREADSHEET_ID = ""' "$HOME/.claude/skills/bank_manager/scripts/payment_request.py" 2>/dev/null && echo "⚠️ SPREADSHEET_ID 미설정 (Step 5)" || echo "✓ SPREADSHEET_ID 설정됨"
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

`.env`와 OAuth credentials를 담은 폴더. **Google Drive에서 수동 다운로드** 필요.

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
  echo "→ 사용자가 다운로드 완료했다고 알려주면 다음 단계 진행"
else
  echo "✓ module_api_key/.env 이미 있음"
fi
```

---

## Step 3. 스크립트 + favorites.json 확인

```bash
SCRIPTS="$HOME/.claude/skills/bank_manager/scripts"
[ -f "$SCRIPTS/payment_request.py" ] && echo "✓ payment_request.py" || echo "❌ payment_request.py 누락"
[ -f "$SCRIPTS/favorites.json" ]     && echo "✓ favorites.json"     || echo "❌ favorites.json 누락"
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

## Step 5. payment_request.py에 본인 시트 정보 입력

`SPREADSHEET_ID`와 `SHEET_NAME`이 비어있으면 스크립트가 동작 안 함.

```bash
SCRIPT="$HOME/.claude/skills/bank_manager/scripts/payment_request.py"
if grep -q '^SPREADSHEET_ID = ""' "$SCRIPT"; then
  echo "⚠️ SPREADSHEET_ID/SHEET_NAME 입력이 필요. 사용자에게 안내:"
  echo ""
  echo "  1. 본인이 입금요청을 기록할 Google Sheets를 준비"
  echo "     (E=항목, F=받는사람, I=계좌, J=주민/사업자번호, K=금액, P=상태(입금요청)"
  echo "      열 구조 — 다르면 스크립트 상단 COL_* 변수 조정 필요)"
  echo ""
  echo "  2. 스크립트 열기:"
  echo "     $SCRIPT"
  echo ""
  echo "  3. 상단의 SPREADSHEET_ID, SHEET_NAME 채우기:"
  echo "     SPREADSHEET_ID = \"<URL의 /d/ 뒤 ID>\""
  echo "     SHEET_NAME = \"<탭 이름>\""
  echo ""
  echo "→ 사용자가 채웠다고 알려주면 Step 6로 진행"
else
  echo "✓ SPREADSHEET_ID 설정됨"
fi
```

---

## Step 6. 첫 실행 + 셀프 테스트 (OAuth 토큰 발급)

처음 sheet API 호출 시 브라우저 OAuth 로그인 화면이 뜸.

```bash
python3 "$HOME/.claude/skills/bank_manager/scripts/payment_request.py" --list
```

입금요청 목록(빈 목록이라도 OK)이 출력되면 통과. 권한 오류(`PERMISSION_DENIED`)가 나면 본인 Google 계정에 해당 시트 편집 권한 있는지 확인.

---

## 검증

위 Step 6에서 명령이 에러 없이 동작하면 세팅 완료. 이후 사용자가 "X에게 Y원 입금 등록해줘", "자주쓰는 곳 등록해줘" 같은 말을 하면 정상 동작.

> 참고: `favorites.json`은 빈 객체로 시작해. `--add-favorite` 명령으로 자주 쓰는 거래처를 등록하면 됨.
