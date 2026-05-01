---
name: ads-manager
description: 쿠팡 광고 관리 스킬이야. 광고 전략 상담, NotebookLM 강의 내용 조회, 캠페인/광고그룹/광고 세팅 가이드를 제공해. 사용자가 "광고 관리", "광고 전략", "쿠팡 광고", "/ads-manager" 등을 말할 때 실행해.
version: 1.0.0
---

# ads-manager — 쿠팡 광고 관리

> **🚀 처음 사용하는 환경이라면**: 같은 폴더의 `heyclaude.md`를 따라 환경 세팅을 먼저 진행. nlm CLI, chrome-cdp 9341 포트, .env 자격증명이 필요해.

## NotebookLM 연동

| 노트북명 | ID | 용도 |
|---------|-----|------|
| 1억맵 4주차 강의 | `5ad76bf3-0328-4f8d-8fca-bbadc52c3b23` | 1억맵 4주차 강의 MD (1+2교시, 3교시) |
| 쿠팡 광고 도움말 | `c840a495-8ab4-432e-8c9a-4317bccecb72` | ads.coupang.com 공식 가이드 수집본 |

- **nlm CLI 경로**: `~/.local/bin/nlm`

강의 내용 조회 시:
```bash
~/.local/bin/nlm notebook query 5ad76bf3-0328-4f8d-8fca-bbadc52c3b23 "질문"
```

## 광고 계산기

광고 계산기는 별도 스킬 `ads-manager-app`에서 담당. 사용자가 계산기 필요한 상황(예: ROAS/CPC 계산, 입찰가 시뮬레이션)이면 `/ads-manager-app` 실행을 안내해.

## 사용 흐름

사용자가 광고 관련 질문을 하면:
1. 내 생각(광고 원칙 기반 — `scripts/skill.md` 참조)을 먼저 답변
2. NotebookLM에 같은 질문 조회해서 강의 내용과 비교
3. 계산기가 필요한 상황이면 `/ads-manager-app`으로 안내

## CDP 광고 데이터 조회

### 계정 정보
| 별칭 | advertiserId | vendorId | 프로필 |
|------|-------------|----------|--------|
| redcombo | 469680 | A01627828 | coupang_ads (port 9341) |
| goya | - | - | port 9341 전환 필요 |

### redcombo 조회 방법
port **9341** `coupang_ads` 프로필 전용. MCP: `chrome-devtools-9341`.
열기: `/chrome-cdp -coupang-ads` (CC 재시작 후 사용 가능).
세션 만료 시 해당 Chrome 창에서 수동 재로그인 후 스크립트 재실행.

크리덴셜: `~/Documents/claude_skills/.env`
- `COUPANG_ADS_REDCOMBO_ID` / `COUPANG_ADS_REDCOMBO_PW`
- `COUPANG_ADS_GOYA_ID` / `COUPANG_ADS_GOYA_PW`

```python
import json, websocket, time, urllib.request, urllib.parse
import os

ENV_PATH = os.path.expanduser("~/Documents/claude_skills/.env")
TARGET_URL = "https://advertising.coupang.com/marketing/dashboard/sales"
LOGIN_URL = "https://login.coupang.com"

# .env 읽기
def load_env():
    env = {}
    with open(ENV_PATH) as f:
        for line in f:
            line = line.strip()
            if line and not line.startswith('#') and '=' in line:
                k, v = line.split('=', 1)
                env[k.strip()] = v.strip()
    return env

def cdp_eval(ws_url, expr):
    ws = websocket.WebSocket()
    ws.connect(ws_url)
    ws.send(json.dumps({"id":1,"method":"Runtime.evaluate","params":{"expression":expr,"returnByValue":True}}))
    r = json.loads(ws.recv()); ws.close()
    return r.get('result',{}).get('result',{}).get('value')

def cdp_send(ws_url, method, params={}):
    ws = websocket.WebSocket()
    ws.connect(ws_url)
    ws.send(json.dumps({"id":1,"method":method,"params":params}))
    ws.recv(); ws.close()

# 1. 탭 찾기 또는 생성
with urllib.request.urlopen('http://localhost:9341/json', timeout=5) as r:
    tabs = json.loads(r.read())
tab = next((t for t in tabs if 'advertising.coupang.com' in t.get('url','')), None)
if not tab:
    req = urllib.request.Request(f'http://localhost:9341/json/new?{urllib.parse.quote(TARGET_URL)}', method='PUT')
    with urllib.request.urlopen(req) as r:
        tab = json.loads(r.read())

WS = f"ws://localhost:9341/devtools/page/{tab['id']}"

# 2. 대시보드 이동
cdp_send(WS, "Page.navigate", {"url": TARGET_URL})
time.sleep(3)

# 3. 로그인 필요 여부 확인 및 자동 로그인
current_url = cdp_eval(WS, "location.href")
if 'login' in current_url or 'signin' in current_url:
    env = load_env()
    # ID/PW 입력
    cdp_eval(WS, f"document.querySelector('input[name=\"loginId\"], #loginId, input[type=\"text\"]').value = '{env['COUPANG_ADS_REDCOMBO_ID']}'")
    cdp_eval(WS, f"document.querySelector('input[name=\"password\"], #password, input[type=\"password\"]').value = '{env['COUPANG_ADS_REDCOMBO_PW']}'")
    cdp_eval(WS, "document.querySelector('button[type=\"submit\"], .login-btn, #loginBtn')?.click()")
    time.sleep(3)
    cdp_send(WS, "Page.navigate", {"url": TARGET_URL})
    time.sleep(3)

# 4. 데이터 읽기
data = cdp_eval(WS, "document.body.innerText")
```

### 주요 URL
- 광고 관리(캠페인 목록): `https://advertising.coupang.com/marketing/dashboard/sales/campaign`
- 캠페인 상세(상품별): `https://advertising.coupang.com/marketing/dashboard/pa/campaign/{캠페인ID}/group/{그룹ID}/product`
- 광고보고서: `https://advertising.coupang.com/marketing/report`

### 알려진 캠페인
| 캠페인명 | 캠페인ID | 그룹ID |
|---------|---------|--------|
| 캐리어바퀴_홍성_260415 | 104839190 | 205249311 |

### 페이지 구조 및 데이터 읽는 방법

**1단계: 캠페인 목록 페이지** (`/sales/campaign`)
`document.body.innerText`로 전체 텍스트 파싱.

날짜 필터 버튼 텍스트: `어제`, `최근 7일`, `이번달`
→ 날짜 필터 클릭:
```python
cdp_eval("([...document.querySelectorAll('button,span')].find(el => el.textContent.trim()==='어제'))?.click()")
time.sleep(2)
```

전체 성과 요약 파싱 (텍스트에서 추출):
```
집행 광고비 / 광고 전환 매출 / 클릭수 / 클릭률 / 광고 수익률 / 전환율
```

캠페인 목록 행: `캠페인명 | ON/OFF | 상태 | 예산 | 집행광고비 | 광고전환매출 | 전환율 | 클릭률 | 노출수 | 클릭수`

**2단계: 캠페인 클릭 → 상품별 상세**
캠페인 이름 `<a>` 태그를 JS click으로 클릭:
```python
cdp_eval("[...document.querySelectorAll('a')].find(a => a.textContent.includes('캠페인명'))?.click()")
time.sleep(3)
```
→ URL이 `/pa/campaign/{id}/group/{id}/product`로 이동

상품별 데이터: `상품명 | ID | 노출수 | 클릭수 | 클릭률 | 광고전환판매수 | 광고전환매출 | 전환율 | 집행광고비 | ROAS`

### 사용자 요청 패턴별 처리

**"레드콤보 광고 확인해줘" / "어제 성과 어때?"**
1. `/sales/campaign` 이동
2. 날짜 필터 클릭 (어제/최근7일/이번달)
3. `document.body.innerText` 파싱 → 전체 성과 요약 + 캠페인별 수치 추출
4. 요약 출력

**"캐리어바퀴_홍성_260415 캠페인 어제 성과 어때?"**
1. `/sales/campaign` 이동 → 날짜 필터 "어제" 클릭
2. 캠페인명 JS click → `/pa/campaign/104839190/group/205249311/product` 이동
3. 상품별 노출/클릭/전환/광고비/ROAS 파싱
4. 상품별 성과표 출력

## 핵심 광고 원칙

전체 상세 내용은 `~/.claude/skills/ads-manager/scripts/skill.md` 참조.

## 추가 도구

광고 히스토리 저장 (필요 시):
```bash
python3 ~/.claude/skills/ads-manager/scripts/save_ad_history.py
```

쿠팡 광고 공식 가이드 참고:
`~/.claude/skills/ads-manager/scripts/coupang_ads_official_guide.md`
