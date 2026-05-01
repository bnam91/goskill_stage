# heyclaude.md — gdrive_manager 환경 세팅

> 이 파일은 Claude Code가 읽고 따라하는 안내문이야. 사람이 직접 보지 않아도 OK.

## 트리거 조건

사용자가 "gdrive_manager 세팅", "구글 드라이브 스킬 처음 사용", "gdrive 환경 세팅해줘" 같은 말을 하거나, 아래 사전 요구사항 중 하나라도 빠져있으면 이 단계를 진행해. 모든 단계는 **idempotent** — 이미 있으면 skip.

## 사전 요구사항 체크

```bash
echo "=== gdrive_manager 환경 점검 ==="
[ -d "$HOME/Documents/github_cloud/module_auth" ]                                && echo "✓ module_auth"            || echo "❌ module_auth 없음"
[ -f "$HOME/Documents/github_cloud/module_api_key/.env" ]                        && echo "✓ module_api_key/.env"    || echo "❌ module_api_key/.env 없음"
[ -f "$HOME/.claude/skills/gdrive_manager/scripts/gdrive_manager.py" ]           && echo "✓ gdrive_manager.py (스킬에 포함됨)" || echo "❌ gdrive_manager.py 없음 (goskill 다운로드 다시 시도)"
[ -f "$HOME/.claude/skills/gdrive_manager/scripts/drives.json" ]                 && echo "✓ drives.json"            || echo "❌ drives.json 없음"
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

## Step 3. gdrive_manager 스크립트 확인

스킬과 같이 배포되는 파일들 (`~/.claude/skills/gdrive_manager/scripts/`). goskill로 다운로드 시 자동으로 함께 받음.

```bash
SCRIPTS="$HOME/.claude/skills/gdrive_manager/scripts"
[ -f "$SCRIPTS/gdrive_manager.py" ] && echo "✓ gdrive_manager.py" || echo "❌ gdrive_manager.py 누락"
[ -f "$SCRIPTS/drives.json" ]       && echo "✓ drives.json"       || echo "❌ drives.json 누락"
```

`drives.json`이 비어있으면 정상 (직원이 직접 자주 쓰는 드라이브 별칭을 등록).

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

## Step 5. 첫 실행 + 셀프 테스트 (OAuth 토큰 발급)

처음 실행 시 브라우저에서 Google 로그인 화면이 뜸. 로그인하면 토큰이 `module_auth` 폴더에 저장되어 이후 자동 인증.

```bash
python3 "$HOME/.claude/skills/gdrive_manager/scripts/gdrive_manager.py" --index
```

인덱스 아키텍처 표가 출력되면 (Drive API 호출 없이 동작) 스크립트 경로/모듈 정상.

실제 Drive API 테스트는 별칭 1개 등록한 뒤:

```bash
# 예시: 본인이 자주 쓰는 회사 공용 폴더 등록 (URL의 folders/ 뒤 ID)
# python3 "$HOME/.claude/skills/gdrive_manager/scripts/gdrive_manager.py" --add-drive --alias 회사 --folder-id <폴더ID>
# python3 "$HOME/.claude/skills/gdrive_manager/scripts/gdrive_manager.py" --list --drive 회사
```

→ 사용자에게 자주 쓰는 폴더 1개 알려달라고 요청 후 등록 + 조회 테스트.

---

## 검증

위 Step 5에서 OAuth 통과 + 등록한 별칭으로 폴더 조회가 정상 출력되면 세팅 완료. 이후 사용자가 "X 드라이브 보여줘", "파일 찾아줘" 같은 말을 하면 정상 동작.

권한 오류(`PERMISSION_DENIED`, `not found`)가 나면 해당 폴더에 본인 Google 계정 읽기 권한 있는지 확인 후 다시 시도.

## TODO

- **Drive → ImgBB 변환 기능** (`utils_imgbb` 의존): 별도 외부 모듈 정식 배포 메커니즘 미정. 사용 시점에 본인 환경에 `~/Documents/github_cloud/utils_mac/utils_imgbb` 따로 설치 필요.
