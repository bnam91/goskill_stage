---
name: nlm
description: NotebookLM 노트북을 이름(alias)으로 빠르게 열거나 질문하는 스킬이야. 사용자가 "/nlm qna_goya", "/nlm sku_stepred 질문" 등을 말할 때 실행해.
---

NotebookLM 노트북을 이름(alias)으로 빠르게 열거나 질문하는 스킬이야.
사용자가 "/nlm qna_goya", "/nlm sku_stepred 질문" 등을 말할 때 실행해.

> **🚀 처음 사용하는 환경이라면**: 같은 폴더의 `heyclaude.md`를 따라 환경 세팅을 먼저 진행. nlm CLI 설치 + Google OAuth 첫 인증 + 본인 alias 등록이 필요해. (아래 "참고 - 등록된 alias" 표는 본인 환경 예시. 직원은 본인 노트북에 맞게 alias 직접 등록해서 사용.)

## 인수 파싱

ARGUMENTS에서 첫 번째 단어를 커맨드 또는 alias로 파싱해.

예시:
- `/nlm qna_goya` → alias=qna_goya, 질문=없음 → 브라우저 열기
- `/nlm qna_goya 오늘 날씨는?` → alias=qna_goya, 질문="오늘 날씨는?" → CLI 질문
- `/nlm sync` → 전체 노트북 sync
- `/nlm sync qna_goya` → 특정 노트북 sync
- `/nlm` (인수 없음) → alias 목록 보여주고 종료

## 커맨드: sync

첫 번째 단어가 `sync`이면 소스 동기화 실행.

### `/nlm sync` (전체)
등록된 alias 전체를 순서대로 sync:
```bash
nlm source stale qna_goya
nlm source sync qna_goya --confirm

nlm source stale sku_stepred
nlm source sync sku_stepred --confirm
```
stale 소스가 없으면 "최신 상태예요" 출력하고 스킵.

### `/nlm sync <alias>` (특정 노트북)
```bash
nlm source stale <alias>
nlm source sync <alias> --confirm
```
결과 요약해서 출력. 예: "qna_goya 소스 1개 동기화 완료"

인수가 없으면:
```bash
nlm alias list
```
출력 후 "어떤 노트북 열까요?" 안내.

## alias → notebook ID 조회

```bash
nlm alias get <alias>
```
결과에서 UUID 추출. 실패하면 "등록된 alias가 없어요. `nlm alias set <이름> <id>`로 등록해주세요." 안내.

## 질문이 없는 경우 → 브라우저로 열기

1. CDP 상태 확인:
```bash
curl -s http://localhost:9222/json/version >/dev/null 2>&1 && echo "RUNNING" || echo "NOT_RUNNING"
```

2. **RUNNING** → MCP chrome-devtools로 바로 이동:
```
mcp__chrome-devtools__navigate_page(type="url", url="https://notebooklm.google.com/notebook/<UUID>")
mcp__chrome-devtools__take_screenshot()
```
스크린샷 보여주고 완료.

3. **NOT_RUNNING** → `/chrome-cdp` 스킬 실행 후 위 단계 진행.

## 질문이 있는 경우 → CLI로 질문

```bash
nlm notebook query <alias> "<질문>"
```
결과의 `answer` 필드만 추출해서 보기 좋게 출력.

## 참고 - 등록된 alias

| alias | 노트북 |
|-------|--------|
| qna_goya | 고야 QnA |
| sku_stepred | SKU Stepred |
| golden_formula | The Golden Formula |
| 잘사는김대리 | 회의_잘사는김대리 |
