# heyclaude.md — ads-manager 환경 세팅

> 이 파일은 Claude Code가 읽고 따라하는 안내문이야. 사람이 직접 보지 않아도 OK.

## 트리거 조건

사용자가 "ads-manager 세팅", "광고 스킬 처음 사용", "ads 환경 세팅해줘" 같은 말을 하거나, 아래 사전 요구사항 중 하나라도 빠져있으면 진행. 모든 단계 **idempotent**.

## 사전 요구사항 체크

```bash
echo "=== ads-manager 환경 점검 ==="
[ -f "$HOME/.claude/skills/ads-manager/scripts/save_ad_history.py" ]            && echo "✓ save_ad_history.py" || echo "❌ save_ad_history.py 없음"
[ -f "$HOME/.claude/skills/ads-manager/scripts/skill.md" ]                      && echo "✓ skill.md (광고 원칙)" || echo "❌ skill.md 없음"
[ -f "$HOME/.claude/skills/ads-manager/scripts/coupang_ads_official_guide.md" ] && echo "✓ coupang_ads_official_guide.md" || echo "❌ 가이드 누락"
[ -f "$HOME/Documents/claude_skills/.env" ]                                     && echo "✓ .env (광고 자격증명)" || echo "⚠️ .env 없음 (Step 2)"
command -v "$HOME/.local/bin/nlm" >/dev/null 2>&1                               && echo "✓ nlm CLI" || echo "⚠️ nlm CLI 없음 (Step 3)"
curl -s --connect-timeout 1 http://localhost:9341/json/version >/dev/null 2>&1  && echo "✓ chrome-cdp port 9341 열림" || echo "⚠️ port 9341 닫힘 (Step 4)"
[ -d "$HOME/.claude/skills/ads-manager-app" ]                                   && echo "✓ ads-manager-app 스킬 (광고 계산기)" || echo "ℹ️ ads-manager-app 미설치 — 광고 계산기 사용 시 필요"
```

빠진 게 있으면 아래 단계로 보충.

---

## Step 1. 스크립트 확인

```bash
SCRIPTS="$HOME/.claude/skills/ads-manager/scripts"
[ -f "$SCRIPTS/save_ad_history.py" ]            && echo "✓ save_ad_history.py" || echo "❌ 누락"
[ -f "$SCRIPTS/skill.md" ]                      && echo "✓ skill.md" || echo "❌ 누락"
[ -f "$SCRIPTS/coupang_ads_official_guide.md" ] && echo "✓ guide" || echo "❌ 누락"
```

누락 시 goskill 앱에서 ads-manager 다시 다운로드.

---

## Step 2. .env 자격증명 확인

쿠팡 광고센터 자동 로그인에 필요. 본인 환경에 맞는 ID/PW를 직접 채워야 함.

```bash
ENV="$HOME/Documents/claude_skills/.env"
if [ ! -f "$ENV" ]; then
  mkdir -p "$HOME/Documents/claude_skills"
  cat > "$ENV" <<'EOF'
# 쿠팡 광고센터 자격증명
COUPANG_ADS_REDCOMBO_ID=
COUPANG_ADS_REDCOMBO_PW=
COUPANG_ADS_GOYA_ID=
COUPANG_ADS_GOYA_PW=
EOF
  echo "✅ .env 템플릿 생성됨: $ENV"
  echo "→ 사용자에게 ID/PW 채우라고 안내 후 다음 단계 진행"
else
  echo "✓ .env 이미 있음"
fi
```

---

## Step 3. nlm CLI 설치 확인

NotebookLM 강의 조회용.

```bash
if command -v "$HOME/.local/bin/nlm" >/dev/null 2>&1; then
  echo "✓ nlm CLI 설치됨"
else
  echo "⚠️ nlm CLI 없음. 설치 가이드는 nlm 공식 저장소 참고."
  echo "→ NotebookLM 조회 기능을 쓰지 않을 거면 이 단계는 skip"
fi
```

---

## Step 4. chrome-cdp port 9341 열기 (광고센터 자동화)

`chrome-cdp` 스킬이 같이 설치되어 있어야 함. 광고센터 데이터 조회 전 한 번 띄워야 함.

```bash
if curl -s --connect-timeout 1 http://localhost:9341/json/version >/dev/null 2>&1; then
  echo "✓ port 9341 이미 열림"
else
  echo "⚠️ port 9341 닫힘. 사용자에게 안내:"
  echo "   '/chrome-cdp -coupang-ads' 실행 후 CC 재시작"
fi
```

---

## Step 5. ads-manager-app 동시 설치 권장

광고 계산기는 별도 스킬. 같이 설치되어 있으면 자연스럽게 연동됨.

```bash
if [ -d "$HOME/.claude/skills/ads-manager-app" ]; then
  echo "✓ ads-manager-app 설치됨 (계산기 연동 가능)"
else
  echo "ℹ️ ads-manager-app 미설치 — 광고 계산기 필요시 goskill에서 함께 다운로드 권장"
fi
```

---

## 검증

```bash
# 광고 원칙 가이드가 정상 읽히는지
head -5 "$HOME/.claude/skills/ads-manager/scripts/skill.md"

# NotebookLM 조회가 동작하는지 (선택)
# ~/.local/bin/nlm notebook query c840a495-8ab4-432e-8c9a-4317bccecb72 "광고 원칙"
```

광고센터 자동 로그인은 첫 사용 시 사용자가 .env를 채우고, port 9341 열린 상태에서 SKILL.md의 Python 코드 그대로 실행하면 됨.
