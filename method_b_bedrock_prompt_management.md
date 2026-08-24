# 방법 B: Bedrock Prompt Management 기반 버전 관리

> Codex 개발용 명세 — Amazon Bedrock Prompt Management를 활용한 프롬프트 버전/A/B 관리 구현

---

## 개요

Amazon Bedrock Prompt Management를 사용하여 프롬프트를 AWS 관리형으로 저장/버전 관리합니다.
프롬프트를 ARN으로 참조하므로, 프롬프트 변경 시 코드 수정이 불필요합니다.

---

## Bedrock Prompt Management 주요 기능

| 기능 | 설명 |
|---|---|
| **Prompt 생성/저장** | Console 또는 API로 프롬프트 생성 및 버전 관리 |
| **Variants (변형)** | 같은 프롬프트의 여러 변형을 만들어 비교 |
| **Side-by-side 비교** | Console에서 동일 입력 대비 변형 간 출력 비교 |
| **버전 태그** | Draft / LIVE 등 태그로 배포 상태 관리 |
| **ARN 참조** | 코드에서 ARN으로 프롬프트 참조 → 내용 변경해도 코드 수정 불필요 |
| **Flows/Agents 연동** | Bedrock Flows, Agents에서 프롬프트를 직접 참조 |

---

## 구현 코드

### 프롬프트 생성 (API)

```python
import boto3

bedrock_agent = boto3.client('bedrock-agent', region_name='us-east-1')

# 프롬프트 생성
response = bedrock_agent.create_prompt(
    name="tv-sports-system-prompt",
    description="TV Sports 큐레이션 에이전트 시스템 프롬프트",
    variants=[
        {
            "name": "v1",
            "templateType": "TEXT",
            "templateConfiguration": {
                "text": {
                    "text": "당신은 TV Sports 콘텐츠 큐레이션 전문가입니다...",
                    "inputVariables": [
                        {"name": "max_items"},
                        {"name": "channel_id"}
                    ]
                }
            },
            "modelId": "anthropic.claude-sonnet-4-6-20260620-v1:0",
            "inferenceConfiguration": {
                "text": {
                    "temperature": 0.3,
                    "maxTokens": 4096
                }
            }
        }
    ],
    defaultVariant="v1"
)

prompt_id = response['id']
prompt_arn = response['arn']
print(f"Prompt ARN: {prompt_arn}")
```

### 프롬프트 버전 생성

```python
# 기존 프롬프트에 새 버전 생성
response = bedrock_agent.create_prompt_version(
    promptIdentifier=prompt_id,
    description="v2: 검색 전략 개선 + 종목 분산 규칙 강화"
)

version = response['version']
print(f"Created version: {version}")
# ARN: arn:aws:bedrock:us-east-1:123456:prompt/tv-sports-system-prompt:2
```

### Variant 추가 (A/B 테스트용)

```python
# 기존 프롬프트에 새 변형 추가
response = bedrock_agent.update_prompt(
    promptIdentifier=prompt_id,
    variants=[
        {
            "name": "v1_original",
            "templateType": "TEXT",
            "templateConfiguration": {
                "text": {
                    "text": "기존 프롬프트 내용..."
                }
            },
            "modelId": "anthropic.claude-sonnet-4-6-20260620-v1:0"
        },
        {
            "name": "v2_improved",
            "templateType": "TEXT",
            "templateConfiguration": {
                "text": {
                    "text": "개선된 프롬프트 내용..."
                }
            },
            "modelId": "anthropic.claude-sonnet-4-6-20260620-v1:0"
        }
    ],
    defaultVariant="v1_original"  # 아직 기존 버전이 기본
)
```

### 프롬프트 호출 (ARN 참조)

```python
bedrock_runtime = boto3.client('bedrock-runtime', region_name='us-east-1')

# 버전 명시 호출
response = bedrock_runtime.converse(
    modelId="anthropic.claude-sonnet-4-6-20260620-v1:0",
    messages=[{"role": "user", "content": [{"text": "오늘 핫한 경기 3개 선정해줘"}]}],
    promptVariables={
        "max_items": {"text": "3"},
        "channel_id": {"text": "UC_xxxxx"}
    },
    # 프롬프트 ARN으로 참조 (버전 명시)
    additionalModelRequestFields={
        "promptArn": "arn:aws:bedrock:us-east-1:123456:prompt/tv-sports-system-prompt:2"
    }
)
```

### A/B 비교 (코드 레벨)

```python
def compare_variants(prompt_id: str, variant_a: str, variant_b: str, test_input: dict) -> dict:
    """두 변형의 출력을 비교"""

    # Variant A 실행
    response_a = bedrock_runtime.converse(
        modelId="anthropic.claude-sonnet-4-6-20260620-v1:0",
        messages=test_input['messages'],
        additionalModelRequestFields={
            "promptArn": f"arn:aws:bedrock:us-east-1:123456:prompt/{prompt_id}",
            "promptVariant": variant_a
        }
    )

    # Variant B 실행
    response_b = bedrock_runtime.converse(
        modelId="anthropic.claude-sonnet-4-6-20260620-v1:0",
        messages=test_input['messages'],
        additionalModelRequestFields={
            "promptArn": f"arn:aws:bedrock:us-east-1:123456:prompt/{prompt_id}",
            "promptVariant": variant_b
        }
    )

    # 골든셋으로 양쪽 평가
    score_a = evaluate_output(response_a, test_input['golden_rules'])
    score_b = evaluate_output(response_b, test_input['golden_rules'])

    return {
        "variant_a": {"name": variant_a, "score": score_a},
        "variant_b": {"name": variant_b, "score": score_b},
        "winner": variant_a if score_a >= score_b else variant_b
    }
```

### 프로덕션 배포 (defaultVariant 전환)

```python
def promote_variant(prompt_id: str, winning_variant: str):
    """A/B 테스트 승리 변형을 기본으로 설정"""
    bedrock_agent.update_prompt(
        promptIdentifier=prompt_id,
        defaultVariant=winning_variant
    )
    # 새 버전으로 확정
    bedrock_agent.create_prompt_version(
        promptIdentifier=prompt_id,
        description=f"Production: {winning_variant} promoted after A/B test"
    )
```

---

## 워크플로우

```
1. Console 또는 API로 새 Variant 생성
    │
    ▼
2. Console에서 Side-by-side 비교 (수동 확인)
    또는
   코드로 골든셋 A/B 비교 (자동)
    │
    ├── 기존이 나음 → 새 Variant 삭제
    │
    ▼ 새 Variant 우수
3. defaultVariant를 새 것으로 전환
    │
    ▼
4. create_prompt_version으로 확정 (이력 생성)
    │
    ▼
5. 에이전트가 ARN 참조 → 자동으로 새 버전 사용
   (코드 변경 불필요!)
```

---

## Console에서 A/B 테스트 (수동)

1. Bedrock Console → **Prompt Management** → 프롬프트 선택
2. **Variants** 탭 → 변형 2개 확인
3. **Test** → 같은 입력으로 두 변형 실행
4. **Side-by-side** 출력 비교
5. 우수한 변형을 **Default**로 설정
6. **Create Version** → 프로덕션 확정

---

## 드리프트 체크

```python
def run_drift_check_bedrock(prompt_id: str, version: int, golden_set: dict) -> float:
    """Bedrock 프롬프트의 골든셋 점수 확인"""
    results = []
    for test_case in golden_set['test_cases']:
        response = bedrock_runtime.converse(
            modelId="anthropic.claude-sonnet-4-6-20260620-v1:0",
            messages=test_case['messages'],
            additionalModelRequestFields={
                "promptArn": f"arn:aws:bedrock:us-east-1:123456:prompt/{prompt_id}:{version}"
            }
        )
        score = evaluate_output(response, test_case['then']['validation_rules'])
        results.append(score)

    avg_score = sum(results) / len(results)
    if avg_score < 0.8:
        alert(f"⚠️ 드리프트! prompt/{prompt_id}:{version} = {avg_score:.1%}")
    return avg_score
```

---

## 프롬프트 구조 (Bedrock에 등록할 것)

| 프롬프트 이름 | 용도 | 변수 |
|---|---|---|
| `tv-sports-system-prompt` | 시스템 프롬프트 (공통 규칙) | - |
| `tv-sports-task-masthead` | Masthead 태스크 | `max_items` |
| `tv-sports-task-team-row` | 팀 Row 태스크 | `team_name`, `channel_id` |

---

## 장점 / 한계

| 장점 | 한계 |
|---|---|
| Console UI로 편리한 편집/비교 | 커스텀 평가 로직은 코드 필요 |
| ARN 참조 → 코드 변경 없이 프롬프트 교체 | DB 기반 대비 유연성 낮음 (variables 제한) |
| Bedrock Flows/Agents와 네이티브 연동 | 현재 LangGraph에서 직접 ARN 참조 방식이 제한적 |
| 버전 이력 자동 관리 | 세밀한 조건부 로드(시간대별 등) 어려움 |
| Side-by-side 비교 UI 기본 제공 | 골든셋 자동 평가는 별도 코드 필요 |

---

## 전환 시점 (방법 A → B)

방법 B로 전환하기 좋은 시점:
- LangGraph 워크플로우가 안정화된 후
- Bedrock Flows 또는 Agents로 마이그레이션할 때
- 팀이 커져서 Console UI 기반 협업이 필요할 때
- 프롬프트 수가 많아져 DB 관리가 번거로울 때

---

## 참고 문서

- [Amazon Bedrock Prompt Management](https://aws.amazon.com/bedrock/prompt-management/)
- [CreatePrompt API](https://docs.aws.amazon.com/bedrock/latest/APIReference/API_agent_CreatePrompt.html)
- [CreatePromptVersion API](https://docs.aws.amazon.com/bedrock/latest/APIReference/API_agent_CreatePromptVersion.html)
