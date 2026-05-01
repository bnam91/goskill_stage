# heyclaude.md — hometax_manager 환경 세팅

> 이 파일은 Claude Code가 읽고 따라하는 안내문이야. 사람이 직접 보지 않아도 OK.

## 트리거 조건

사용자가 "hometax_manager 세팅", "홈택스 스킬 처음 사용", "세금계산서 환경 세팅해줘" 같은 말을 하거나, 아래 사전 요구사항 중 하나라도 빠져있으면 이 단계를 진행해. 모든 단계는 **idempotent** — 이미 있으면 skip.

## 사전 요구사항 체크

```bash
echo "=== hometax_manager 환경 점검 ==="
[ -d "$HOME/Documents/github_cloud/module_auth" ]                                && echo "✓ module_auth"            || echo "❌ module_auth 없음"
[ -f "$HOME/Documents/github_cloud/module_api_key/.env" ]                        && echo "✓ module_api_key/.env"    || echo "❌ module_api_key/.env 없음"
[ -f "$HOME/.claude/skills/hometax_manager/scripts/tax_invoice.py" ]             && echo "✓ tax_invoice.py"         || echo "❌ tax_invoice.py 없음"
[ -f "$HOME/.claude/skills/hometax_manager/scripts/vendor.py" ]                  && echo "✓ vendor.py"              || echo "❌ vendor.py 없음"
python3 -c "import google.auth, google_auth_oauthlib, googleapiclient" 2>/dev/null && echo "✓ Google API 패키지" || echo "❌ Google API 패키지 없음"
grep -q '^SUPPLIER = ""' "$HOME/.claude/skills/hometax_manager/scripts/tax_invoice.py" 2>/dev/null && echo "⚠️ SUPPLIER/SPREADSHEET_ID 미설정 (Step 5)" || echo "✓ 스크립트 설정값 입력됨"
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

## Step 3. 스크립트 확인

```bash
SCRIPTS="$HOME/.claude/skills/hometax_manager/scripts"
[ -f "$SCRIPTS/tax_invoice.py" ] && echo "✓ tax_invoice.py" || echo "❌ tax_invoice.py 누락"
[ -f "$SCRIPTS/vendor.py" ]      && echo "✓ vendor.py"      || echo "❌ vendor.py 누락"
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

## Step 5. tax_invoice.py / vendor.py 에 본인 정보 입력

회사 상호, 시트 ID, 시트 탭 이름이 비어있으면 스크립트가 동작 안 함.

```bash
TAX="$HOME/.claude/skills/hometax_manager/scripts/tax_invoice.py"
VENDOR="$HOME/.claude/skills/hometax_manager/scripts/vendor.py"

if grep -q '^SUPPLIER = ""' "$TAX"; then
  echo "⚠️ 본인 정보 입력이 필요. 사용자에게 안내:"
  echo ""
  echo "  1. 본인 회사 세금계산서/거래처 시트 준비 (Google Sheets)"
  echo "     - 시트 1: 세금계산서(발행) — A~S 19개 열 구조"
  echo "     - 시트 2: 거래처 — A~I 9개 열 구조"
  echo "     자세한 열 구조는 SKILL.md 참조"
  echo ""
  echo "  2. 두 스크립트 열기:"
  echo "     $TAX"
  echo "     $VENDOR"
  echo ""
  echo "  3. tax_invoice.py 상단에서 채우기:"
  echo "     SUPPLIER = \"<본인 회사 상호>\""
  echo "     SPREADSHEET_ID = \"<URL의 /d/ 뒤 ID>\""
  echo "     SHEET_INVOICE = \"<세금계산서 탭 이름>\""
  echo "     SHEET_VENDOR  = \"<거래처 탭 이름>\""
  echo ""
  echo "  4. vendor.py 상단에서 채우기:"
  echo "     SPREADSHEET_ID = \"<tax_invoice.py와 동일>\""
  echo "     SHEET_NAME = \"<거래처 탭 이름>\""
  echo ""
  echo "→ 사용자가 채웠다고 알려주면 Step 6로 진행"
else
  echo "✓ 스크립트 설정값 입력됨"
fi
```

---

## Step 6. 첫 실행 + 셀프 테스트 (OAuth 토큰 발급)

처음 sheet API 호출 시 브라우저 OAuth 로그인 화면이 뜸.

```bash
cd "$HOME/.claude/skills/hometax_manager/scripts" && \
python3 vendor.py --list
```

거래처 목록(빈 목록이라도 OK)이 출력되면 통과. 권한 오류가 나면 본인 Google 계정에 해당 시트 편집 권한 있는지 확인.

---

## 검증

위 Step 6에서 명령이 에러 없이 동작하면 세팅 완료. 이후 사용자가 "X 거래처 계산서 발행해줘", "거래처 목록 보여줘" 같은 말을 하면 정상 동작.
