# 5축 분류 및 RAG 필터 구현 계획

- 작성일: 2026-11-16
- 기준: [5축 분류 및 RAG 필터 요구사항 답변](5-axis-tagging-answers.md)
- 상태: 공공 Python 태깅/검증 경로 1차 구현 완료, 원자료 매핑·운영 검증 및 Q3/Q6 계약 최종 확인 필요
- 목표: 공공 사고보고서·SIF·가이드 자료를 5축 분류 기준에 따라 색인하고, `화기 + 인화성 물질 + 아차사고 자료` 같은 조건으로 검색한다.

## 확정 범위

이번 단계의 태깅 및 RAG 검색 대상은 **공공 자료**다. 사내 아차사고(`NEAR_MISS`)는 RAG 색인과 자동 태깅 대상에서 제외한다. 현재 main-server의 보고, 목록, 사람의 수동 태그 편집, `/incident-search` SQL 검색, 같은 구역 경고를 유지하고 `APP_KB_INGEST_CLIENT=none`도 그대로 둔다. 사내 아차사고 RAG 연동과 자동 태깅은 별도 후속 단계로 둔다.

공공 자료 범위는 사고보고서·SIF·가이드다. 단, 모든 분류축이 모든 문서 종류에 적용되는 것은 아니다. 예를 들어 가이드에는 심각도나 발생형태가 해당하지 않을 수 있으므로, 축별 `미확인`과 `해당 없음`을 구분한다.

| 결정 항목 | 회신 반영 |
| --- | --- |
| 태깅 대상 | 공공 사고보고서·SIF·가이드. 사내 아차사고는 이번 RAG/자동 태깅에서 제외 |
| 코드 | 영문 enum 유지. 화면 라벨·동의어·출처별 매핑은 별도 메타데이터 |
| 발생형태 | 산소결핍은 `ASPHYXIATION`에 포함. 추락/떨어짐, 맞음/부딪힘, 누출/접촉 등은 정한 동의어를 같은 코드에 매핑 |
| 작업유형 | AI 분류용 `tag_codes` 6종(GENERAL, HOT, CONFINED, HEIGHT, ELECTRICAL, LIFTING)을 유지. `works.work_type` 4종은 변경하지 않음 |
| CAS/GHS | 물질별 CAS에 정규화 GHS 분류와 출처를 연결 |
| 자동 태깅 | 공공 자료는 결정적 매핑 및 검증으로 확정. 출처와 태깅 규칙 버전을 기록 |
| 누락 | 저장을 차단하지 않음. 값 있음/미확인/해당 없음 구분, 빈 배열은 해당 없음으로 해석하지 않음 |
| 태그 기준 원본 | 원본 문서/데이터의 태그를 정본으로 하고, 필요한 태그를 청크 payload에 복사. 재청크 시 원본 ID·태그 버전 유지 |
| RAG 검색 대상 | `site_id=0` 공공 자료만. 사내 기록 검색은 이번 단계 제외 |
| 필터 의미 | 한 축 안의 여러 값은 OR, 축 간은 AND, 여러 `source_type`은 OR. 결과가 적어도 자동 완화하지 않음 |
| 과거 데이터 | 공공 청크만 dry-run 집계 → 검토·백업 → 배치 재색인 → 전후 비교 |
| 서버/UI/인프라 | main-server, main-vite-web, main-infra-manage 변경 없음. 사내 아차사고 KB 동기화 기본값 `none` 유지 |

## 미결 세부 계약

### Q3. SIF 가능성 값과 근거

**구현 초안:** 심각도와 분리된 `sif_potential` enum을 둔다. 허용 값은 `POTENTIAL`, `PRECURSOR`, `NOT_POTENTIAL`이고 미확인은 배열형 축과 일관되게 `null` + `tag_status.sif_potential="UNKNOWN"`으로 표현한다. 공단 SIF 원자료의 명시 필드가 각 값으로 직접 매핑되는 경우에만 자동 입력한다.

현재 Python SIF 프로필의 `tier` A/B/C/X는 자료 우선순위/선별 등급이며 SIF 가능성 값으로 재사용하지 않는다. 현재 `severity_of()`도 사망 여부만 분류하므로 SIF 가능성을 대신하지 않는다. 확인한 SIF 원자료 구조에는 SIF 가능성의 명시 필드가 없어 현재 값은 `null`/UNKNOWN이다. `POTENTIAL`/`PRECURSOR`의 실제 원자료 필드명·코드 매핑과 이 초안의 최종 사용 승인은 미완료다.

**남은 확인:** “명시적인 원자료 값만 자동 매핑, 그 외 UNKNOWN” 정책 및 필드명 `sif_potential`을 최종 승인한다. 코드는 우선 이 계약을 반영했다.

### Q6. 미확인·해당 없음 표현

**구현 초안:** 실제 필터 값과 상태 메타데이터를 분리한다. payload는 아래 형태를 따른다.

```json
{
  "severity": [],
  "accident_type": ["FIRE"],
  "cas": [],
  "ghs": [],
  "tag_status": {
    "severity": "UNKNOWN",
    "accident_type": "KNOWN",
    "hazard": "NOT_APPLICABLE",
    "work_type": "UNKNOWN",
    "substance": "UNKNOWN",
    "ghs": "UNKNOWN",
    "sif_potential": "UNKNOWN"
  }
}
```

상태는 `KNOWN`, `UNKNOWN`, `NOT_APPLICABLE`로 구분한다. `KNOWN`은 해당 배열에 검증된 값이 하나 이상 있음을 뜻한다. `UNKNOWN`은 자료만으로 판단할 수 없음을, `NOT_APPLICABLE`은 문서 종류상 그 축이 적용되지 않음을 뜻한다. `NOT_APPLICABLE`이나 `UNKNOWN`을 `severity`, `hazard`, `ghs` 등 검색 태그 배열에 넣지 않는다. 이렇게 하면 기존 축 필터가 실제 분류값만 검색하고, 품질 점검에서는 누락 이유를 구분할 수 있다.

Python 태깅, schema, ingest payload, 로컬 벡터 및 SIF 검증 경로에 이 표현을 반영했다. tag status 자체의 Qdrant 검색 인덱스는 추가하지 않았으며, 이는 품질 상태를 조회 필터로 쓰기로 결정할 때 별도 작업이다.

**남은 확인:** `tag_status` 객체와 `KNOWN`/`UNKNOWN`/`NOT_APPLICABLE` 세 상태를 최종 승인한다.

## 기존 구현과 주의점

- Python에는 SIF의 `accident_type`, `hazard`, `work_type`, `severity`를 결정적으로 만드는 코드와 `tag_codes_v5.json` 검증이 있다. 새 `substances: [{cas, ghs, source}]` 관계와 flat `cas`/`ghs` 배열 일치, CAS 체크섬, GHS 코드 형태 및 출처를 검증하고 ingest payload까지 전달한다. 하지만 현재 확인된 SIF 원자료에는 구조화 CAS/GHS가 없어 값은 비어 있으며, 실제 출처 매핑과 권위 있는 GHS 정규화/동의어 사전은 미완성이다.
- 법령 청크의 `hazard`/`work_type` 태깅은 설정상 기본 비활성이다. 이번 범위에서 가이드 자료의 어떤 축을 태깅할지, 어떤 축을 `NOT_APPLICABLE`로 둘지별 규칙을 정한다.
- SIF 태깅 규칙 버전과 출처, LAW 계층 태깅의 버전과 설정 상태를 payload에 기록하고, 벡터 파일/ingest validation까지 전달하도록 구현했다. 이 값들이 배포된 전체 공공 컬렉션에 반영됐는지는 재색인 후 검증해야 한다.
- `tag_codes_v5.json`은 main-server V5 값의 Python 사본이다. main-server 스키마는 변경하지 않더라도 공공 자료의 Python 코드·검증은 회신에서 합의한 기준과 일치하도록 관리해야 한다.
- Qdrant의 `source_type`은 같은 필드에서 여러 값을 OR로 처리하고, 서로 다른 조건은 AND로 묶는다. Spring `/incident-search`는 별도 사내 아차사고 SQL 경로이며 `source_type` 조건을 받지 않는다. 이번 단계의 공공 RAG 필터 계약과 혼합해서 설명하지 않는다.
- 이번 단계는 공공 자료이므로 `site_id=0` 필터를 유지한다. 사내 아차사고 검색을 다시 열 때는 site ID를 클라이언트 입력만으로 신뢰하지 않고, Spring이 확인한 권한 범위와 검색을 결합한다.

## 저장소별 작업

| 저장소 | 이번 단계 작업 |
| --- | --- |
| main-ai-model | 태깅 metadata/status/provenance, SIF enum 계약, CAS-GHS 연결 검증 및 payload 전달은 1차 구현. 공공 출처 매핑·GHS 정규화 사전·재색인·Qdrant 검증은 남음 |
| main-server | 변경 없음. 기존 아차사고 태그·SQL 검색·경고·KB 동기화 코드는 보존하되 기본 KB ingest 설정 `none` 유지 |
| main-vite-web | 변경 없음 |
| main-infra-manage | 변경 없음. 기존 공공 ingest 서비스와 키 설정 사용, `APP_KB_INGEST_CLIENT=none` 유지 |

## 실행 순서와 완료 기준

1. Q3 SIF 상태와 Q6 `tag_status` 표현의 구현 초안을 담당자와 최종 확정한다.
2. SIF/공공 사고/가이드별 source-to-code 매핑과 적용 불가 축을 문서화한다. SIF 가능성/CAS/GHS는 근거 자료가 확보된 축만 채운다.
3. 현재 구현한 청크의 정본 태그, `tagging_source`, `tagging_version`, `tag_status`를 실제 공공 원자료 표본과 대조하고 빠진 source mapping을 보완한다.
4. 로컬 dry-run에서 축별 분포, `UNKNOWN`/`NOT_APPLICABLE` 비율, 잘못된 코드 및 CAS/GHS 오류를 검토한다.
5. 기존 공공 Qdrant 점을 백업/기록한 뒤 안정적인 point ID로 배치 재색인한다. main-server DB나 사내 아차사고 자료는 수정하지 않는다.
6. 전체 point ID, payload, dense/sparse 벡터, `source_type`/5축 필터 조건을 검증하고 재색인 전후 건수·분포를 비교한다.

### 사용자가 직접 검증하는 방법

아래 명령은 `main-ai-model` 저장소 루트의 PowerShell에서 실행한다. Dockerfile과 맞는 Python 3.12.15 가상환경을 만들고 개발 의존성을 설치한다. `uv`가 Python을 아직 내려받지 않았다면 이 과정에서 설치한다.

```powershell
uv venv --python 3.12.15
.\.venv\Scripts\Activate.ps1
uv pip install -r requirements.dev.txt
```

이후 `python`은 활성화한 `.venv`의 인터프리터를 가리킨다. 가상환경 활성화가 PowerShell 실행 정책에 막히면 새 터미널에서 활성화를 허용하거나 `.venv\Scripts\python.exe -m pytest ...`처럼 가상환경 인터프리터를 직접 지정한다.

#### 1. 태깅·검증 테스트

```powershell
python -m pytest -q tests/test_accident_ingest.py tests/test_ingest_api.py tests/test_rag_configuration.py tests/test_rag_tags.py tests/test_verify_sif_search.py tests/test_vectorize_accidents.py
```

위 테스트는 SIF 태깅, ingest payload, 태그 상태/CAS-GHS 검증, RAG 필터 및 벡터화 경로를 확인한다. 전체 테스트도 실행하려면 `python -m pytest -q`를 사용한다. 현재 `tests/test_ingest_via_api.py`는 저장소에 없는 `samples/ingest_demo_sif.json`을 참조하므로, 이 파일 때문에 수집이 실패하면 해당 원인을 기록하고 focused 테스트 결과와 구분한다.

#### 2. 공공 SIF 자료 생성 및 검증

원본 폴더와 결과 폴더를 실제 경로로 바꾸어 실행한다. `run`은 원본부터 검증 보고서까지 생성하며, `validate`를 다시 실행해 결과를 재확인할 수 있다.

```powershell
$raw = "..\out\<dir-name>\raw"
$out = "..\out\sif\tagging-check"
python -m app.chunking.accident_ingest run --raw $raw --out $out --profile sif
python -m app.chunking.accident_ingest validate --out $out --profile sif
python -m app.chunking.accident_ingest show --out $out
```

보고서가 `통과`이고, 표본 및 생성 청크에서 원본 ID, 태그, `tag_status`, `tagging_source`, `tagging_version`을 확인할 수 있어야 한다. 실패 또는 경고가 있으면 해당 원본을 대조하고 조치 내용을 기록한다.

#### 3. 가이드 자료 생성 및 검증

가이드/법령 HWPX 파일과 결과 폴더를 실제 경로로 바꾼다. `--tags`를 사용하면 계층 기반 `hazard`/`work_type` 태깅을 활성화하므로, 생성과 검증 명령에 모두 같은 옵션을 전달한다.

```powershell
$lawFile = "C:\data\guides\guide.hwpx"
$lawOut = "C:\data\guides\tagging-check"
python -m app.law_ingest run $lawFile --out $lawOut --tags
python -m app.law_ingest validate --out $lawOut --tags
```

태깅을 사용하지 않는 기준 동작을 확인할 때는 두 명령 모두에서 `--tags`를 뺀다. 검증 보고서와 출력 청크에서 적용 축의 상태 및 `tagging_version`을 확인한다.

#### 4. 검색 필터 및 재색인 확인

운영 컬렉션이 아닌 별도 테스트 컬렉션에서 다음 조건으로 검색하고, 요청 조건과 반환 payload를 대조한다.

| 시험 | 통과 기준 |
| --- | --- |
| 같은 축에 값 여러 개 지정 | 지정한 값 중 하나와 일치하는 결과만 반환(OR) |
| 서로 다른 축에 조건 지정 | 지정한 모든 축 조건을 만족하는 결과만 반환(AND) |
| 여러 `source_type` 지정 | 지정한 유형 중 하나와 일치하는 결과만 반환(OR) |
| 공공 자료 범위 | 모든 결과가 `site_id=0`; 사내 자료가 포함되지 않음 |
| 결과가 적거나 없음 | 조건이 자동 완화되지 않음 |
| 재색인 재시도 | 같은 point ID가 갱신되고 중복 point가 생기지 않음 |

재색인 전후의 전체 point ID와 건수, payload 태그/상태/출처/버전, dense/sparse 벡터 존재 여부 및 축별 분포를 비교한다. 실행 명령, 테스트 컬렉션 식별자, 전후 수치, 실패 항목과 조치 결과를 검증 기록으로 남긴다. 백업과 dry-run 검토가 끝나기 전에는 운영 컬렉션에 재색인하지 않는다.

현재 확인: 회신에 맞춘 Python focused suite 357개 통과. `tests/test_ingest_via_api.py`는 저장소에 없는 `samples/ingest_demo_sif.json` 때문에 수집 불가했고, 실제 전체 공공 데이터 재색인/검색 검증은 아직 수행하지 않았다.

완료 조건은 공공 자료에서 코드 매핑과 상태 표현이 재현 가능하고, 같은 축 OR·축 간 AND 필터가 기대대로 동작하며, 재색인 실패를 동일 ID로 안전하게 재시도할 수 있는 것이다. Q3/Q6은 코드 초안이 존재하지만 담당자 확인 전에는 최종 정책으로 간주하지 않는다.
