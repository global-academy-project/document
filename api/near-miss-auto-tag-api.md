# 아차사고 자동 태깅 API 계약 초안

- 상태: 제안 초안, 구현 전 합의 필요
- 작성일: 2026-11-02
- 적용 범위: 웹 → Spring 실행 요청, Spring → `main-ai-model` 태그 추론
- 담당 영역: A(검색·5축 태깅), Spring 실행·저장 담당자, AI 에이전트 담당자

> 이 문서는 API 경계와 JSON 계약을 제안합니다. 현재 Spring의 자동 태깅 실행 경로와 `main-ai-model`의 `NEAR_MISS_TAG` 운영 라우트·처리기는 아직 구현되지 않았습니다. 경로와 필드는 담당자 합의 후 확정해야 합니다.

## 1. 책임 경계

| 구성 요소 | 책임 |
| --- | --- |
| 웹 | 사용자 동작으로 실행 요청, `runId` 상태 조회, 태그 후보·신뢰도 표시, 사용자 검토 및 수정 |
| Spring | 인증·현장 권한 확인, 최신 `lockVersion` 확인, 비동기 실행 생성, AI 호출, 결과 검증·저장, 실행 상태와 감사 이력 관리 |
| `main-ai-model` | 전달받은 보고 스냅샷과 허용 태그 코드로 분류 후보 생성. 업무 DB를 직접 읽거나 쓰지 않음 |

웹은 `main-ai-model`을 직접 호출하지 않습니다. 보고 등록과 작업 경고 생성은 자동 태깅 성공 여부와 분리합니다. AI 요청이 실패해도 이미 저장된 보고와 서버가 만든 경고는 유지합니다.

## 2. 웹 → Spring: 실행 시작

### 요청

```http
POST /api/v1/near-misses/{nearMissId}/auto-tags
Authorization: Bearer <access-token>
Content-Type: application/json
```

```json
{
  "lockVersion": 0
}
```

- 로그인과 해당 현장의 권한이 필요합니다.
- 권한은 보고자 본인 또는 태그 저장 권한이 있는 현장 승인자 이상으로 제한하는 것을 제안합니다.
- `nearMissId`는 양의 정수, `lockVersion`은 0 이상의 정수입니다.
- 보고의 현재 버전이 요청 버전과 다르면 `409 CONFLICT_MODIFIED`를 반환하고 AI를 호출하지 않습니다.
- 같은 보고·같은 버전의 실행이 이미 `PROCESSING`이면 새 실행을 만들지 않습니다. 같은 요청자가 시작한 실행이면 기존 `runId`를 반환하고, 다른 사용자가 시작한 실행이면 ID를 노출하지 않고 `409 INVALID_STATE`를 반환합니다. 버전이 바뀐 실행이 처리 중이어도 `409 INVALID_STATE`를 반환합니다.

### 접수 응답

`202 Accepted`

```json
{
  "success": true,
  "data": {
    "runId": 901,
    "nearMissId": 42,
    "status": "PROCESSING"
  }
}
```

Spring은 `agent_runs`에 `taskType=NEAR_MISS_TAG`, `conversationId=null`, `targetType=NEAR_MISS`, `targetId=nearMissId`인 실행을 저장하고 커밋한 뒤 AI 호출을 시작합니다. `input_json`에는 원본 보고 스냅샷과 실행 시작 시점의 `lockVersion`을 보관합니다.

### 실행 상태와 결과 조회

기존 실행 조회 API를 사용합니다.

```http
GET /api/v1/agent/runs/{runId}
Authorization: Bearer <access-token>
```

- 실행한 사용자와 현재 현장 권한을 다시 확인합니다.
- 상태는 `PROCESSING`, `DONE`, `FAILED`를 사용합니다. 자동 태깅은 사용자 되묻기 흐름이 아니므로 `WAITING_INPUT`을 반환하지 않습니다.
- 완료 결과에는 태그 후보와 사용자 검토 안내를 포함합니다. 실패 결과에는 공개 가능한 오류 코드·메시지를 포함합니다.

완료 결과의 `result` 예시:

```json
{
  "runStatus": "DONE",
  "nearMissId": 42,
  "lockVersion": 0,
  "tags": [
    {
      "axis": "ACCIDENT_TYPE",
      "value": "TRIP",
      "confidence": 0.91,
      "rationale": "보고 내용에 미끄러짐과 넘어질 뻔한 상황이 명시되어 있습니다."
    },
    {
      "axis": "HAZARD",
      "value": "ENVIRONMENTAL",
      "confidence": 0.84,
      "rationale": "바닥의 기름 유출이 직접적인 위험요인으로 기술되어 있습니다."
    }
  ],
  "notes": []
}
```

`tags`는 AI 제안이며 최종 안전 판정이 아닙니다. `rationale`은 사용자 검토용 설명으로 실행 결과에만 보관하며, 태그 DB 컬럼으로 저장하지 않습니다. 태그가 불확실하면 추측하지 않고 해당 태그를 생략하고 `notes`에 이유를 남깁니다. 태그가 하나도 없을 수도 있습니다.

## 3. Spring → `main-ai-model`: 태그 추론

### 요청

```http
POST /api/v1/agent/near-miss-tag
X-Internal-Token: <INTERNAL_API_TOKEN>
Content-Type: application/json
Accept: application/json
```

```json
{
  "runId": 901,
  "siteId": 7,
  "nearMissId": 42,
  "report": {
    "occurredAt": "2026-11-02T13:00:00",
    "title": "배관 랙에서 미끄러질 뻔함",
    "description": "점검 중 발판에 고인 물 때문에 미끄러졌으나 난간을 잡아 넘어지지 않았습니다.",
    "immediateAction": "작업을 멈추고 바닥의 물기를 제거했습니다.",
    "zoneName": "배관 랙"
  },
  "allowedTagCodes": [
    { "axis": "SEVERITY", "code": "NEAR_MISS", "label": "아차사고" },
    { "axis": "ACCIDENT_TYPE", "code": "TRIP", "label": "넘어짐" },
    { "axis": "HAZARD", "code": "ENVIRONMENTAL", "label": "작업환경" },
    { "axis": "WORK_TYPE", "code": "HEIGHT", "label": "고소" }
  ]
}
```

| 필드 | 타입 | 규칙 |
| --- | --- | --- |
| `runId` | 정수 | Spring이 발급한 양의 실행 ID. 추적용이며 AI가 생성하지 않음 |
| `siteId` | 정수 | Spring이 확정한 양의 현장 ID. 현장 접근을 위한 AI 도구 호출은 이번 계약에 포함하지 않음 |
| `nearMissId` | 정수 | Spring이 확정한 양의 보고 ID. 추적용 |
| `report.occurredAt` | 문자열 | 오프셋 없는 현장 현지 시각 |
| `report.title` | 문자열 | 1~200자 |
| `report.description` | 문자열 | 1~20,000자 |
| `report.immediateAction` | 문자열 또는 `null` | 최대 10,000자 |
| `report.zoneName` | 문자열 또는 `null` | 태깅 문맥용 현장 구역 이름 |
| `allowedTagCodes` | 배열 | Spring이 현재 사용 중인 코드표에서 전달. `SUBSTANCE`는 이 목록에 넣지 않음 |

보고자 이름, 연락처, 로그인 토큰 및 현장 회원 목록은 전달하지 않습니다. AI는 자체 DB에서 보고를 다시 읽지 않고, 전달된 스냅샷만 사용합니다.

### 성공 응답

`200 OK`; 기존 AI API 성공 envelope을 따릅니다.

```json
{
  "success": true,
  "data": {
    "runStatus": "DONE",
    "nearMissId": 42,
    "tags": [
      {
        "axis": "ACCIDENT_TYPE",
        "value": "TRIP",
        "confidence": 0.91,
        "rationale": "보고 내용에 미끄러짐 상황이 명시되어 있습니다."
      }
    ],
    "notes": [],
    "model": "configured-model",
    "steps": []
  }
}
```

- `runStatus`는 `DONE`만 허용합니다.
- `axis`는 `SEVERITY`, `ACCIDENT_TYPE`, `HAZARD`, `WORK_TYPE`, `SUBSTANCE` 중 하나입니다.
- 네 개 코드 축은 요청의 `allowedTagCodes`에 포함된 `axis`·`code` 조합만 반환할 수 있습니다.
- `SUBSTANCE.value`는 보고에 명시되어 근거가 있는 CAS 번호만 반환합니다. 형식뿐 아니라 체크섬도 Spring이 검증합니다. 모델이 CAS를 추측해서는 안 됩니다.
- `confidence`는 유한한 숫자 `0.0`~`1.0`이며 필수입니다. 안전 점수나 승인 판단으로 사용하지 않습니다.
- `rationale`은 1~500자의 짧은 설명입니다. 보고 문장에 근거하며 외부 문헌 인용처럼 표현하지 않습니다.
- 같은 `axis`·`value` 조합은 한 번만 반환합니다. `SEVERITY`는 최대 1개, 나머지 축은 각각 최대 10개입니다.
- `source`는 AI 응답에서 받지 않습니다. Spring이 저장 시 `source=AI`로 지정합니다.
- `steps`는 현재 에이전트 단계 DTO를 따릅니다. 단계가 없는 단순 분류이면 빈 배열을 반환할 수 있습니다.

### 실패 응답

```json
{
  "success": false,
  "error": {
    "code": "AI_BAD_RESPONSE",
    "message": "태깅 결과를 처리할 수 없습니다.",
    "details": []
  },
  "steps": []
}
```

| HTTP | 코드 예시 | 의미 |
| --- | --- | --- |
| `400` | `INVALID_INPUT` | 요청 필드 또는 허용 코드 목록이 잘못됨 |
| `401` / `403` | `INTERNAL_AUTH_FAILED` | 내부 토큰이 없거나 유효하지 않음 |
| `422` | `TAG_OUTPUT_INVALID` | 모델 결과가 JSON 스키마·코드·축별 개수 제약을 만족하지 않음 |
| `502` | `LLM_ERROR`, `AI_BAD_RESPONSE` | 모델 제공자 실패 또는 결과 파싱·검증 실패 |
| `503` | `AI_UNAVAILABLE` | 모델 서버를 사용할 수 없음 |

오류 메시지나 단계에 보고 전문, 토큰, 내부 프롬프트, 비밀값을 기록하지 않습니다. 오류 envelope은 현재 AI 서버의 `success=false`, `error`, `steps` 형식과 호환합니다.

## 4. Spring 결과 검증·저장 규칙

AI 결과는 신뢰하지 않는 입력으로 취급합니다. Spring은 최소한 다음을 다시 검증합니다.

1. `runId`, `siteId`, `nearMissId`, `taskType`이 현재 실행과 일치하는지 확인합니다.
2. 실행에 저장된 보고 스냅샷의 `lockVersion`과 현재 보고 버전이 같은지 확인합니다. 다르면 결과 태그를 저장하지 않고 실행을 충돌 실패로 종료합니다.
3. 태그 축·코드·중복·축별 개수·confidence 범위·CAS 형식과 체크섬을 검사합니다.
4. 현재 사용 중인 코드표에 없는 값은 저장하지 않습니다.
5. 기존 `USER` 태그는 보존합니다. 자동 태깅은 기존 `AI` 태그만 갱신할 수 있습니다.
6. 한 트랜잭션에서 태그 변경, 보고 상태 전이, `lockVersion` 증가, 이력 및 실행 완료 결과를 저장합니다. 부분 저장은 허용하지 않습니다.
7. 빈 태그 결과는 기존 `USER` 태그를 지우지 않습니다. 보고는 저장된 상태로 유지하고 실행 결과에 검토 안내를 남깁니다.

기존 사용자 태그 저장 API `PUT /api/v1/near-misses/{nearMissId}/tags`는 사용자 검토 전용입니다. 새 AI 태그를 생성하는 경로로 재사용하지 않고, AI 실행 결과를 위한 별도 서버 저장 서비스를 둡니다.

## 5. 실행·재시도·상태 규칙

- 실행 저장 후 커밋 이벤트로 모델을 호출합니다. 모델 호출은 DB 트랜잭션 밖에서 수행합니다.
- Spring의 기존 에이전트 전용 실행기와 `agent_runs` 상태를 사용합니다. 실행은 대화와 연결하지 않고 보고(`NEAR_MISS`)를 대상으로 기록합니다.
- 같은 보고의 같은 버전에서 실행이 이미 진행 중이면 중복 실행을 만들지 않습니다.
- 타임아웃·응답 유실 후 태그 저장 여부가 불명확하면 자동으로 AI 실행이나 쓰기 요청을 재전송하지 않습니다. 보고·태그 최신 상태를 다시 확인합니다.
- AI 실행 실패는 보고 생성이나 같은 구역 경고를 롤백하지 않습니다. 태깅 실패를 확인할 수 있도록 `FAILED`와 오류 코드를 남깁니다.
- 사용자는 완료된 태그를 확인하고 수정할 수 있습니다. AI 완료만으로 작업 승인, 위험성평가 승인 또는 작업 시작을 허용하지 않습니다.

## 6. 기존 API와 구분

| API | 담당 | 목적 |
| --- | --- | --- |
| `POST /api/v1/near-misses` | Spring 공개 API | 보고 저장 및 서버의 같은 구역 경고 생성 |
| `POST /api/v1/near-misses/{id}/auto-tags` | Spring 공개 API, 제안 | AI 태깅 비동기 실행 시작 |
| `GET /api/v1/agent/runs/{runId}` | Spring 공개 API | 실행 상태·결과 조회 |
| `POST /api/v1/agent/near-miss-tag` | Spring→AI 내부 API, 제안 | 보고 스냅샷 분류 및 태그 후보 반환 |
| `PUT` | `/api/v1/near-misses/{id}/tags` | Spring 공개 API | 사람이 검토한 태그 전체 교체 |
| `POST /api/v1/incident-search` | Spring 공개 API | 기존 태그·본문 기반 관련 사고 검색 |

`NEAR_MISS_TAG`는 DB 실행 종류로 정의되어 있지만, 컨트롤러 실행 경로·Spring 처리기·AI 모델 라우트와 결과 저장 처리는 별도 구현이 필요합니다. 자동 태깅 API는 기존 RAG 검색이나 경고 생성 API를 대신하지 않습니다.
