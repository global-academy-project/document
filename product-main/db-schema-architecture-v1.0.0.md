# DB 스키마 (업무 DB + 지식 DB + Qdrant)

산업현장 안전관리 AI 에이전트 · 작성 C · 2026-10-26

기존 미니 프로젝트(React → Spring Boot → FastAPI, MySQL) 위에 올린다. **회사마다 시스템을 따로 설치한다(시스템 1개 = 회사 1개).** 그래서 회사 테이블은 없고, 한 회사 안의 여러 현장을 대시보드 드롭다운으로 골라 본다.

| 저장소 | 소유 서비스 | 내용 | 마이그레이션 |
| --- | --- | --- | --- |
| 업무 DB (MySQL, 기존 DB) | **Spring Boot** | 회원·현장 권한, 업무 기록, 승인·이력, 규칙·판정, 대화, 알림, 서명·출력 문서 | Spring Flyway (`backend/.../db/migration`) |
| 지식 DB `agent_kb` (같은 MySQL 서버, 다른 데이터베이스) | **FastAPI** | 지식 원본·청크, MSDS 캐시, 정답 세트·채점 | Flyway 컨테이너 (`agent/db/migration`) |
| Qdrant | **FastAPI** | 검색용 벡터 | `init_qdrant.py` |

**서비스 경계: 다른 서비스의 저장소에는 접속하지 않는다.** 필요한 건 API로 받는다. 두 DB 사이에 FK도 없다. 계정 권한이 이걸 강제한다(3-1장).

**원칙: 잘못된 데이터는 앱이 아니라 DB에서 먼저 막는다.** 앱 코드에 버그가 있거나 누가 SQL을 직접 실행해도 규칙이 깨지지 않게 한다. 4명이 동시에 개발하면 앱 쪽 검증은 반드시 어딘가 빠진다.

## 1. 파일 구성

```
backend/src/main/resources/db/migration/   ← Spring 프로젝트의 같은 경로에 복사
  V1__baseline_ocr.sql            기존 schema.sql 그대로 (기준선)
  V2__sites_users_role.sql        현장(마감 설정 포함)·구역·설비, 사용자 등급, 현장별 권한, 로그인 세션, 이메일 인증·비밀번호 재설정 토큰, 소속·작업자·현장 배치·자격, 현장 취급 물질
  V3__works_risk.sql              당일 작업, 작업 물질, 작업 배치 인원, 위험성평가(일일·최초·정기·수시)
  V4__permits_inspections_tbm.sql 작업허가·가스측정(작업 전·중·후)·연장/중단/재개, 점검, TBM·다룬 작업·참석자, 첨부·사진 확인
  V5__near_miss_tags.sql          아차사고, 5축 코드표·태그, 작업 경고, 개선 조치
  V6__approvals_history.sql       결재(다단계)·결재선 정책, 이력, 안전일지, 알림, 서명, 양식·출력 문서, 열람 기록, 임시저장, 비동기 작업
  V7__rules.sql                   판단 규칙, 판정 결과·위반
  V8__agent_runs.sql              대화·메시지, 에이전트 실행·단계·근거 인용, AI 결과 평가
  V9__audit_guards.sql            트리거 42개: 증거 수정·삭제 금지, 제출·승인 후 내용 잠금, 측정 단계·허가 이벤트 검증
  R__rules_seed.sql               규칙 데이터 (고치면 다시 적용됨)
agent/db/migration/               ← FastAPI 프로젝트(ocr_api/db/migration)에 복사. 지식 DB 전용
  V1__kb_init.sql                 지식 원본·청크, MSDS 캐시
  V2__eval.sql                    정답 세트, 평가 실행, 문항별 결과
agent/scripts/init_qdrant.py      ← FastAPI 프로젝트의 scripts/에 복사
infra/db/init/10-agent-kb.sh      ← infra/db/init/에 복사. 지식 DB와 계정 3개 생성 (MySQL 최초 기동 시 자동)
verify/verify.sh                  전체 검증 (적용 → 정상 흐름 → 위반 케이스 197개)
verify/smoke_test.sql             정상 흐름: 현장 드롭다운, 시나리오 A·B·C·D, 편집·임시저장·제출·PDF 출력, 알림
verify/negative_tests.sh          위반 케이스: 잘못된 값과 권한 밖 접근이 실제로 거부되는지
```

## 2. 테이블 (업무 DB 62개 = 기존 5 + 신규 57 / 지식 DB 6개)

| 영역 | 테이블 | 업무 담당 |
| --- | --- | --- |
| 기존 | users(+등급·연락처·상태), documents, ocr_runs, ocr_pages, ocr_images | - |
| 현장·권한·로그인 | sites(+마감 설정), work_zones, equipment, site_members, refresh_tokens, account_tokens, password_history | D (로그인은 공통) |
| 소속·작업자 | teams, workers, worker_sites, worker_qualifications | D (자격 판정은 **C** 규칙 엔진) |
| 취급 물질 | site_chemicals, work_materials | A (MSDS 연결)·B |
| 작업 | works, work_risk_factors, work_workers | B |
| 위험성평가 | risk_assessments, risk_assessment_items | B |
| 작업허가 | permits, permit_measurements, permit_checks, permit_events | **C** |
| 점검 | inspections, inspection_items | **C** |
| TBM·사진 | tbm_records, tbm_works, tbm_attendees (B), attachments (공용), photo_checks (**C**) | B·C |
| 아차사고·분류 | near_misses, tag_codes, record_tags | A |
| 경고·개선 조치 | work_warnings (A), corrective_actions (A·D) | A·D |
| 결재·이력 | approvals, approval_steps, approval_policies, record_history, daily_safety_logs | D |
| 서명·문서 | signatures, form_templates, generated_documents, record_access_logs | D |
| 편집·비동기 | form_drafts (임시저장, 공통), async_jobs (PDF 출력·재판정 등, D) | 공통·D |
| 알림 | notifications, notification_recipients, notification_preferences | D |
| 규칙 | rules, rule_evaluations, rule_violations | **C** |
| 대화·에이전트 | agent_conversations, agent_messages, agent_runs, agent_steps, evidence_citations, agent_feedback | B |
| **지식 (agent_kb, FastAPI)** | kb_sources, kb_chunks, msds_cache | A |
| **평가 (agent_kb, FastAPI)** | eval_cases, eval_runs, eval_results | 4명 각자 문항, 실행 도구는 B |

```mermaid
erDiagram
  sites ||--o{ site_members : grants
  users ||--o{ site_members : has
  sites ||--o{ work_zones : has
  sites ||--o{ workers : employs
  workers ||--o{ worker_qualifications : holds
  works ||--o{ work_workers : assigns
  workers ||--o{ work_workers : assigned
  work_zones ||--o{ equipment : has
  sites ||--o{ works : has
  work_zones ||--o{ works : at
  works ||--o{ work_risk_factors : has
  works ||--o{ risk_assessments : has
  risk_assessments ||--o{ risk_assessment_items : has
  works ||--o{ permits : needs
  permits ||--o{ permit_measurements : measured
  permits ||--o{ permit_checks : checked
  permits |o--o| rule_evaluations : last_eval
  rule_evaluations ||--o{ rule_violations : has
  rules ||--o{ rule_violations : violated
  works ||--o{ tbm_records : has
  tbm_records ||--o{ tbm_attendees : attends
  works ||--o{ work_warnings : warned
  near_misses ||--o{ work_warnings : causes
  form_templates ||--o{ generated_documents : prints
  attachments ||--o{ photo_checks : checked
  work_zones ||--o{ near_misses : at
  tag_codes ||--o{ record_tags : allows
  agent_conversations ||--o{ agent_messages : has
  agent_conversations ||--o{ agent_runs : runs
  agent_runs ||--o{ agent_steps : has
  agent_runs ||--o{ evidence_citations : cites
```

지식 DB는 따로: `kb_sources 1 ── N kb_chunks`. 업무 DB의 `evidence_citations`는 `kb_point_id`(Qdrant 점 id) 값만 들고 있고 FK는 없다. 원본이 바뀌어도 근거가 남도록 인용 원문(`snippet`)을 필수로 저장한다.

## 3. 엄격 설계 규칙 (DB가 직접 막는 것)

| # | 규칙 | 어떻게 막나 | 막는 사고 |
| --- | --- | --- | --- |
| 1 | 코드 값은 정해진 것만 | 모든 상태·유형 컬럼에 CHECK (CHECK 235개) | `'APPROVE'`, `'approved'` 같은 오타 상태가 쌓여 조회에서 빠짐 |
| 2 | 다른 현장 기록끼리 연결 금지 | `(id, site_id)` 복합 FK. 작업↔구역, 허가↔작업, 점검↔설비·구역·작업, TBM↔작업, 아차사고↔구역·작업, 작업 배치·TBM 참석·서명↔작업자, 경고↔아차사고 | 현장1 작업에 현장2 구역이 붙어 다른 현장에 경고가 뜸 |
| 3 | 업무 기록은 지우지 않음 | FK 기본값 삭제 금지(RESTRICT). CASCADE는 항목·측정값·메시지 같은 순수 자식만 | 현장·사용자 하나 지웠다가 한 달치 허가 기록이 같이 사라짐 |
| 4 | 증거는 고칠 수 없음 | V9 트리거: 이력, 판정 결과·위반, 서명, 출력 문서, 열람 기록의 UPDATE·DELETE 차단 | 감사 때 "기록을 고쳤다"는 의심을 받을 여지 |
| 5 | 결정된 승인은 되돌릴 수 없음 | V9 트리거 + CHECK | 승인된 허가를 대기로 돌려 다시 승인하는 우회 |
| 6 | 같은 기록에 대기 승인은 하나 | 생성 컬럼 `pending_key` + UNIQUE (MySQL엔 부분 유니크가 없어서) | 버튼 두 번 눌러 승인 요청 2건 |
| 7 | 상태와 짝 값은 항상 같이 | CHECK: 완료↔완료 시각, 결정↔결정 시각·결정자, 반려↔사유, 실패↔오류 내용, 제출↔결과 | "완료인데 완료 시각 없음" 같은 반쪽 데이터 |
| 8 | 측정값은 항목·단위 짝이 맞아야 | CHECK: 산소·탄산가스=%, 가연성=%LEL, 독성=ppm, 음수·100% 초과 금지 | 산소 17.5를 ppm으로 넣어 규칙 엔진이 통과시킴 |
| 9 | 분류 태그는 코드표 값만 | 생성 컬럼 + FK → tag_codes. 물질은 CAS 형식 CHECK | `'FIRE'`, `'Fire'`, `'화재'`가 섞여 5축 검색이 깨짐 |
| 10 | 규칙은 근거 조항 필수, 연산자별 기준값 필수 | NOT NULL + CHECK, 같은 규칙의 켜진 판은 하나(UNIQUE) | 근거 없는 판정, 상한 없는 RANGE 규칙 |
| 11 | 권한은 현장별로, 중복 없이 | `site_members` PK(사용자, 현장), 역할 CHECK. 권한 준 현장은 삭제 금지 | 같은 현장에 역할 두 개가 생겨 어느 권한으로 판단할지 모호해짐 |
| 12 | 동시 수정 충돌 감지 | 여러 사람이 고치는 테이블에 `lock_version` (JPA `@Version`) | 작성자 수정과 승인자 반려가 동시에 일어나 한쪽이 덮어써짐 |
| 13 | 반려 후 재작성은 새 판 | 위험성평가·허가에 `revision` + UNIQUE, 이전 판은 SUPERSEDED | 반려된 내용이 덮여 "무엇을 고쳤는지" 사라짐 |
| 14 | 서명은 내용과 묶임 | 서명 시점 문서 내용 해시(`content_sha256`) 필수, 같은 사람·같은 자격 서명 중복 금지 | 서명 후 내용을 바꿔도 서명이 유효해 보이는 위조 |
| 15 | 양식 판은 범위마다 하나만 켜짐 | 생성 컬럼 `active_key` + UNIQUE (전체 공통 / 현장 전용) | 같은 허가서 양식 v1·v2가 동시에 쓰여 출력물이 섞임 |
| 16 | 긴급 경고는 조치를 적어야 확인 | CHECK: CRITICAL 경고 확인 시 `ack_note` 필수 | 경고 버튼만 누르고 넘어감 |
| 17 | 제출·승인 후 내용 잠금 | V9 트리거: 편집 가능 상태가 아니면 내용 컬럼·하위 항목 변경 차단. 상태 변경은 허용 | 승인 요청 뒤(또는 승인 뒤) 몰래 내용을 바꿔 "승인된 문서"가 달라짐 |
| 18 | 측정값은 추가만 | V9 트리거: 가스 측정값 UPDATE·DELETE 금지, 재측정은 새 행 | 기준 미달 측정값을 고쳐서 차단을 풀어 버림 |
| 19 | 출력은 저장된 판만 | `generated_documents.source_version` 필수 (기록의 lock_version) | 임시저장 중인 내용이 인쇄돼 정식 기록과 종이가 다름 |
| 20 | 같은 요청은 한 번만 처리 | `async_jobs.idempotency_key` UNIQUE | [PDF 출력] 두 번 클릭으로 같은 문서가 두 장 생김 |
| 21 | 측정은 단계에 맞는 허가 상태에서만 | V9 트리거: 작업 전(PRE_WORK)은 신청 단계, 작업 중(DURING)은 승인·중단 상태, 작업 후(POST_WORK)는 승인 상태 | 승인 뒤 측정을 아예 못 넣거나, 승인 뒤에 "작업 전" 값을 끼워 넣음 |
| 22 | 허가 연장은 이벤트로, 뒤로만 | `permit_events` + 트리거: 연장 시각 > 현재 종료 시각, 결정 후 변경·삭제 금지, 재개는 중단된 허가만 | 허가서 시각을 직접 고쳐 무단 연장 |
| 23 | 결재 단계는 순서대로 하나씩 | `approval_steps.pending_key` UNIQUE (내 차례 1개), 결정 후 변경 금지 | 2단계 결재에서 두 사람이 동시에 결재하거나 순서를 건너뜀 |
| 24 | 작업자는 배치된 현장에서만 | 작업 배치·TBM 참석·서명이 `worker_sites`(작업자, 현장)를 참조 | 다른 현장 인원이 명단에 들어감. 그러면서도 현장 이동은 자유롭게 |
| 25 | 작업 하나는 하루 TBM 한 번 | `tbm_works` UNIQUE(작업, 날짜) + TBM 날짜와 복합 FK | 같은 작업이 두 TBM에 중복 집계 |

**DB에서 막지 않고 앱에서 막는 것** (이유가 있는 예외)

| 항목 | 이유 |
| --- | --- |
| 다형 참조 `(target_type, target_id)`의 대상 존재 여부 | 대상 테이블이 여러 개라 FK 불가. CHECK로 유형만 제한 |
| 작성자 = 승인자 금지, 조치자 = 확인자 금지 | 안내 모드(50인 미만)는 1인이 겸하는 자기 확인을 허용해야 함 |
| 승인자가 그 현장의 APPROVER인지 | 현장 권한은 바뀌므로 요청 시점에 서비스가 `site_members`로 확인 |
| 작업자 자격 만료 여부 | 날짜 기준이 "오늘"이라 CHECK로 불가 → 규칙 엔진(C)이 판정 |
| 아차사고 발생 시각 ≤ 보고 시각 | 휴대폰 시계 오차로 정상 보고가 막힐 수 있음 (앱에서 5분 허용) |
| 위험도 등급(HIGH/MEDIUM/LOW)과 점수의 대응 | 현장마다 기준이 다름 → 규칙 엔진에서 판정 |
| 상태 전이 순서 (DRAFT → PENDING → APPROVED) | 업무 흐름마다 다름 → 서비스 계층. 단, 결정된 승인 되돌리기는 DB가 막음 |

## 3-0a. 구조 점검에서 바꾼 것 (나중에 바꾸기 어려운 테이블 단위)

| 문제 | 바꾸기 전 | 바꾼 뒤 |
| --- | --- | --- |
| **작업 중 측정을 넣을 수 없음 (버그)** | 승인되면 측정값 추가 금지 → 밀폐공간 주기 측정이 막힘 | 측정에 단계(작업 전·중·후) 추가, 단계별로 허용 상태를 다르게 |
| **작업자가 현장을 옮길 수 없음 (버그)** | 작업자에 현장 1개 고정 + 기록들이 (작업자, 현장)을 참조 → 현장을 바꾸면 FK가 막음 | 작업자는 회사 단위, 현장 배치는 `worker_sites`. 여러 현장 배치·이동 가능, 과거 기록 유지 |
| 허가 연장·중단·재개·교대 | 승인 후 잠금 때문에 연장할 방법이 없음 | `permit_events` (연장은 승인 필요, 측정 이상 시 중단, 재측정 후 재개) |
| 다단계 결재 | 결재 1건 = 승인자 1명 | `approval_steps` + 현장·문서별 결재선 `approval_policies` (50인 미만은 1단계) |
| TBM이 여러 작업을 다룸 | TBM 1회 = 작업 1개 | `tbm_works`로 여러 작업 연결, TBM에 소속·장소 추가 |
| 소속이 문자열 | "정비팀"/"정비 팀"이 갈려 통계가 깨짐 | `teams` (사내 부서·외부 업체) |
| 취급 물질이 JSON | "톨루엔 다룬 작업" 조회·경고가 어려움 | `work_materials`, 현장·구역별 `site_chemicals` (MSDS 비치 목록, 화기작업 경고 근거) |
| 위험성평가가 일일만 | 정기·수시 평가(공정 단위, 작업 없음)를 저장할 수 없음 | `assessment_type` + 구역 + 일일 평가가 참고한 정기 평가 연결 |

**검토했지만 지금은 넣지 않은 것** (나중에 컬럼이나 테이블 추가로 충분)
- 구역 계층(공장 > 공정 > 설비): `work_zones.parent_id` 컬럼 하나면 된다.
- 현장별 규칙 기준 강화: 법정 기준은 같고, 더 엄격하게 하려면 `site_rule_overrides` 테이블을 추가하면 된다.
- 근무 교대·휴일 달력: 마감 알림 고도화 때 추가한다.
- 이력 테이블 분할(파티션): 데이터가 수백만 건이 넘을 때 `record_history`만 따로 처리한다.

## 3-0. 사용자 등급과 현장 드롭다운

| 등급 (`users.role`) | 드롭다운에 보이는 현장 | 할 수 있는 일 |
| --- | --- | --- |
| ADMIN | 이 회사의 **모든 현장** | 현장 추가, 사용자 가입 승인, 현장 권한 부여, 양식·규칙 관리 |
| MEMBER | `site_members`에 ACTIVE로 있는 현장만 | 현장별 역할에 따라 |

| 현장별 역할 (`site_members.role`) | 할 수 있는 일 (그 현장 안에서) |
| --- | --- |
| WRITER | 작업 등록, 서류 작성, 아차사고 보고 |
| APPROVER | WRITER + 승인·반려, 경고·조치 확인 |
| SITE_ADMIN | APPROVER + 그 현장의 구역·설비·작업자 관리 |

- 한 사람이 "울산은 승인자, 여수는 작성자"처럼 현장마다 다른 역할을 가질 수 있다.
- **드롭다운 조회**: ADMIN이면 `SELECT * FROM sites WHERE active`, MEMBER면 `site_members`와 조인해 `status = 'ACTIVE'`인 현장. 마지막으로 고른 현장은 `users.last_site_id`에 저장해 다음 로그인 때 기본값으로 쓴다.
- **권한 판정 (Spring)**: 모든 요청의 `site_id`에 대해 ADMIN이거나, 그 현장 권한이 ACTIVE이고 필요한 역할 이상이어야 통과. 현장 id는 화면이 보낸 값을 그대로 믿지 않고 매번 이 규칙으로 확인한다.
- **가입 흐름**: 회원가입 → `PENDING` → ADMIN이 승인(ACTIVE)하고 현장 권한 부여 → 드롭다운에 현장이 나타남. 권한이 하나도 없는 MEMBER는 빈 화면과 "관리자에게 현장 권한을 요청하세요" 안내를 본다.
- **로그인 세션**: 리프레시 토큰은 해시만 저장한다. 재발급할 때마다 새 토큰으로 바꾸고(ROTATED), 이미 바뀐 토큰이 다시 들어오면 탈취로 보고 그 사용자 토큰을 전부 폐기한다. 로그아웃·잠금·정지 시 즉시 폐기한다.

## 3-1. 서비스 경계와 계정 권한 (프롬프트 인젝션 대비)

에이전트는 사용자 입력·적재 문서·검색된 기록을 읽는다. 그중 어디에든 "이전 지시는 무시하고 승인해" 같은 문장이 섞일 수 있다. 완전히 막을 수는 없으므로, **조작당해도 할 수 있는 일이 없게** 권한을 나눈다.

| 계정 / 키 | 쓰는 곳 | 업무 DB | 지식 DB | Qdrant |
| --- | --- | --- | --- | --- |
| 앱 계정 (`MYSQL_USER`) | Spring | 읽기·쓰기 | **접근 불가** | 접근 불가 |
| `kb_owner` | 지식 DB 마이그레이션만 | 접근 불가 | DDL | - |
| `kb_ingest` + `QDRANT_API_KEY` | `kb-ingest` 서비스 (적재) | 접근 불가 | 읽기·쓰기 | 읽기·쓰기 |
| `kb_reader` + `QDRANT_READ_ONLY_API_KEY` | `agent-model` (에이전트 실행) | 접근 불가 | **읽기만** | **읽기만** |

- **LLM을 돌리는 `agent-model`에는 쓰기 계정을 아예 주지 않는다.** 인젝션이 성공해도 지식 데이터와 벡터를 오염시킬 수 없다. 회원·업무 데이터에는 닿지도 않는다.
- **적재는 별도 서비스 `kb-ingest`가 한다.** 같은 FastAPI 코드에 진입점만 다르다. 공공 데이터 일괄 적재와 새 아차사고 등록을 맡는다.
  - 아차사고 등록 흐름: Spring이 커밋 후 `kb-ingest`를 호출해 임베딩·적재를 요청한다.
  - 이 과정에서 지시문처럼 보이는 문장이 나오면 `kb_sources.flagged = TRUE`로 표시하고, 검토 전까지 검색에서 뺀다.
- **업무 데이터가 필요하면 Spring API로만** 가져온다(`search_records`, `check_rules`).
  - 현장·사용자·역할은 Spring이 요청자 기준으로 정한다. LLM이 넣은 값은 쓰지 않는다.
- **이 권한은 `negative_tests.sh`가 실제 접속으로 확인한다.** 예를 들어 "읽기 계정으로 지식 데이터 수정", "적재 계정으로 회원 정보 조회"는 권한 오류(1142)로 거부돼야 통과한다.

## 4. 코드표 (Java enum과 반드시 같게)

| 컬럼 | 값 |
| --- | --- |
| sites.mode | VERIFY, GUIDE |
| users.role (계정 등급) | ADMIN, MEMBER |
| site_members.role (현장별 역할) | WRITER, APPROVER, SITE_ADMIN |
| users.status | PENDING, ACTIVE, INACTIVE, LOCKED |
| refresh_tokens.revoke_reason | LOGOUT, ROTATED, LOCKED, ADMIN, REUSE_DETECTED |
| workers.employment_type | DIRECT, CONTRACTOR, DAILY |
| worker_qualifications.qual_type | TRAINING, LICENSE, HEALTH_CHECK |
| work_workers.role_in_work | LEADER, WORKER, FIRE_WATCHER, ATTENDANT, SUPERVISOR |
| teams.team_type | INTERNAL, EXTERNAL |
| site_chemicals.unit / work_materials.unit | KG, L, TON, M3 |
| risk_assessments.assessment_type | DAILY, INITIAL, REGULAR, ADHOC |
| permit_measurements.phase | PRE_WORK, DURING, POST_WORK |
| permit_events.event_type / status | EXTEND, SUSPEND, RESUME, HANDOVER, CLOSE_CHECK / REQUESTED, APPROVED, REJECTED |
| approval_steps.status / required_role | WAITING, PENDING, APPROVED, REJECTED, SKIPPED / APPROVER, SITE_ADMIN, ADMIN |
| works.work_type | GENERAL, HOT, CONFINED, HEIGHT |
| works.status | DRAFT, PENDING, APPROVED, REJECTED, BLOCKED, IN_PROGRESS, DONE, CANCELED |
| risk_assessments.status | DRAFT, PENDING, APPROVED, REJECTED, SUPERSEDED |
| permits.status | DRAFT, CHECKING, BLOCKED, PENDING, APPROVED, SUSPENDED, REJECTED, CLOSED, SUPERSEDED |
| permit_measurements.item / unit | O2, LEL, H2S, CO, CO2, NH3, CL2, OTHER / PCT, PPM, PCT_LEL |
| inspections.status / result | DRAFT, PENDING, APPROVED, REJECTED / PASS, FAIL, PARTIAL |
| tbm_records.status | DRAFT, DONE, APPROVED |
| near_misses.status | REPORTED, TAGGED, REVIEWED, CLOSED |
| approvals.status | PENDING, APPROVED, REJECTED, CANCELED |
| approvals.target_type | WORK, RISK, PERMIT, PERMIT_EVENT, INSPECTION, TBM, DAILY_LOG, CORRECTIVE_ACTION |
| record_history.action | CREATE, UPDATE, SUBMIT, APPROVE, REJECT, BLOCK, UNBLOCK, AI_DRAFT, CLOSE, CANCEL, ROLE_CHANGE, STATUS_CHANGE, GRANT, REVOKE, ACKNOWLEDGE, VERIFY, SIGN, EXPORT, PASSWORD_RESET, PASSWORD_CHANGE, UNLOCK, EMAIL_VERIFY, EMAIL_CHANGE |
| account_tokens.purpose | EMAIL_VERIFY, EMAIL_CHANGE, PASSWORD_RESET |
| users.email_verified_via | LINK, ADMIN |
| work_warnings.source_type / severity | NEAR_MISS, RULE, ZONE_HAZARD, QUALIFICATION / WARN, CRITICAL |
| corrective_actions.status | OPEN, IN_PROGRESS, DONE, VERIFIED, CANCELED |
| signatures.purpose | WRITER, APPROVER, ATTENDEE, FIRE_WATCHER, ATTENDANT, SUPERVISOR, VERIFIER |
| form_templates.form_type | RISK_ASSESSMENT, HOT_WORK_PERMIT, CONFINED_SPACE_PERMIT, HEIGHT_WORK_PERMIT, INSPECTION, TBM, DAILY_LOG, AUDIT_REPORT |
| form_templates.status | DRAFT, ACTIVE, RETIRED |
| agent_feedback.rating / reason_code | UP, DOWN / WRONG_FACT, MISSING_HAZARD, BAD_CITATION, WRONG_RULE, FORMAT, TOO_SLOW, OTHER |
| rules.operator / severity | RANGE, GTE, GT, LTE, LT, EQ, IN, REQUIRED / BLOCK, WARN |
| agent_runs.status | PROCESSING, WAITING_INPUT, DONE, FAILED |
| agent_runs.task_type | WORK_REGISTER, RISK_DRAFT, PERMIT_DRAFT, INSPECTION_DRAFT, TBM_DRAFT, NEAR_MISS_TAG, PHOTO_CHECK, RECORD_QUERY, DAILY_LOG, AUDIT_REPORT |
| agent_steps.tool_name | search_accidents, search_guides, lookup_msds, classify_work, check_rules, check_photo, fill_form, export_form, search_records, request_approval |
| notifications.type | APPROVAL_REQUEST, APPROVAL_RESULT, BLOCKED, NEAR_MISS_WARNING, DEADLINE, ACTION_DUE, AGENT_DONE, DOCUMENT_READY, JOB_FAILED, ACCOUNT, SYSTEM |
| form_drafts.target_type | WORK, RISK, PERMIT, INSPECTION, TBM, NEAR_MISS, DAILY_LOG, CORRECTIVE_ACTION |
| async_jobs.job_type / status | EXPORT_PDF, RULE_CHECK, AUDIT_REPORT, DAILY_LOG_BUILD / QUEUED, RUNNING, DONE, FAILED, CANCELED |
| notifications.severity | INFO, WARN, CRITICAL |
| 5축 코드 | tag_codes 테이블 (V5 INSERT 참고) |

값을 추가하려면 CHECK를 바꾸는 새 마이그레이션 + Java enum을 같은 PR에서 함께 바꾼다.

## 5. 대화 기록과 최근 항목

| 테이블 | 하는 일 |
| --- | --- |
| agent_conversations | 대화방. 제목(= 첫 사용자 메시지 앞부분), 고정, 누적 요약, 삭제(숨김) |
| agent_messages | 메시지. USER / ASSISTANT / TOOL, 화면 글과 구조화 내용 |
| agent_runs | 메시지 하나를 처리한 실행. 되묻기는 WAITING_INPUT으로 끝나고 답변이 오면 같은 대화의 다음 실행 |
| agent_steps | 도구 호출 원본 입출력. 메시지에는 요약만 |

- **최근 항목**: Claude처럼 **첫 질문만** 제목으로 보여준다. 대화를 만들 때 첫 사용자 메시지 앞 100자를 `title`에 넣고(빈 제목은 CHECK로 금지), 이후에는 바뀌지 않는다. 사용자가 이름 바꾸기를 하면 그 값으로 덮어쓴다.
  - 목록: `SELECT id, title, pinned, last_message_at ... WHERE user_id = ? AND deleted_at IS NULL ORDER BY pinned DESC, last_message_at DESC LIMIT 30` (색인 있음)
  - 정렬은 마지막 대화 시각 기준이라, 오래된 대화라도 다시 이어서 말하면 위로 올라온다. 제목 검색은 FULLTEXT(ngram).
- **메시지 추가 순서** (한 트랜잭션): 대화 행 `SELECT ... FOR UPDATE` → `seq = message_count + 1`로 INSERT → 대화의 `message_count`, `last_message_at` 갱신. 이 순서를 안 지키면 동시 전송 때 seq 중복으로 실패한다(UNIQUE가 막음).
- **LLM 문맥**: `summary` + 최근 메시지 20개(또는 토큰 한도). 넘치면 오래된 부분을 요약에 합치고 `summary_until_seq`를 올린다. 원본 메시지는 지우지 않는다.
- **대화에 묶인 작업이 있으면** 그 작업의 현재 상태를 DB에서 새로 읽어 문맥 맨 앞에 넣는다. 대화 속 옛날 값이 아니라 현재 기록이 기준이다.
- 대화를 삭제해도 목록에서만 숨긴다. 그 대화로 만든 허가서가 감사 대상일 수 있다.

## 5-0. 화면 편집 → 임시저장 → 저장 → 제출 → 출력

AI 초안이든 직접 작성이든, 문서는 아래 순서로만 움직인다.

```
[화면 열기]   기록의 lock_version을 기준 판(base_version)으로 기억, form_drafts 생성
    │
[편집 중]     2~3초 멈출 때마다 자동 임시저장 (form_drafts만 바뀜, 정식 기록·이력은 그대로)
    │         client_seq 증가: UPDATE ... WHERE id=? AND client_seq < :seq  → 늦게 도착한 옛 요청은 0행
    │         첨부 사진은 owner_type='DRAFT'로 먼저 올려 둠
    │
[저장]        한 트랜잭션: 정식 기록 UPDATE ... WHERE lock_version = :base_version
    │           → 1행이면 성공: 하위 항목 반영, 첨부 owner 옮김, record_history(UPDATE), 임시저장 삭제
    │           → 0행이면 "다른 사람이 먼저 고쳤습니다": 최신 내용과 내 임시저장을 나란히 보여주고 고르게 함
    │         수정 후 규칙 재판정이 필요하면 async_jobs(RULE_CHECK) 등록
    │
[제출]        status DRAFT → PENDING, approvals 생성. 이 순간부터 내용 잠금(V9 트리거)
    │         고치려면 승인 요청 취소 → DRAFT로 되돌린 뒤 수정 / 반려되면 revision+1 새 판
    │
[PDF 출력]    async_jobs(EXPORT_PDF) 등록 → 즉시 응답(작업 id) → 화면은 진행률 폴링
              작업 완료 시 generated_documents(source_version = 출력한 판) + 알림 DOCUMENT_READY
```

**임시저장 (`form_drafts`)**
- 정식 기록과 분리했다. 그래서 반쯤 고친 내용이 승인자 화면에 보이지 않고, 키 입력마다 이력이나 버전이 늘지 않는다.
- 기존 기록은 사람당 하나만 둔다(UNIQUE). 새 문서는 여러 개를 동시에 쓸 수 있다(아차사고 2건 등).
- "작성 중인 문서" 목록은 `WHERE user_id = ? ORDER BY updated_at DESC`로 보여준다. 브라우저가 꺼져도 이어 쓸 수 있다.
- 같은 기록에 최근 5분 안에 다른 사람의 임시저장이 있으면 "김OO님이 편집 중"을 표시한다. 막지는 않는다. 충돌은 저장할 때 `lock_version`으로 잡는다.
- 30일이 지나면 정리 배치가 지운다(`expires_at`). 그 임시저장에 올린 사진(owner_type='DRAFT')도 같이 지운다.
- 1MB가 넘는 임시저장은 거부한다. 사진은 첨부로 따로 올린다.

**비동기 작업 (`async_jobs`)**
- PDF 출력, 수정 후 규칙 재판정, 감사 보고서, 안전일지 집계처럼 오래 걸리는 일을 맡는다. AI 실행은 기존처럼 `agent_runs`가 맡는다.
- 왜 테이블이 필요한가: `@Async`만 쓰면 서버가 재시작될 때 처리 중인 작업이 사라진다. 테이블에 남겨 두면 재시작 후 이어서 처리하고, 실패하면 `run_after`를 뒤로 미뤄 최대 `max_attempts`번 재시도한다.
- 작업 가져가기: `SELECT ... WHERE status='QUEUED' AND run_after <= NOW(6) ORDER BY id LIMIT 1 FOR UPDATE SKIP LOCKED`. 서버가 여러 대여도 한 작업을 두 번 처리하지 않는다.
- 중복 방지: `idempotency_key = EXPORT_PDF:RISK:{id}:v{lock_version}`. 같은 판을 두 번 출력 요청하면 기존 작업을 돌려준다. 내용을 고쳐 판이 바뀌면 새 작업이 된다.
- 처리 중에 기록 판이 바뀌면(`target_version` ≠ 현재 `lock_version`) 그 작업은 실패로 끝내고 다시 요청하게 한다. 옛 내용이 인쇄되는 걸 막기 위해서다.
- API 예: `POST /api/v1/jobs`(등록, 202 + jobId), `GET /api/v1/jobs/{id}`(상태·진행률), 완료 알림은 SSE로 보낸다.

**편집 가능 상태 (V9 트리거가 강제)**

| 기록 | 내용을 고칠 수 있는 상태 | 잠긴 뒤 허용되는 것 |
| --- | --- | --- |
| 작업 | DRAFT, PENDING, REJECTED, BLOCKED | 상태 변경 (진행 시작, 완료) |
| 위험성평가 (+위험요인) | DRAFT | 승인·반려·대체(SUPERSEDED) |
| 작업허가 (+체크 항목) | DRAFT, CHECKING, BLOCKED | 승인·반려·중단·재개·마감, 연장은 `permit_events`로 |
| 가스 측정값 | 추가만. 작업 전은 신청 단계, 작업 중은 승인·중단 상태, 작업 후는 승인 상태 | 수정·삭제는 언제나 불가 |
| 작업 취급 물질 | 작업이 승인 전일 때 | 없음 |
| 점검표 (+항목) | DRAFT | 승인·반려 |
| TBM (+다룬 작업·참석자) | DRAFT, DONE | 없음 (승인 후 완전 잠금) |
| 안전일지 | DRAFT, PENDING | 없음 (승인 후 완전 잠금) |

## 5-1. 알림

**구조: 알림 내용 1건 + 받는 사람별 상태 N건**

| 테이블 | 하는 일 |
| --- | --- |
| notifications | 알림 내용. 유형, 중요도, 제목·문구, 누르면 이동할 기록, 일으킨 주체, 중복 방지 키, 만료 시각 |
| notification_recipients | 사용자별 수신함. 실시간 전달 시각, **읽음**, **끄기** |
| notification_preferences | 사용자별 설정. 행이 없으면 기본값(웹 켜짐). 차단·아차사고 경고·승인 요청은 끌 수 없음(CHECK) |

같은 승인 결과라도 작성자는 읽고 승인자는 안 읽을 수 있다. 그래서 읽음·끄기는 내용이 아니라 받는 사람 행에 둔다.

**읽음과 끄기는 다르다**

| 동작 | SQL | 화면 |
| --- | --- | --- |
| 목록 | `WHERE user_id=? AND dismissed_at IS NULL` + 만료 안 된 것, `ORDER BY id DESC LIMIT 50` | 끄기 전까지 계속 보임 |
| 배지 | `COUNT(*) WHERE user_id=? AND read_at IS NULL AND dismissed_at IS NULL` | 빨간 숫자 |
| 하나 읽음 | `SET read_at=NOW(6) WHERE id=? AND user_id=?` | 굵은 글씨 해제, 목록에는 남음 |
| **모두 읽음** | `SET read_at=NOW(6) WHERE user_id=? AND read_at IS NULL AND id <= :화면에서 본 마지막 id` | 배지 0 |
| **X 버튼 (하나씩 지우기)** | `SET read_at=COALESCE(read_at,NOW(6)), dismissed_at=NOW(6) WHERE id=? AND user_id=?` | 그 알림만 목록에서 사라짐 |
| 모두 지우기 | X와 같은 처리 + `id <= :마지막 id` | 목록 비움 |

- 사용자에게는 "삭제"지만 DB에서는 **내 수신 행에 `dismissed_at`만 찍는다.** 같은 알림을 받은 다른 사람(승인자 등)의 목록에는 그대로 남아야 하기 때문이다. 다시 나타나지 않으니 화면에서는 완전히 지운 것과 같다. 실제 행 정리는 30일 정리 배치가 한다.

- "모두 읽음"에 `id <= 마지막 id`를 붙이는 이유: 버튼을 누르는 순간 새로 도착한 알림까지 읽음 처리되면, 사용자가 보지 못한 차단 알림이 묻힌다.
- 모든 UPDATE에 `user_id = 로그인 사용자`를 붙인다. 남의 알림을 id만 바꿔 끌 수 없게.
- CRITICAL(차단 등)은 끄기 전에 확인창을 띄운다(화면 규칙).

**누가 받나** (발송 서비스 한 곳에서 결정)

| 상황 | 유형 | 받는 사람 |
| --- | --- | --- |
| 승인 요청 | APPROVAL_REQUEST | 지정 승인자, 없으면 `site_members`에서 그 현장 APPROVER·SITE_ADMIN (ACTIVE) |
| 승인·반려 | APPROVAL_RESULT | 요청한 사람 + 기록 작성자 (같은 사람이면 1번만) |
| 규칙 차단 | BLOCKED (CRITICAL) | 작성자 + 같은 현장 승인자 |
| 같은 구역 아차사고 | NEAR_MISS_WARNING (WARN) | 그 구역에 오늘 작업을 등록한 작성자 |
| 마감 임박 (TBM, 안전일지) | DEADLINE | 해당 작성자. 기준 시각은 `sites.tbm_deadline`, `daily_log_deadline`. `dedupe_key`로 하루 1번 |
| 승인 지연 | APPROVAL_REQUEST (재알림) | 요청 후 `sites.approval_due_minutes`가 지나도 대기면 승인자에게 한 번 더 |
| 개선 조치 기한 | ACTION_DUE | 조치 담당자. 기한 전날·당일 |
| 에이전트 작업 완료 | AGENT_DONE | 실행을 요청한 사용자 (화면을 떠나 있었을 때) |
| PDF·보고서 준비 완료 | DOCUMENT_READY | 작업을 요청한 사용자 |
| 비동기 작업 최종 실패 | JOB_FAILED (WARN) | 작업을 요청한 사용자 (재시도까지 모두 실패했을 때) |

**비동기 전달 흐름** (기존 OCR의 커밋 후 비동기 처리 패턴과 같음)

```
승인 서비스 @Transactional
  ├─ approvals 상태 변경, record_history 추가
  ├─ notifications 1건 + notification_recipients N건 INSERT   ← 같은 트랜잭션 (승인되면 알림도 반드시 저장)
  └─ 이벤트 발행
        ↓ @TransactionalEventListener(AFTER_COMMIT) + @Async
     SSE로 접속 중인 사용자에게 푸시 → delivered_at 기록
        ↓ 접속 안 했거나 푸시 실패
     다음 접속 시 Last-Event-ID(= recipients.id) 이후 것을 보내 줌. 폴링(30초 배지 조회)도 백업으로 둠
```

- 알림 저장이 업무 트랜잭션 안에 있다. 그래서 "승인은 됐는데 알림이 사라지는" 일은 없다. 푸시는 실패해도 다시 받으면 된다.
- 푸시를 커밋 후에 보낸다. 그래서 롤백된 승인의 알림이 나가는 일도 없다.
- API 예: `GET /api/v1/notifications/stream`(SSE), `GET /api/v1/notifications?cursor=`, `GET /api/v1/notifications/unread-count`, `PATCH /api/v1/notifications/{id}/read`, `PATCH .../{id}/dismiss`, `POST .../read-all {lastId}`, `POST .../dismiss-all {lastId}`
- **보관·정리 (매일 새벽 배치)**:
  1. X를 누른 지 30일이 지난 수신 행 삭제: `DELETE FROM notification_recipients WHERE dismissed_at < NOW(6) - INTERVAL 30 DAY LIMIT 1000` (반복, 색인 `idx_nr_cleanup`)
  2. 받는 사람이 하나도 안 남은 알림 내용 삭제: `DELETE n FROM notifications n LEFT JOIN notification_recipients r ON r.notification_id = n.id WHERE r.id IS NULL AND n.created_at < NOW(6) - INTERVAL 30 DAY`
  - X를 누르지 않은 알림은 남는다. 만료 시각(`expires_at`)이 있는 알림은 화면에서 자동으로 안 보인다.
  - 지워도 되는 이유: 알림은 감사 증거가 아니다. 증거는 `record_history`에 따로 있다.

**"5분 전", "1시간 전" 표시** (컬럼 추가 없음)

- DB에는 **도착 시각(`notification_recipients.created_at`)만** 저장한다. "N분 전"은 화면이 계산한다.
  - "5분 전"을 저장하면 1분 뒤에는 틀린 값이 된다.
- 서버 응답: 시각은 시간대를 붙인 ISO 형식(`2026-10-26T23:14:05.123+09:00`), 목록 응답에 `serverTime`을 같이 준다.
- 화면 계산: `경과 = serverTime - createdAt`. 브라우저 시계 대신 서버 시각을 기준으로 해야 PC 시계가 틀려도 "-3분 전" 같은 값이 안 나온다.
- 표시 규칙:

| 경과 | 표시 |
| --- | --- |
| 1분 미만 | 방금 전 |
| 1~59분 | N분 전 |
| 1~23시간 | N시간 전 |
| 어제 | 어제 오후 3:20 |
| 그 이전 | 10월 24일 (올해가 아니면 2025년 10월 24일) |

- 목록이 열려 있으면 1분마다 다시 계산한다(서버 재조회 없이). 마우스를 올리면 정확한 시각(`2026-10-26 23:14`)을 보여준다.

## 6. 회원과 관리자

- **관리자는 별도 테이블 없이 등급으로 구분한다.** 회사 전체 관리자는 `users.role = 'ADMIN'`, 특정 현장만 관리하는 사람은 `site_members.role = 'SITE_ADMIN'` (3-0장).
- **가입 흐름**: 회원가입하면 `PENDING`으로 시작한다. ADMIN이 승인(ACTIVE)하고 현장 권한을 주면 드롭다운에 현장이 나타난다.
  - **로그인 서비스는 `status = 'ACTIVE'`만 통과시켜야 한다.** 기존 AuthService에 이 조건을 추가해야 한다.
  - 기존 미니 프로젝트 사용자도 V2 적용 후 PENDING이 된다.
- **본인 수정**: 이름, 전화번호(숫자만, `01012345678`), 소속, 직책, 비밀번호(현재 비밀번호 확인).
- **관리자만 수정**: 등급, 현장 권한, 상태. 바꿀 때마다 `record_history`에 `ROLE_CHANGE` / `GRANT` / `REVOKE` / `STATUS_CHANGE`로 남긴다.
- **로그인 잠금**: 실패하면 `failed_login_count`를 올리고, 5회가 되면 `LOCKED`로 바꾼다. 성공하면 0으로 되돌린다.

**로그인 화면 구성**

| 요소 | 동작 |
| --- | --- |
| 이메일·비밀번호·로그인 | 실패 메시지는 "이메일 또는 비밀번호가 맞지 않습니다" 하나로 통일 (어느 쪽이 틀렸는지 알려주지 않음) |
| 상태별 안내 | PENDING: "관리자 승인 대기 중", LOCKED: "5회 실패로 잠김. 비밀번호 찾기 또는 관리자 문의", INACTIVE: "사용이 중지된 계정" |
| 비밀번호를 잊으셨나요? | 메일 서버 설정이 있으면 이메일 입력 → 재설정 링크. 없으면(폐쇄망) "관리자에게 초기화를 요청하세요" + 관리자 연락처 |
| 회원가입 | 가입 → **인증 메일**(72시간) → 링크 클릭 → "관리자 승인 후 사용할 수 있습니다" 안내. 인증 메일 다시 보내기 버튼 |
| 비밀번호 변경 강제 | 임시 비밀번호로 로그인하면(`must_change_password`) 다른 화면으로 못 가고 변경 화면만 |

**이메일 인증 (아이디 = 이메일)**

- 오타 난 주소로 가입하면 나중에 비밀번호 재설정 메일을 받을 수 없다. 그래서 **이메일 소유가 확인된 계정만 사용 승인(ACTIVE)할 수 있게** DB가 막는다(`ck_users_active_verified`).
- 인증 방법은 두 가지다.
  - `LINK`: 인증 메일의 링크 클릭
  - `ADMIN`: 메일이 막힌 환경에서 관리자가 대면으로 확인 후 처리. 처리할 때 이력을 남긴다.
- 관리자 승인 화면에는 "이메일 인증됨 / 안 됨"이 표시된다. 인증 안 된 사람은 승인 버튼이 비활성화된다.
- **이메일 변경**: 새 주소로 확인 메일(`EMAIL_CHANGE`)을 보내고, 링크를 눌러야 실제로 바뀐다. 그 전까지는 옛 주소로 로그인한다. 바뀌면 옛 주소로 "아이디가 변경되었습니다" 메일을 보낸다(탈취 대비).
- 첫 ADMIN은 기동 시 환경 변수로 만들면서 `email_verified_via = 'ADMIN'`으로 넣는다.

**비밀번호 찾기·초기화**

| 방법 | 언제 | 흐름 |
| --- | --- | --- |
| 관리자 초기화 (예비) | 항상. 메일이 막힌 망 분리 현장의 유일한 방법 | 관리자 화면 [비밀번호 초기화] → 임시 비밀번호 1회 표시 → `must_change_password = TRUE` → 사용자가 첫 로그인에서 변경 |
| **이메일 링크 (기본)** | 메일 발송이 되는 환경 (인터넷 또는 사내 메일 서버 연결, `app.mail.enabled`) | 이메일 입력 → `account_tokens`(PASSWORD_RESET, 30분, 해시만 저장) → 링크 클릭 → 새 비밀번호 → 토큰 `used_at` |

- **공통 처리** (한 트랜잭션):
  - 비밀번호 해시 교체, `password_changed_at` 기록, `password_history` 추가(직전 3개 재사용 금지)
  - 그 사용자의 리프레시 토큰을 전부 폐기해서 다른 기기의 로그인도 끊는다
  - `failed_login_count = 0`, LOCKED였으면 ACTIVE로 되돌린다
  - `record_history`에 `PASSWORD_RESET` / `PASSWORD_CHANGE` / `UNLOCK`을 남긴다
  - 알림(`ACCOUNT`) "비밀번호가 변경되었습니다"
- **이메일 링크 요청 화면은 가입된 이메일인지 알려주지 않는다.** 항상 "가입된 이메일이면 메일을 보냈습니다"라고만 답한다. 같은 IP는 1시간에 5번까지만 받는다.
- **DB가 막는 것** (`account_tokens`):
  - 사용자·목적마다 쓸 수 있는 토큰은 하나뿐이다. 새로 요청하면 이전 토큰을 폐기해야 들어간다.
  - 유효시간은 비밀번호 재설정 최대 1시간, 이메일 인증·변경 최대 72시간이다.
  - 토큰 원문은 저장할 수 없고, 이메일 변경 토큰은 새 주소가 필수다.
  - 폐기된 토큰은 사용 처리할 수 없다.
- INACTIVE(퇴사·정지) 계정은 초기화할 수 없다(앱). 관리자 본인 계정은 다른 ADMIN만 초기화할 수 있다. ADMIN이 1명뿐이라 잠기면 서버에서 `ADMIN_EMAIL` / `ADMIN_PASSWORD` 환경 변수로 재설정하는 비상 절차를 쓴다.
- 비밀번호 규칙(앱): 10자 이상, 영문·숫자 포함, 이메일·이름 포함 금지.
- **첫 관리자**: 비밀번호를 BCrypt로 해시해야 해서 SQL로는 넣을 수 없다. Spring 기동 시 `ADMIN_EMAIL`, `ADMIN_PASSWORD`가 있고 ADMIN이 하나도 없으면 ADMIN을 만든다. 설치하는 회사마다 이 값으로 첫 관리자가 생긴다.
- **개발용 계정**: 현장 2곳, ADMIN 1명, 현장별 WRITER·APPROVER, 작업자, 작업구역은 D의 시드 스크립트로 넣는다. 마이그레이션에는 넣지 않는다.
- **개인정보 열람 기록**: 관리자가 사용자·작업자 전화번호를 펼쳐 보거나 감사 자료를 내려받으면 `record_access_logs`에 남긴다(수정·삭제 불가).

## 7. 규칙 데이터 (C)

- 지금은 원문으로 확인한 **밀폐공간 적정공기 4개만** 넣었다. 근거는 산업안전보건기준에 관한 규칙 제618조다.
  - 산소 18% 이상 23.5% 미만
  - 탄산가스 1.5% 미만
  - 일산화탄소 30ppm 미만
  - 황화수소 10ppm 미만
- 화기·고소·측정 누락·감시인·동시작업 규칙은 1주차에 원문을 확인한 뒤 `R__rules_seed.sql`에 추가한다. 파일 끝 TODO에 목록이 있다.
- **규칙 내용을 바꾸는 순서** (같은 파일 안에서):
  1. 이전 판 `enabled = FALSE`
  2. `version`을 올린 새 행 INSERT
  - 켜진 판이 둘이 되면 UNIQUE가 막는다.

## 8. Qdrant (벡터 DB)

| 컬렉션 | 내용 | 도구 |
| --- | --- | --- |
| kb_accidents | 재해조사보고서, SIF 아카이브, 사내 아차사고 | search_accidents |
| kb_guides | KOSHA GUIDE, 위험성평가 안내서, 체크리스트, 법령 조항 | search_guides |

- **벡터:** 점 하나에 `dense`(의미)와 `sparse`(키워드, IDF)를 둔다. 하이브리드 검색용이다.
- **payload 색인:** 5축, source_type, site_id, kb_source_id, kb_chunk_id. 전체 목록은 스크립트 상단 주석에 있다.
- **현장 구분:** 공공 데이터는 `site_id=0`(지식 DB에서는 NULL), 사내 데이터는 현장 id다. 검색할 때는 `site_id in [0, 내 현장]`으로 거른다. 현장 id는 Spring이 넘겨준 값만 쓴다.
- **점 id:** `uuid5(source_type:external_id:chunk_index)`. 지식 DB `kb_chunks.qdrant_point_id`와 업무 DB `evidence_citations.kb_point_id`가 같은 값이다. 양쪽 다 uuid5 형식 CHECK가 걸려 있다.
- **flagged 제외:** 검색 필터에 항상 검토 안 된 의심 원본 제외 조건을 넣는다(적재 시 payload에 `flagged`도 기록).
- **지식 DB와 Qdrant 일관성:**
  - 적재 순서: 지식 DB에 청크 저장 → Qdrant upsert → 성공하면 `kb_sources.status = 'EMBEDDED'`.
  - 중간에 실패하면 PARSED에 머문다. 같은 id로 다시 돌리면 덮어쓰기라 중복이 생기지 않는다.
- **임베딩 모델 권장:** bge-m3 하나로 OPEN·CLOSED 공용(1024차원). 모델을 바꾸면 `EMBEDDING_DIM`을 맞추고 `--recreate`한다.

```bash
QDRANT_API_KEY=... python -m scripts.init_qdrant --check    # 만들기 + 5축 필터 검색 점검 (쓰기 키 필요)
```

## 9. 기존 프로젝트에 적용

**① `backend/build.gradle`**

```gradle
implementation 'org.springframework.boot:spring-boot-starter-flyway'
implementation 'org.flywaydb:flyway-mysql'
```

**② `application.yaml`**

```yaml
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: true   # 기존 테이블이 있는 DB는 V1을 건너뛰고 V2부터
    baseline-version: 1
    validate-on-migrate: true   # 적용된 파일이 바뀌었으면 기동 실패 (몰래 고치기 방지)
  jpa:
    hibernate:
      ddl-auto: validate
    properties:
      hibernate.jdbc.time_zone: Asia/Seoul
  datasource:
    url: jdbc:mysql://${MYSQL_HOST}:3306/${MYSQL_DATABASE}?connectionTimeZone=Asia/Seoul&forceConnectionTimeZoneToSession=true
```

**③ `infra/docker-compose.yml`**

```yaml
  ocr-mysql:
    image: mysql:8.4.11-oraclelinux9
    command:
      - --log-bin-trust-function-creators=1   # V9 트리거 생성에 필요 (없으면 Flyway 실패)
      - --default-time-zone=+09:00            # 시간은 전부 한국 시간으로 저장
    environment:
      TZ: Asia/Seoul
      KB_OWNER_PASSWORD: ${KB_OWNER_PASSWORD}
      KB_INGEST_PASSWORD: ${KB_INGEST_PASSWORD}
      KB_READER_PASSWORD: ${KB_READER_PASSWORD}
    volumes:
      - mysql-data:/var/lib/mysql
      - ./db/init:/docker-entrypoint-initdb.d:ro   # 10-agent-kb.sh (schema.sql 마운트는 삭제)

  # 지식 DB 마이그레이션 (한 번 실행하고 종료). MySQL 드라이버는 라이선스 문제로 이미지에 없어서 직접 넣는다
  agent-kb-migrate:
    image: flyway/flyway:10
    command: -url=jdbc:mysql://ocr-mysql:3306/agent_kb -user=kb_owner -password=${KB_OWNER_PASSWORD} -validateOnMigrate=true migrate
    volumes:
      - ../ocr_api/db/migration:/flyway/sql:ro
      - ./flyway-drivers:/flyway/drivers:ro        # mysql-connector-j-x.y.z.jar 를 이 폴더에 둔다
    depends_on:
      ocr-mysql:
        condition: service_healthy

  qdrant:
    image: qdrant/qdrant:v1.12.4
    environment:
      QDRANT__SERVICE__API_KEY: ${QDRANT_API_KEY}
      QDRANT__SERVICE__READ_ONLY_API_KEY: ${QDRANT_READ_ONLY_API_KEY}
    volumes:
      - qdrant-data:/qdrant/storage
    # ports는 열지 않는다 (내부 네트워크 전용). 로컬 확인이 필요할 때만 잠깐 연다

  agent-model:            # 에이전트 실행: 읽기 전용 자격만
    environment:
      QDRANT_URL: http://qdrant:6333
      QDRANT_API_KEY: ${QDRANT_READ_ONLY_API_KEY}
      KB_DB_URL: mysql+pymysql://kb_reader:${KB_READER_PASSWORD}@ocr-mysql:3306/agent_kb
      EMBEDDING_DIM: "1024"
    depends_on:
      agent-kb-migrate:
        condition: service_completed_successfully
      qdrant:
        condition: service_started

  kb-ingest:              # 적재: 같은 이미지, 진입점만 다름. 쓰기 자격은 여기에만
    build: ../ocr_api
    command: uvicorn app.ingest_main:app --host 0.0.0.0 --port 8001
    environment:
      QDRANT_URL: http://qdrant:6333
      QDRANT_API_KEY: ${QDRANT_API_KEY}
      KB_DB_URL: mysql+pymysql://kb_ingest:${KB_INGEST_PASSWORD}@ocr-mysql:3306/agent_kb
      EMBEDDING_DIM: "1024"
    depends_on:
      agent-kb-migrate:
        condition: service_completed_successfully

volumes:
  qdrant-data:
```

- `.env.example`에 `KB_OWNER_PASSWORD`, `KB_INGEST_PASSWORD`, `KB_READER_PASSWORD`, `QDRANT_API_KEY`, `QDRANT_READ_ONLY_API_KEY`를 추가한다. 값은 서로 다르게 둔다.
- 이미 MySQL 볼륨이 있으면 초기화 스크립트가 자동으로 돌지 않는다. 한 번 직접 실행한다:
  `docker compose exec ocr-mysql bash /docker-entrypoint-initdb.d/10-agent-kb.sh`
- Spring은 `agent_kb`를 모르고, `agent-model`·`kb-ingest`는 업무 DB 접속 정보를 받지 않는다.

**④ `ocr_api/requirements.txt`**: `qdrant-client==1.12.1`, `pymysql` (지식 DB 접속)

**⑤ JPA 엔티티**

- JSON: `@JdbcTypeCode(SqlTypes.JSON)`
- 생성 컬럼(`risk_score`, `residual_score`, `accident_axis`, `pending_key`, `code_axis`, `enabled_key`, `signer_key`, `scope_key`, `active_key`, `existing_key`, `account_tokens.active_key`): `@Column(insertable = false, updatable = false)`
- 트리거 거부(SQLSTATE 45000)는 메시지가 한국어 문장이다. 공통 예외 처리에서 409(상태 충돌)로 바꾸고 메시지를 그대로 화면에 보여주면 된다.
- `lock_version`: `@Version`
- 상태·유형: `@Enumerated(EnumType.STRING)` + 4장 코드표와 같은 enum
- FK는 `@ManyToOne(fetch = LAZY)`. 삭제 cascade를 JPA에 걸지 않는다 (`CascadeType.REMOVE` 금지).
- CHECK·트리거 위반은 `DataIntegrityViolationException`으로 올라온다. 공통 예외 처리에서 400(입력 오류)이나 409(충돌)로 바꿔 응답 형식 `{success:false, error}`에 담는다.

## 10. 이후 스키마 변경 규칙 (4명 공통)

- 이미 적용된 V 파일은 **절대 고치지 않는다.** 바꿀 게 있으면 새 파일을 만든다. 고치면 `validate-on-migrate`가 기동을 막는다.
- **업무 DB는 V10부터 날짜 버전**을 쓴다. 예: `V2026_11_02_1__add_permit_photo.sql`. 번호 충돌을 막기 위해서다.
- 지식 DB(`agent/db/migration`)는 FastAPI 쪽 담당(A)이 관리하고 같은 규칙을 따른다. 업무 DB와 번호는 따로 간다.
- 한쪽 DB에 다른 서비스의 테이블을 만들지 않는다. 필요하면 API를 추가한다.
- **새 테이블 체크리스트** (PR에 C 리뷰 필수):
  - [ ] 모든 상태·유형 컬럼에 CHECK, Java enum과 같은 값
  - [ ] 업무 테이블이면 `site_id` + 같은 현장 복합 FK (참조하는 쪽 부모에 `UNIQUE (id, site_id)`). 작업자는 `worker_sites`를 참조
  - [ ] 승인 후에도 바뀌어야 하는 것은 본문을 고치지 말고 이벤트·새 판으로 (permit_events, revision)
  - [ ] FK는 기본 삭제 금지, CASCADE는 순수 자식만
  - [ ] 여러 사람이 고치면 `lock_version`
  - [ ] 상태와 짝 값(완료 시각 등) CHECK
  - [ ] `created_at`, 수정되는 테이블이면 `updated_at`
  - [ ] 위반 케이스를 `negative_tests.sh`에 추가
- 시연용 가상 한 달치 데이터는 별도 시드 스크립트로 넣는다(D). 시드도 이 제약을 다 통과해야 한다. 통과하지 못하면 시드가 틀린 것이다.

## 11. 검증

```bash
bash verify/verify.sh
```

1. 빈 MySQL 8.4를 띄우면 `10-agent-kb.sh`가 자동 실행돼 지식 DB와 계정이 만들어진다.
   - 업무 DB에는 **root가 아닌 앱 계정으로** V1~V9, R__를 적용한다. 트리거 권한 문제를 실제와 같은 조건에서 미리 잡기 위해서다.
   - 지식 DB에는 `kb_owner`로 적용한다.
2. R__는 두 번 적용해 반복 실행해도 문제없는지 확인한다.
3. 정상 흐름(`smoke_test.sql`)을 돌린다.
   - 현장 드롭다운: 두 현장 권한이 있는 작성자에게 2곳이 보이는지
   - 시나리오 A(대화·되묻기·작업 배치·취급 물질), B(산소 17.5% → BLOCK → 재측정 통과 → 2단계 결재 → 작업 중 측정 → 2시간 연장), C(아차사고 → 경고 → 확인 → 개선 조치), D(TBM이 다룬 작업·참석·서명 → 알림 → X로 지우기)
   - 위험성평가: AI 초안 → 임시저장(늦게 온 옛 요청이 새 내용을 못 덮는지) → 판 확인 저장 → 제출 → PDF 출력 두 번 클릭해도 작업 1개 → 문서 기록 → 완료 알림
4. **위반 케이스 197개**(`negative_tests.sh`)를 돌린다. 잘못된 값·권한 밖 접근 155개는 거부되고, 정상 동작 42개는 통과해야 한다.
   - 권한 테스트는 오류 번호까지 확인한다. 접속 실패처럼 엉뚱한 이유로 거부돼도 "통과"로 세지 않는다.
   - 하나라도 어긋나면 실패로 끝난다.
