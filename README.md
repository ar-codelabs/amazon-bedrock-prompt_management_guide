# amazon-bedrock-prompt_management_guide
amazon-bedrock-prompt_management_guide

# 프롬프트 관리 가이드

> 시스템 프롬프트, 골든룰, 버전 관리, 운영 플로우 정리

---

## 1. 시스템 프롬프트 (System Prompt)

### 역할

시스템 프롬프트는 **에이전트의 "성격과 규칙"을 정의**합니다. 모든 태스크 실행 시 공통으로 적용되며, 태스크 프롬프트가 바뀌어도 이 규칙은 유지됩니다.

### 포함 내용

| 섹션 | 설명 | 예시 |
|---|---|---|
| **Role** | 에이전트의 역할 정의 | "콘텐츠 큐레이션 전문가" |
| **Data Source Priority** | 검색 우선순위 (1순위 OpenDB → 2순위 웹검색 → 3순위 YouTube) | OpenDB에서 충분하면 웹검색 안 함 |
| **DB Schema** | OpenDB DB 테이블 구조 (matches, players, teams, content, schedules) | 에이전트가 SQL 생성 시 참조 |
| **날짜/시간 규칙** | UTC 기준 정렬, timezone 표기, 복수 경기 처리 | "같은 팀 복수 경기 → 각각 별도 아이템" |
| **검색 전략** | Masthead / 팀 Row 각각의 검색 순서 | Step 1~6 순서 고정 |
| **SQL 규칙** | SELECT만, JOIN 활용, LIMIT 필수, JSON_EXTRACT 활용 | INSERT/UPDATE/DELETE 금지 |
| **Fallback Rules** | 데이터 부족 시 대응 규칙 | OpenDB 없으면 → source="web_search", match_id=null |
| **Output Rules** | JSON 형식, 텍스트 규칙(30자 제목, 80자 설명), 금지 사항 | 이모지 금지, 허구 정보 생성 금지 |

### 시스템 프롬프트 vs 태스크 프롬프트 분리 이유

```
시스템 프롬프트 (공통): "너는 누구고, 어떤 규칙을 지켜야 해"
    ↓ 매번 동일하게 적용
태스크 프롬프트 (개별): "이번엔 Masthead 만들어줘" / "이번엔 팀 Row 만들어줘"
    ↓ 태스크마다 다름
```

**장점:**
- 공통 규칙은 한 곳에서 관리 (중복 제거)
- 태스크 추가 시 시스템 프롬프트 수정 불필요
- 버전 관리가 독립적 (시스템 v3 + Masthead v2 + Row v1 조합 가능)

---

## 3. 런타임 검증 & 재시도 루프

프롬프트 실행 후 출력물을 자동으로 검증하고, 실패 시 재시도합니다:

```
LLM 호출 (Tool Use 루프)
    │
    ▼
출력 검증 (JSON 스키마 + 비즈니스 규칙)
    │
    ├── PASS → 초안 저장 → PM 알림
    │
    └── FAIL → 검증 실패 사유를 메시지에 추가 → 재호출 (최대 2회)
              │
              └── 2회 초과 → 그대로 저장 (수동 검토 표시)
```

### 검증 항목

| 검증 | 내용 |
|---|---|
| JSON 스키마 | 출력이 지정된 스키마에 맞는지 |
| 중복 체크 | 같은 match_id 중복 없는지 |
| 제목 길이 | ≤ 30자 |
| 설명 길이 | ≤ 80자 |
| 종목 분산 | 동일 종목 최대 2개 |
| match_id 존재 | OpenDB 출처면 match_id 필수 |
| timezone 포함 | 모든 경기에 timezone 표기 |

---

## 4. PM 검토/승인 플로우

에이전트는 "초안"만 생성합니다. 프로덕션 적용은 PM 승인 후:

```
에이전트 실행 (데일리 배치)
    │
    ▼
초안 생성 → Aurora MySQL 저장 (status: "review")
    │
    ▼
PM에게 알림 (SNS/Slack)
    │
    ▼
PM 검토
    ├── 승인 (status: "approved" → "published")
    ├── 수정 후 승인
    └── 반려 (status: "rejected")
    │
    ▼
프로덕션 적용
```

모든 액션(생성/검토/승인/반려/배포)은 `curation_history` 테이블에 기록됩니다.

---

## 5. 골든셋 (Golden Set)

### 골든셋이란?

**입력-기대출력 쌍**으로, 프롬프트의 품질을 측정하는 테스트 케이스입니다.

```
Given (입력): "이런 데이터가 있을 때" (mock 도구 응답 고정)
When (실행): "이 프롬프트를 실행하면"
Then (기대): "이런 결과가 나와야 한다" (검증 규칙 + 내용 힌트)
```

### 왜 필요한가?

| 상황 | 골든셋 없으면 | 골든셋 있으면 |
|---|---|---|
| 프롬프트 수정 | 기존 기능 깨졌는지 모름 | ✅ 회귀 테스트로 즉시 확인 |
| 모델 업데이트 | 품질 변했는지 모름 | ✅ 드리프트 감지 |
| A/B 테스트 | 어느 버전이 나은지 감으로 판단 | ✅ 정량적 비교 가능 |
| 새 팀원 합류 | "잘 되는 건 어떤 건데?" | ✅ 기대 동작 명확 |

### 골든셋 구조

```json
{
  "golden_set_version": "1.0",
  "prompt_type": "masthead",
  "test_cases": [
    {
      "id": "TC-MAST-001",
      "description": "NBA 2개 + NFL 1개 정상 케이스",
      "given": {
        "mode": "masthead",
        "conditions": {"max_items": 3},
        "mock_tool_responses": {
          "query_opendb": [...],
          "web_search": [...],
          "search_youtube_channel": [...]
        }
      },
      "when": "프롬프트 실행",
      "then": {
        "validation_rules": [
          {"rule": "items_count", "operator": "==", "value": 3},
          {"rule": "title_max_length", "operator": "<=", "value": 30},
          {"rule": "no_duplicate_match_id"},
          {"rule": "max_same_sport", "operator": "<=", "value": 2}
        ],
        "expected_content_hints": [
          "M001 (Lakers vs Celtics) 포함",
          "key_players에 LeBron 포함"
        ]
      }
    }
  ]
}
```

### 필수 테스트 시나리오

**Masthead:**

| ID | 시나리오 | 검증 포인트 |
|---|---|---|
| TC-MAST-001 | 정상: NBA 2 + NFL 1 | 3개 생성, 종목 분산 |
| TC-MAST-002 | 같은 팀 복수 경기 | 각각 별도 아이템, 시간순 |
| TC-MAST-003 | 시간대 다른 경기 혼재 | timezone 정확 표기 |
| TC-MAST-004 | OpenDB 부족 → 웹 폴백 | source="web_search", match_id=null |
| TC-MAST-005 | 동일 종목 3개+ 후보 | 최대 2개만 선정 |
| TC-MAST-006 | 선수 데이터 없음 | key_players=[], 에러 없음 |
| TC-MAST-007 | 모든 경기 postponed | 빈 items, 에러 없음 |
| TC-MAST-008 | max_items=1 | 정확히 1개만 |
| TC-MAST-009 | 웹 검색 결과 0건 | OpenDB만으로 생성 |
| TC-MAST-010 | completed 경기 score 포함 | score 정확히 포함 |

**팀 Row:**

| ID | 시나리오 | 검증 포인트 |
|---|---|---|
| TC-ROW-001 | 정상: 최근 경기 + 콘텐츠 | 5개+ 아이템, 다양한 type |
| TC-ROW-002 | 같은 팀 2일 연속 경기 | 각각 별도, match_time 구분 |
| TC-ROW-003 | 콘텐츠 5개 미만 | YouTube로 보충 |
| TC-ROW-004 | 팀 검색 결과 없음 | error JSON 반환 |
| TC-ROW-005 | 14일 내 경기 없음 | 30일로 확장 재조회 |
| TC-ROW-006 | YouTube 결과 0건 | YouTube 없이 정상 출력 |
| TC-ROW-007 | 선수 스탯 없는 경기 | key_players=[] |
| TC-ROW-008 | 동일 경기 콘텐츠 4개+ | 최대 3개만 포함 |
| TC-ROW-009 | EPL 팀 (timezone Europe/London) | timezone 정확 |
| TC-ROW-010 | 다중 Row 생성 (2개 팀) | 각 팀별 독립 Row |

### 평가 기준

| 기준 | 가중치 | 평가 방법 |
|---|---|---|
| 구조 정확성 | 30% | 자동 (JSON schema 검증) |
| 내용 정확성 | 30% | 자동 + 수동 |
| 비즈니스 규칙 | 20% | 자동 (중복, 개수, 분산) |
| 텍스트 품질 | 20% | 수동 (PM 평가) |

**합격:** 80% 이상 = PASS / 70~79% = WARNING / 70% 미만 = FAIL

### 드리프트 체크

동일 프롬프트 + 동일 입력으로 **시간이 지나면서 품질이 변하는지** 모니터링:

```python
def run_drift_check(prompt_version, golden_set):
    results = []
    for test_case in golden_set['test_cases']:
        output = run_agent_with_mocks(
            prompt_version=prompt_version,
            mock_responses=test_case['given']['mock_tool_responses']
        )
        score = evaluate(output, test_case['then'])
        results.append(score)

    avg_score = sum(results) / len(results)
    if avg_score < 0.8:
        alert(f"⚠️ 드리프트 감지! v{prompt_version} 점수: {avg_score:.1%}")
    return avg_score
```

실행 주기:
- 프롬프트 변경 시 → 즉시
- 정기 → 주 1회
- 모델 버전 변경 시 → 즉시

---

## 6. 버전 관리

### 방법 A: DB 기반 (현재 구현)

프롬프트를 Aurora MySQL `prompt_versions` 테이블에서 관리합니다.

#### 테이블 스키마

```sql
CREATE TABLE prompt_versions (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    prompt_type ENUM('system', 'task_masthead', 'task_team_row') NOT NULL,
    version INT NOT NULL,
    content TEXT NOT NULL,
    variables JSON COMMENT '템플릿 변수 목록 (예: max_items, channel_id)',
    is_active BOOLEAN DEFAULT FALSE,
    created_by VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    approved_at TIMESTAMP NULL,
    approved_by VARCHAR(100) NULL,
    performance_score DECIMAL(5,2) NULL COMMENT '골든셋 대비 점수',
    UNIQUE KEY uk_type_version (prompt_type, version)
);
```

#### 버전 관리 워크플로우

```
1. 새 프롬프트 작성 → INSERT (is_active=FALSE)
2. 골든셋 평가 실행 → performance_score 기록
3. 80% 이상 → 승인 (approved_at, approved_by 기록)
4. 기존 활성 버전 비활성화 → 새 버전 is_active=TRUE
5. 에이전트 실행 시 is_active=TRUE인 최신 버전 로드
```

#### A/B 비교 방법

```python
# 동일 입력(골든셋)으로 두 버전 출력 비교
score_v1 = run_drift_check(prompt_version=1, golden_set=golden_set)
score_v2 = run_drift_check(prompt_version=2, golden_set=golden_set)

print(f"v1: {score_v1:.1%}, v2: {score_v2:.1%}")
# → 높은 쪽을 is_active=TRUE로 전환
```

#### 프롬프트 로드 코드

```python
def load_prompt(prompt_type: str) -> str:
    """DB에서 활성 프롬프트 로드"""
    query = """
        SELECT content FROM prompt_versions
        WHERE prompt_type = %s AND is_active = TRUE
        ORDER BY version DESC LIMIT 1
    """
    result = db.execute(query, (prompt_type,))
    return result['content']
```

### 방법 B: Bedrock Prompt Management (향후 고도화 옵션)

현재 구현은 DB 기반이지만, 향후 **Amazon Bedrock Prompt Management**로 전환하여 추가 기능을 활용할 수 있습니다:

| 기능 | 설명 |
|---|---|
| Prompt 버전 관리 | Console에서 버전별 저장/이력 관리 |
| Prompt Variants | 같은 프롬프트의 여러 변형을 만들어 비교 |
| Side-by-side 비교 | Console에서 변형 간 출력 비교 |
| ARN 참조 | 프롬프트를 ARN으로 참조 → 코드 변경 없이 프롬프트 교체 |
| Flows/Agents 연동 | Bedrock Flows, Agents에서 프롬프트 직접 참조 |

```python
# 향후 Bedrock Prompt Management 전환 시:
response = bedrock.converse(
    modelId="anthropic.claude-sonnet-4-6-20260620-v1:0",
    promptArn="arn:aws:bedrock:us-east-1:123456:prompt/sports-masthead:2"
    # :2 = 버전 2. 또는 "LIVE" 태그로 최신 프로덕션 참조
)
```

**전환 시점:** LangGraph 워크플로우가 안정화되고, Bedrock Flows/Agents로 마이그레이션할 때 함께 전환 검토.

---

## 7. 전체 운영 흐름 요약

```
[프롬프트 수정]
    │
    ▼
[골든셋 자동 테스트] → FAIL → 수정
    │
    ▼ PASS (80%+)
[DB에 새 버전 INSERT + performance_score 기록]
    │
    ▼
[A/B 비교 (기존 활성 버전 vs 새 버전)]
    │
    ▼ 새 버전 우수
[새 버전 is_active=TRUE, 기존 비활성화]
    │
    ▼
[데일리 배치 실행 (EventBridge Scheduler)]
    │
    ▼
[에이전트: 활성 프롬프트 로드 → LLM 호출 → Tool Use → 출력]
    │
    ▼
[런타임 검증] → FAIL → 재시도 (최대 2회)
    │
    ▼ PASS
[초안 저장 (status: "review")]
    │
    ▼
[PM 알림 → PM 검토]
    ├── 승인 → 프로덕션 적용 (status: "published")
    └── 반려 → 재생성 또는 수동 수정
    │
    ▼
[주간 드리프트 체크] → 품질 저하 감지 시 알림
```

---

## 8. 프롬프트 관련 파일 구조

```
prompts/
├── system_prompt.md           # 시스템 프롬프트 (공통 규칙)
├── task_masthead.md           # Masthead 태스크 프롬프트
└── task_team_row.md           # 팀 Row 태스크 프롬프트

golden_set/
├── guide.md                   # 골든셋 작성 가이드
├── masthead_golden.json       # Masthead 골든셋 (테스트 케이스)
├── team_row_golden.json       # 팀 Row 골든셋
└── results/                   # 드리프트 체크 결과 이력
    └── YYYY-MM-DD_drift.json

schemas/
├── masthead_output.json       # Masthead 출력 JSON 스키마 (검증용)
└── team_row_output.json       # 팀 Row 출력 JSON 스키마
```

---

## 참고 문서

- [Amazon Bedrock Prompt Management](https://aws.amazon.com/bedrock/prompt-management/) — 향후 고도화 시 참고
- [Bedrock Converse API](https://docs.aws.amazon.com/bedrock/latest/userguide/conversation-inference-call.html)
