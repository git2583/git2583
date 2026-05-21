---
name: "🤖 [AI] 자동화 에이전트 가동 지시"
about: >
  Make.com 파이프라인 또는 승인된 Webhook이 감지하여 AI 에이전트를 구동하기 위한
  머신 가독형(Machine-readable) 업무 명세서입니다.
  이 템플릿은 사람이 임의로 수정하지 않으며, 파싱 키(Key)의 명칭과 구조를 변경할 경우
  Make.com 시나리오가 파손될 수 있습니다.
title: "[AI-RUN] {AGENT_TYPE} | {TASK_NAME} | {YYYYMMDD}"
labels: ["AI-Task", "Ready-to-Run"]
assignees: ""
---

<!--
════════════════════════════════════════════════════════
  ⚠️  MACHINE-READABLE ZONE — DO NOT EDIT MANUALLY  ⚠️
  이 Issue는 Make.com 파이프라인 또는 승인된 API에 의해 자동 생성됩니다.
  사람이 수동으로 편집할 경우 파싱 오류 및 에이전트 오작동이 발생합니다.
  수동 수정이 필요한 경우, 반드시 해당 섹션 아래 [HUMAN_OVERRIDE] 블록을 사용하십시오.
════════════════════════════════════════════════════════
-->

---

## [BLOCK_1: SYSTEM_METADATA]
> **파싱 규칙:** Make.com 시나리오가 이 블록을 최초 감지하여 라우팅 및 에이전트 선택에 사용합니다.
> 모든 값은 지정된 Enum 내에서만 입력됩니다. 범위 외 값 입력 시 파이프라인이 `ERROR_UNROUTABLE`로 분류합니다.

```yaml
SCHEMA_VERSION: "2.1"
ISSUE_UUID: "{UUID_v4}"                          # Make.com이 자동 생성하는 고유 식별자
CREATED_AT: "{ISO_8601}"                         # 예: 2026-05-21T09:00:00+09:00
EXPIRES_AT: "{ISO_8601}"                         # 미처리 시 자동 에스컬레이션 기준 시각

TARGET_AGENT: "{ENUM: Operations-Agent | Visual-Agent | Content-Agent | Data-Agent}"
AGENT_VERSION: "{예: v2.3.1}"                   # 구동할 에이전트 프롬프트 버전 (SSoT Prompts 디렉토리 기준)

TRIGGER_SOURCE: "{ENUM: Make.com_Scheduler | Webhook_Inbound | Manual_Dispatch | API_External}"
TRIGGER_ID: "{Make.com 시나리오 ID 또는 Webhook Event ID}"
PRIORITY: "{ENUM: P0_Critical | P1_High | P2_Normal | P3_Low}"
RETRY_POLICY: "{ENUM: No_Retry | Retry_3x | Retry_Exponential}"
MAX_EXECUTION_TIME_SEC: 120

DATA_PIPELINE_STATUS: "{ENUM: READY | PENDING_VALIDATION | BLOCKED}"
PIPELINE_HEALTH_CHECK_URL: "{Make.com 시나리오 모니터링 URL}"
```

---

## [BLOCK_2: REQUIRED_CONTEXT — SSoT 주입 명세]
> **파싱 규칙:** Make.com이 아래 경로를 기반으로 GitHub API를 호출하여 파일 원문을 읽고,
> LLM의 `system` 메시지 및 상위 컨텍스트(Context)로 순서대로 주입합니다.
> `INJECTION_ORDER` 값이 낮을수록 System Prompt에 먼저 적재됩니다.

```yaml
SSoT_REFERENCES:
  - INJECTION_ORDER: 1
    TYPE: "Philosophy"
    FILE_PATH: "Philosophy/decision-matrix.md"
    REQUIRED: true
    CACHE_TTL_MIN: 60                            # 동일 에이전트 반복 호출 시 캐시 허용 시간

  - INJECTION_ORDER: 2
    TYPE: "SOP"
    FILE_PATH: "SOP/operations-sop.md"           # ← 실행 태스크에 맞게 교체
    REQUIRED: true
    CACHE_TTL_MIN: 30

  - INJECTION_ORDER: 3
    TYPE: "Prompt"
    FILE_PATH: "Prompts/operations-agent.md"     # ← TARGET_AGENT와 반드시 일치해야 함
    REQUIRED: true
    CACHE_TTL_MIN: 0                             # Prompt는 항상 최신본 사용

  - INJECTION_ORDER: 4
    TYPE: "Reference_Data"
    FILE_PATH: "Data/reference-tables.json"      # 선택적 참조 데이터 (없으면 SKIP)
    REQUIRED: false
    CACHE_TTL_MIN: 120

CONTEXT_WINDOW_BUDGET:
  SYSTEM_PROMPT_MAX_TOKENS: 4000
  CONTEXT_MAX_TOKENS: 8000
  RESPONSE_MAX_TOKENS: 2000
  OVERFLOW_POLICY: "{ENUM: Truncate_Tail | Summarize | Error_Escalate}"
```

---

## [BLOCK_3: INPUT_DATA — 가변형 원시 데이터]
> **파싱 규칙:** 이 블록의 JSON이 LLM `user` 메시지의 핵심 입력값으로 주입됩니다.
> `raw_payload`는 반드시 이스케이프 처리된 문자열이어야 합니다.
> 민감 정보(PII)가 포함된 경우 `pii_detected: true`로 표기하며,
> Make.com 파이프라인이 마스킹 처리 후 적재합니다.

```json
{
  "task_id": "TASK-{YYYYMMDD}-{SEQ}",
  "customer_id": "CUST-2026-0521",
  "order_number": "ORD-994102",
  "issue_category": "배송_지연_문의",
  "issue_subcategory": "배송_추적_불가",
  "channel_source": "{ENUM: email | chat | webhook | internal}",
  "language_detected": "ko",
  "pii_detected": true,
  "pii_fields_masked": ["customer_name", "phone_number"],
  "raw_payload": "5일 전에 주문한 상품이 아직도 안 왔어요. 언제 오나요? 확인 좀 해주세요.",
  "attached_assets": [],
  "session_history_ref": null
}
```

---

## [BLOCK_4: OUTPUT_SPECIFICATION — 결과물 명세]
> **파싱 규칙:** AI 에이전트는 반드시 이 블록에 정의된 스키마를 준수하여 응답을 생성해야 합니다.
> 형식 불일치 시 Make.com 후처리 모듈이 `OUTPUT_VALIDATION_FAIL`로 분류하고 재시도합니다.

```yaml
RESPONSE_FORMAT: "JSON"
RESPONSE_SCHEMA:
  task_id: "string (INPUT의 task_id와 동일)"
  agent_id: "string"
  status: "{ENUM: SUCCESS | PARTIAL | FAILED | ESCALATE_TO_HUMAN}"
  confidence_score: "float (0.0 ~ 1.0)"
  action_taken: "string (수행한 처리 내용 한 줄 요약)"
  output_payload: "object (태스크 유형에 따라 가변)"
  escalation_reason: "string | null (status가 ESCALATE_TO_HUMAN일 때 필수)"
  recommended_next_action: "string | null"
  tokens_used: "integer"
  execution_time_ms: "integer"

OUTPUT_DESTINATION:
  PRIMARY: "{ENUM: GitHub_Issue_Comment | Notion_DB | Google_Sheets | Webhook_Outbound}"
  SECONDARY: "Slack_Notification"
  ARCHIVE: "Google_Drive/AI-Logs/{YYYYMM}/{TASK_ID}.json"
```

---

## [BLOCK_5: ESCALATION_POLICY — 에스컬레이션 규칙]
> AI 에이전트가 자체 판단으로 처리를 완료할 수 없는 경우의 분기 규칙을 정의합니다.
> 이 블록은 Make.com 시나리오의 조건 분기(Router)와 직접 연동됩니다.

```yaml
ESCALATION_RULES:
  - CONDITION: "confidence_score < 0.70"
    ACTION: "ESCALATE_TO_HUMAN"
    NOTIFY: ["@ops-lead"]
    ISSUE_LABEL_ADD: ["Needs-Human-Review", "AI-Low-Confidence"]

  - CONDITION: "status == FAILED AND retry_count >= MAX_RETRY"
    ACTION: "CREATE_HUMAN_TASK_ISSUE"
    TEMPLATE: ".github/ISSUE_TEMPLATE/task-assignment.md"
    NOTIFY: ["@ops-lead", "@system-admin"]
    ISSUE_LABEL_ADD: ["Human-Task", "AI-Escalation"]

  - CONDITION: "pii_detected == true AND pii_masking_status == FAILED"
    ACTION: "ABORT_AND_ALERT"
    NOTIFY: ["@privacy-officer"]
    ISSUE_LABEL_ADD: ["Privacy-Risk", "Immediate-Review"]

  - CONDITION: "execution_time_ms > MAX_EXECUTION_TIME_SEC * 1000"
    ACTION: "TIMEOUT_ESCALATE"
    NOTIFY: ["@system-admin"]
```

---

## [BLOCK_6: HUMAN_OVERRIDE — 수동 개입 전용 구역]
> ⚠️ 사람이 이 Issue에 개입해야 할 경우, **반드시 이 블록 아래에만 내용을 추가**하십시오.
> BLOCK_1 ~ BLOCK_5는 어떠한 경우에도 수동 수정을 금지합니다.

**개입 사유:**
- [ ] AI 에스컬레이션 수신 후 판단 개입
- [ ] 파이프라인 오류로 인한 수동 재처리
- [ ] 예외 케이스 직접 처리
- [ ] 기타: ______

**개입 내용 및 처리 결과:**
> (여기에 자유 형식으로 기록)

**후속 SSoT 업데이트 필요 여부:**
- [ ] 예 → 관련 PR # 을 생성하고 이 Issue에 연결
- [ ] 아니오 → 사유: ______