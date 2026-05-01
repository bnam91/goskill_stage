---
name: bank_manager
description: 구글 시트에 입금요청을 등록하는 스킬이야.
---

구글 시트에 입금요청을 등록하는 스킬이야.
스크립트 경로: ~/.claude/skills/bank_manager/scripts/payment_request.py

> **🚀 처음 사용하는 환경이라면**: 같은 폴더의 `heyclaude.md`를 따라 환경 세팅을 먼저 진행. 사전 요구사항(module_auth, module_api_key/.env, Google API 패키지) 외에도 본인 시트의 ID/탭 이름을 스크립트에 입력해야 동작해.

## 실제 이체 실행
입금요청 등록 후 실제 은행 이체는 **별도 자동화(예: Windows의 macro 스크립트, RPA)**로 실행. 이 스킬은 시트 등록까지만 담당. 사용자가 "이체 실행해줘", "실제로 입금해줘" 등을 요청하면 본인 환경의 이체 자동화로 안내해.

## 시트 정보 (본인이 채워야 할 부분)
- 스프레드시트 ID: `payment_request.py`의 `SPREADSHEET_ID` 변수
- 시트 탭 이름: `payment_request.py`의 `SHEET_NAME` 변수
- 열 구성 (LOCAL 시트 기준): E=항목, F=받는사람, I=계좌번호, J=주민/사업자번호, K=금액, P=상태(입금요청)
  → 본인 시트 열 구조가 다르면 스크립트 상단 `COL_*` 인덱스 조정 필요

## 자주쓰는 곳 (favorites.json)

자주 입금하는 거래처를 별칭으로 등록해두면 매번 계좌번호를 입력 안 해도 됨. 처음에는 빈 파일.

favorites.json 경로: `~/.claude/skills/bank_manager/scripts/favorites.json`

예시 (등록 명령으로 만들어짐):
```json
{
  "별칭1": {
    "item": "기본 항목명",
    "recipient": "실제 받는사람",
    "account": "은행 계좌번호 예금주",
    "business_id": ""
  }
}
```

## 입력 파싱 규칙

사용자 입력 예시:
- "거래처A에 관리비 381,430 입금 등록해줘"
- "홍길동에게 서비스비 500,000 입금 요청 등록"
- "ABC회사 광고비 1,200,000 등록해줘"

파싱:
- 받는사람/별칭(recipient): 금액 앞에 오는 사람/회사/별칭
- 항목(item): 받는사람 다음에 오는 항목명 (관리비, 서비스비, 급여, 광고비 등)
- 금액(amount): 숫자 + 콤마 조합. 그대로 전달.

**파싱이 불명확한 경우**: 사용자에게 확인 후 진행.

## Step 1: 기존 계좌정보 조회

```bash
python3 ~/.claude/skills/bank_manager/scripts/payment_request.py \
  --lookup --recipient [받는사람/별칭]
```

결과 형식:
- `FOUND|계좌번호|주민번호|항목템플릿|실제수취인|출처` → Step 2로 바로 진행
- `NOT_FOUND` → Step 1-1로 이동

FOUND일 때: 파이프(|)로 split해서 각 필드 추출.

### Step 1-1: 계좌정보 없을 때

사용자에게 요청:
> "**[받는사람]**의 등록된 계좌 정보가 없습니다. 계좌번호를 알려주시면 함께 등록할게요. (모르면 '없음'이라고 해주세요)"

## Step 2: 입금요청 등록

`--recipient`에는 항상 원래 사용자 입력 별칭을 전달 (스크립트가 favorites/시트에서 자동 매핑):

```bash
python3 ~/.claude/skills/bank_manager/scripts/payment_request.py \
  --recipient "[별칭 또는 이름]" \
  --item "[항목]" \
  --amount "[금액]"
```

계좌를 사용자가 직접 제공한 경우에만 --account, --business-id 추가.

## Step 3: 결과 보고

```
✅ 입금요청 등록 완료
  항목     : [항목]
  받는사람 : [이름]
  금액     : [금액]
  계좌번호 : [계좌번호]
  상태     : 입금요청
```

## 추가 기능

**입금요청 목록 조회** ("입금요청 목록", "입금 대기 목록" 요청 시):
```bash
python3 ~/.claude/skills/bank_manager/scripts/payment_request.py --list
```

**자주쓰는 곳 목록** ("자주쓰는 곳 보여줘"):
```bash
python3 ~/.claude/skills/bank_manager/scripts/payment_request.py --show-favorites
```

**자주쓰는 곳 새로 등록** ("XXX를 자주쓰는 곳에 등록해줘"):
```bash
python3 ~/.claude/skills/bank_manager/scripts/payment_request.py \
  --add-favorite \
  --alias "[별칭]" \
  --item "[항목템플릿]" \
  --recipient "[받는사람]" \
  --account "[계좌번호]" \
  --business-id "[주민/사업자번호]"  # 선택
```

## 주의사항
- 금액은 입력 그대로 전달 ("381,430" → "381,430")
- 별칭 조회는 favorites → 시트 E열 부분일치 → 시트 F열 정확일치 순
- 모든 결과는 한국어로 정리해서 보여줘
