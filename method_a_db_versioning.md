# 방법 A: DB 기반 프롬프트 버전 관리

> Codex 개발용 명세 — Aurora MySQL `prompt_versions` 테이블로 프롬프트 버전을 관리하는 구현

---

## 개요

프롬프트를 DB 테이블에 저장하고, `is_active` 플래그로 활성 버전을 관리합니다.
에이전트 실행 시 활성 버전을 로드하여 LLM에 주입합니다.

---

## 테이블 스키마

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
    performance_score DECIMAL(5,2) NULL COMMENT '골든셋 대비 점수 (0.00~1.00)',
    UNIQUE KEY uk_type_version (prompt_type, version)
);
```

### 컬럼 설명

| 컬럼 | 설명 |
|---|---|
| `prompt_type` | 프롬프트 종류 (system / task_masthead / task_team_row) |
| `version` | 해당 타입 내 버전 번호 (1, 2, 3...) |
| `content` | 프롬프트 본문 (마크다운 텍스트) |
| `variables` | 템플릿 변수 정의 (예: `{"max_items": 3, "channel_id": "xxx"}`) |
| `is_active` | 현재 활성 버전인지 (타입당 1개만 TRUE) |
| `performance_score` | 골든셋 평가 점수 (0.80 이상이면 합격) |

---

## 구현 코드

### 프롬프트 로드

```python
def load_prompt(prompt_type: str, db_connection) -> str:
    """DB에서 활성 프롬프트 로드"""
    query = """
        SELECT content, variables FROM prompt_versions
        WHERE prompt_type = %s AND is_active = TRUE
        ORDER BY version DESC LIMIT 1
    """
    result = db_connection.execute(query, (prompt_type,))
    if not result:
        raise ValueError(f"No active prompt found for type: {prompt_type}")
    return result['content']
```

### 새 버전 등록

```python
def register_new_version(prompt_type: str, content: str, created_by: str, db_connection) -> int:
    """새 프롬프트 버전 등록 (비활성 상태로)"""
    # 현재 최대 버전 조회
    max_version = db_connection.execute(
        "SELECT MAX(version) as v FROM prompt_versions WHERE prompt_type = %s",
        (prompt_type,)
    )['v'] or 0

    new_version = max_version + 1

    db_connection.execute(
        """INSERT INTO prompt_versions (prompt_type, version, content, is_active, created_by)
           VALUES (%s, %s, %s, FALSE, %s)""",
        (prompt_type, new_version, content, created_by)
    )
    return new_version
```

### 골든셋 평가 후 점수 기록

```python
def record_score(prompt_type: str, version: int, score: float, db_connection):
    """골든셋 평가 점수 기록"""
    db_connection.execute(
        """UPDATE prompt_versions
           SET performance_score = %s
           WHERE prompt_type = %s AND version = %s""",
        (score, prompt_type, version)
    )
```

### 버전 활성화 (A/B 비교 후)

```python
def activate_version(prompt_type: str, version: int, approved_by: str, db_connection):
    """새 버전 활성화 + 기존 비활성화"""
    # 기존 활성 버전 비활성화
    db_connection.execute(
        "UPDATE prompt_versions SET is_active = FALSE WHERE prompt_type = %s AND is_active = TRUE",
        (prompt_type,)
    )
    # 새 버전 활성화
    db_connection.execute(
        """UPDATE prompt_versions
           SET is_active = TRUE, approved_at = NOW(), approved_by = %s
           WHERE prompt_type = %s AND version = %s""",
        (approved_by, prompt_type, version)
    )
```

### A/B 비교

```python
def compare_versions(prompt_type: str, v1: int, v2: int, golden_set: dict, db_connection) -> dict:
    """두 버전을 골든셋으로 비교"""
    content_v1 = db_connection.execute(
        "SELECT content FROM prompt_versions WHERE prompt_type = %s AND version = %s",
        (prompt_type, v1)
    )['content']

    content_v2 = db_connection.execute(
        "SELECT content FROM prompt_versions WHERE prompt_type = %s AND version = %s",
        (prompt_type, v2)
    )['content']

    score_v1 = run_golden_set_evaluation(content_v1, golden_set)
    score_v2 = run_golden_set_evaluation(content_v2, golden_set)

    return {
        "v1": {"version": v1, "score": score_v1},
        "v2": {"version": v2, "score": score_v2},
        "winner": v1 if score_v1 >= score_v2 else v2
    }
```

---

## 워크플로우

```
1. 새 프롬프트 작성 → register_new_version() (is_active=FALSE)
    │
    ▼
2. 골든셋 평가 실행 → record_score()
    │
    ├── 80% 미만 → 수정 후 1번으로
    │
    ▼ 80% 이상
3. A/B 비교 → compare_versions() (기존 활성 vs 새 버전)
    │
    ├── 기존이 나음 → 새 버전 보류
    │
    ▼ 새 버전 우수
4. 활성화 → activate_version()
    │
    ▼
5. 다음 배치 실행 시 새 버전 자동 로드
```

---

## 드리프트 체크

```python
def run_drift_check(prompt_type: str, golden_set: dict, db_connection) -> float:
    """현재 활성 프롬프트의 골든셋 점수 확인"""
    content = load_prompt(prompt_type, db_connection)
    score = run_golden_set_evaluation(content, golden_set)

    if score < 0.8:
        alert(f"⚠️ 드리프트 감지! {prompt_type} 점수: {score:.1%}")

    return score
```

실행 주기:
- 프롬프트 변경 시 → 즉시
- 정기 → 주 1회 (cron)
- 모델 버전 변경 시 → 즉시

---

## 장점 / 한계

| 장점 | 한계 |
|---|---|
| 코드와 프롬프트가 같은 DB에서 관리 | Console UI 없음 (코드/SQL로만 관리) |
| 이력 + 점수가 한 테이블에 | Side-by-side 비교 UI 없음 |
| LangGraph 워크플로우와 자연스럽게 연동 | 멀티 사용자 동시 편집 제어 필요 |
| 세밀한 커스텀 가능 (variables, 조건부 로드 등) | Bedrock 네이티브 기능(Flows 등)과 연동 불가 |
