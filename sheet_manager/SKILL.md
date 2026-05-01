---
name: sheet_manager
description: Google Sheets를 읽고/쓰고/수정하는 스킬. 사용자가 "시트 읽어줘", "구글시트에 써줘", "탭 목록", "스프레드시트 행 추가", "시트 범위 삭제", "구글 시트 업데이트" 등을 말할 때 실행해.
---

Google Sheets를 읽고, 쓰고, 수정하는 스킬이야.
스크립트 경로: ~/.claude/skills/sheet_manager/scripts/sheet_manager.py

> **🚀 처음 사용하는 환경이라면**: 같은 폴더의 `heyclaude.md`를 따라 환경 세팅을 먼저 진행. 사전 요구사항(module_auth, module_api_key/.env, Google API 패키지) 중 하나라도 빠져있으면 자동으로 거기로 이동. (`scripts/sheet_manager.py`는 이 스킬 폴더에 같이 들어있어서 별도 설치 불필요.)

## 사용 가능한 기능

### 1. 탭 목록 확인
```bash
python3 ~/.claude/skills/sheet_manager/scripts/sheet_manager.py tabs <spreadsheet_id>
```

### 2. 시트 읽기
```bash
# 전체 읽기
python3 ~/.claude/skills/sheet_manager/scripts/sheet_manager.py read <spreadsheet_id> --tab <탭이름>

# 범위 지정 읽기
python3 ~/.claude/skills/sheet_manager/scripts/sheet_manager.py read <spreadsheet_id> --tab <탭이름> --range A1:D10
```

### 3. 값 덮어쓰기
```bash
python3 ~/.claude/skills/sheet_manager/scripts/sheet_manager.py write <spreadsheet_id> --tab <탭이름> --range A2:C2 --values '[["값1","값2","값3"]]'
```

### 4. 마지막 행에 추가
```bash
python3 ~/.claude/skills/sheet_manager/scripts/sheet_manager.py append <spreadsheet_id> --tab <탭이름> --values '[["값1","값2","값3"]]'
```

### 5. 범위 삭제
```bash
python3 ~/.claude/skills/sheet_manager/scripts/sheet_manager.py clear <spreadsheet_id> --tab <탭이름> --range A2:D10
```

## 사용 방법

사용자의 요청을 분석하여 적절한 명령을 Bash 도구로 실행해줘.

- "시트 읽어줘" → read 실행
- "값 입력해줘 / 수정해줘" → write 실행
- "행 추가해줘" → append 실행
- "탭 목록 보여줘" → tabs 실행
- "삭제해줘" → clear 실행

스프레드시트 ID는 URL에서 추출:
`https://docs.google.com/spreadsheets/d/<여기가_ID>/edit`

결과는 한국어로 정리해서 보여줘.
