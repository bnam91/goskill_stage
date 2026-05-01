# heyclaude.md — nlm 환경 세팅

> 이 파일은 Claude Code가 읽고 따라하는 안내문이야. 사람이 직접 보지 않아도 OK.

## 트리거 조건

사용자가 "nlm 세팅", "노트북LM 단축어 처음 사용", "/nlm" 같은 말을 하거나, 아래 사전 요구사항 중 하나라도 빠져있으면 진행. 모든 단계는 **idempotent** — 이미 있으면 skip.

## 사전 요구사항 체크

```bash
echo "=== nlm 환경 점검 ==="
command -v "$HOME/.local/bin/nlm" >/dev/null 2>&1 || command -v nlm >/dev/null 2>&1 \
                                                            && echo "✓ nlm CLI 설치됨" || echo "❌ nlm CLI 없음 (Step 1)"
( "$HOME/.local/bin/nlm" alias list 2>/dev/null || nlm alias list 2>/dev/null ) | grep -q . \
                                                            && echo "✓ alias 1개 이상 등록됨"  || echo "⚠️ alias 미등록 (Step 3)"
curl -s --connect-timeout 1 http://localhost:9222/json/version >/dev/null 2>&1 \
                                                            && echo "✓ chrome-cdp 9222 열림 (브라우저 모드 가능)" || echo "ℹ️ chrome-cdp 9222 닫힘 (브라우저 모드 사용 시 /chrome-cdp 필요)"
```

빠진 게 있으면 아래 단계로 보충.

---

## Step 1. nlm CLI 설치

`nlm`은 NotebookLM 비공식 CLI (`notebooklm-mcp-cli` 패키지). Python uv 또는 pipx로 설치.

```bash
if command -v "$HOME/.local/bin/nlm" >/dev/null 2>&1 || command -v nlm >/dev/null 2>&1; then
  echo "✓ nlm 이미 설치됨"
else
  if command -v uv >/dev/null 2>&1; then
    uv tool install notebooklm-mcp-cli
    echo "✅ uv tool로 설치 완료"
  elif command -v pipx >/dev/null 2>&1; then
    pipx install notebooklm-mcp-cli
    echo "✅ pipx로 설치 완료"
  else
    echo "⚠️ uv 또는 pipx 가 필요함. 사용자에게 안내:"
    echo ""
    echo "  옵션 1 (uv 권장): brew install uv 또는 https://docs.astral.sh/uv/"
    echo "  옵션 2 (pipx):    brew install pipx"
    echo ""
    echo "  설치 후 다시:    uv tool install notebooklm-mcp-cli"
    echo "                또는: pipx install notebooklm-mcp-cli"
    echo ""
    echo "→ 사용자가 설치 완료했다고 알려주면 다음 단계 진행"
  fi
fi

# PATH 확인 — uv tool은 ~/.local/bin/ 에 심볼릭 링크 만듦
case ":$PATH:" in
  *":$HOME/.local/bin:"*) echo "✓ PATH에 ~/.local/bin 포함" ;;
  *) echo "⚠️ ~/.local/bin 이 PATH에 없음. ~/.zshrc 또는 ~/.bashrc에 추가:"
     echo '   export PATH="$HOME/.local/bin:$PATH"' ;;
esac
```

---

## Step 2. 첫 인증 (Google OAuth)

`nlm`을 처음 어떤 명령으로든 호출하면 Google 로그인 화면이 브라우저에 뜸. 본인 NotebookLM 접근 권한이 있는 Google 계정으로 로그인하면 토큰이 로컬에 저장돼서 이후 자동 인증.

```bash
nlm alias list
```

→ 처음 실행 시 브라우저가 열림. 로그인 완료 후 터미널에 alias 목록(빈 목록이라도 OK)이 출력되면 인증 통과.

권한 오류나 401 같은 응답이 나오면 NotebookLM이 활성화된 Google 계정으로 로그인했는지 재확인.

---

## Step 3. 본인 자주 쓰는 노트북 alias 등록

이 스킬의 핵심 가치는 **긴 UUID 대신 짧은 별칭으로 노트북 호출**하는 것. 사용자가 자주 쓰는 노트북 1~2개부터 등록.

### alias 등록 방법

```bash
# 노트북 UUID 확인 — NotebookLM URL의 /notebook/<여기> 부분
# 예: https://notebooklm.google.com/notebook/5ad76bf3-0328-4f8d-8fca-bbadc52c3b23

# 등록
nlm alias set <alias-name> <UUID>

# 확인
nlm alias list
```

→ 사용자에게 "자주 쓰는 NotebookLM 노트북 URL 1~2개 알려주세요" 라고 요청 후, URL에서 UUID 추출해서 alias 등록.

```bash
# 예시 — 사용자가 URL 알려주면 그걸로
# nlm alias set my_research 5ad76bf3-0328-4f8d-8fca-bbadc52c3b23
```

---

## Step 4. chrome-cdp 9222 (선택 — 브라우저 열기 모드용)

`/nlm <alias>` (질문 없이) 형태로 호출하면 NotebookLM 노트북이 브라우저에 열려야 함. 이때 chrome-cdp 9222 포트가 필요.

```bash
if curl -s --connect-timeout 1 http://localhost:9222/json/version >/dev/null 2>&1; then
  echo "✓ chrome-cdp 9222 이미 열림"
else
  echo "ℹ️ chrome-cdp 9222 닫힘. 브라우저 모드 쓰려면:"
  echo "   사용자에게 '/chrome-cdp -coq3820' 실행 안내 (또는 본인 프로필 인자)"
  echo "→ CLI 질문만 쓸 거면 이 단계 skip 가능"
fi
```

CLI 질문만 사용(`/nlm <alias> <질문>`) 한다면 이 단계는 필요 없음.

---

## 검증

```bash
# alias 목록에 등록한 게 보이는지
nlm alias list

# 등록한 alias로 짧은 질문 테스트
# nlm notebook query <등록한-alias> "이 노트북은 무엇에 관한 거야?"
```

질문 응답이 정상 출력되면 세팅 완료. 이후 사용자가 "/nlm <alias>", "/nlm <alias> <질문>" 같은 말을 하면 정상 동작.

> 참고: SKILL.md 하단의 "참고 - 등록된 alias" 표는 본인 환경 예시. 직원 본인이 등록한 alias가 실제 동작 대상.
