# heyclaude.md — notion_manager 환경 세팅

> 이 파일은 Claude Code가 읽고 따라하는 안내문이야. 사람이 직접 보지 않아도 OK.

## 트리거 조건

사용자가 "notion_manager 세팅", "노션 스킬 처음 사용", "노션 스킬 환경 세팅해줘" 같은 말을 하거나, 아래 사전 요구사항 중 하나라도 빠져있으면 이 단계를 진행해. 모든 단계는 **idempotent** — 이미 있으면 skip.

## 사전 요구사항 체크

```bash
echo "=== notion_manager 환경 점검 ==="
[ -f "$HOME/.claude/skills/notion_manager/scripts/notion_manager.js" ]    && echo "✓ notion_manager.js (스킬에 포함됨)" || echo "❌ 스크립트 없음 (goskill 다시 다운로드)"
command -v node >/dev/null 2>&1                                           && echo "✓ Node.js ($(node --version 2>/dev/null))" || echo "❌ Node.js 없음"
[ -f "$HOME/.claude/skills/notion_manager/scripts/config.json" ]          && echo "✓ config.json"                || echo "❌ config.json 없음 (Step 2)"
```

빠진 게 있으면 아래 단계로 보충.

---

## Step 1. Node.js 설치 확인

`notion_manager.js`는 Node.js 18+ 가 필요(내장 fetch 사용).

```bash
if ! command -v node >/dev/null 2>&1; then
  echo "❌ Node.js 없음. 설치 안내:"
  echo ""
  echo "  - macOS (Homebrew): brew install node"
  echo "  - Windows: https://nodejs.org/ 에서 LTS 다운로드"
  echo "  - Linux: nvm 또는 패키지 매니저"
  echo ""
  echo "→ 사용자가 설치 완료했다고 알려주면 다음 단계 진행"
else
  NODE_MAJOR=$(node --version | sed 's/v\([0-9]*\)\..*/\1/')
  if [ "$NODE_MAJOR" -lt 18 ]; then
    echo "⚠️ Node.js 버전이 낮음 (현재: $(node --version)). 18+ 필요."
  else
    echo "✓ Node.js $(node --version)"
  fi
fi
```

---

## Step 2. config.json 생성 + Notion API 키 입력

처음 사용하는 직원은 본인 Notion API 키를 발급받아 등록해야 함.

```bash
SCRIPTS_DIR="$HOME/.claude/skills/notion_manager/scripts"
EXAMPLE="$SCRIPTS_DIR/config.example.json"
TARGET="$SCRIPTS_DIR/config.json"

if [ -f "$TARGET" ]; then
  echo "✓ config.json 이미 있음"
else
  cp "$EXAMPLE" "$TARGET"
  echo "✅ config.json 생성됨 (placeholder 상태)"
  echo ""
  echo "⚠️ 다음 작업이 필요해. 사용자에게 안내:"
  echo ""
  echo "  1. https://www.notion.so/my-integrations 접속"
  echo "  2. '+ New integration' 클릭"
  echo "  3. 워크스페이스 선택 → 이름 입력 → Submit"
  echo "  4. 'Internal Integration Secret' (ntn_... 또는 secret_... 로 시작) 복사"
  echo "  5. $TARGET 파일을 열고 api_key 값 채우기"
  echo ""
  echo "  6. 사용할 Notion 페이지/DB 우측 상단 '...' → '연결' → 위에서 만든 Integration 선택"
  echo ""
  echo "→ 사용자가 채웠다고 알려주면 Step 3로 진행"
fi
```

---

## Step 3. 첫 실행 + 셀프 테스트

테스트 명령으로 API 키와 연결을 확인.

```bash
# 검색 API로 연결 테스트 (Integration이 연결된 페이지 1개라도 있으면 결과 나옴)
node "$HOME/.claude/skills/notion_manager/scripts/notion_manager.js" search "test" 2>&1 | head -20
```

- 결과가 정상 출력되면 통과
- `unauthorized` 또는 `Invalid token` 에러 → api_key 다시 확인
- 빈 결과 → Integration이 연결된 페이지/DB가 아직 없는 상태. 사용 시점에 페이지마다 연결하면 됨

---

## 검증

```bash
node "$HOME/.claude/skills/notion_manager/scripts/notion_manager.js" search "test"
```

명령이 에러 없이 동작하면 세팅 완료. 이후 사용자가 "노션에서 ~~ 찾아줘", "노션 페이지에 ~~ 추가해줘" 같은 말을 하면 정상 동작.
