---
name: pubsub-test-orchestrator
description: Kakao Cloud Pub/Sub Swagger API 자동화 테스트를 처음부터 끝까지 실행한다. "테스트 실행", "시나리오 실행", "API 검증", "전체 실행", "s1 실행", "s2 실행", "QA 시나리오 돌려줘", "결과 리포트", "API 테스트해줘", "시나리오 돌려줘", "다시 실행", "재실행", "특정 시나리오만" 등 테스트 관련 모든 요청 시 반드시 이 스킬을 사용할 것. index.html의 시나리오 정의를 CLI에서 에이전트가 실행하고 결과를 리포트한다.
---

## 목적

index.html에 정의된 Pub/Sub API 시나리오(setup→apis→teardown, assert/extract/inject)를 브라우저 없이 CLI 에이전트가 실행한다.

**실행 모드:** 서브 에이전트 패턴 (단계별 순차 파이프라인)

---

## Phase 0: 컨텍스트 확인

```bash
ls _workspace/ 2>/dev/null && echo "exists" || echo "not found"
```

| `_workspace/` 상태 | 요청 내용 | 실행 모드 |
|-------------------|----------|----------|
| 없음 | 어떤 요청이든 | 초기 실행 — Phase 1부터 전체 |
| 있음 | "다시", "재실행", "특정 시나리오만" | 부분 재실행 — Phase 3부터 |
| 있음 | 새 auth 정보 제공 | 새 실행 — `_workspace/`를 `_workspace_prev/`로 이동 후 Phase 1 |

---

## Phase 1: 스펙 로딩

spec-analyst 에이전트를 호출해 Swagger 파일과 index.html 시나리오를 파싱한다.

```
Agent(
  description="Swagger 스펙 + index.html 시나리오 파싱",
  subagent_type="Explore",
  model="opus",
  prompt="""
  spec-analyst 에이전트 역할로 작업한다.
  [에이전트 정의 내용 삽입]

  작업 경로: /Users/kakao_ent/chloe_workspace/api_test_tool/pubsub-swagger-harness
  - ./swagger/console_v1.json, console_v2.json, pubsub_v1.json, pubsub_v2.json 파싱
  - ./index.html에서 SCENARIOS, API_META, ENVIRONMENTS 추출
  - 결과를 _workspace/spec-analysis.json에 저장
  """
)
```

완료 후 `_workspace/spec-analysis.json` 존재 확인.

---

## Phase 2: 인증 정보 수집

다음 순서로 auth config를 구성한다:

**1단계: 환경변수 확인**
```bash
echo "DOMAIN: $PUBSUB_DOMAIN_ID"
echo "PROJECT: $PUBSUB_PROJECT_ID"
echo "TOKEN: ${PUBSUB_TOKEN:0:10}..."
echo "BASE_URL: $PUBSUB_BASE_URL"
```

**2단계: 없으면 사용자에게 질문**
다음 정보를 요청한다:
- Domain-ID
- Project-ID
- Base URL (예: `https://pub-sub.kr-central-2.kakaocloud.com`)
- X-Auth-Token (있으면) 또는 Credential-ID + Credential-Secret

**3단계: IAM 토큰 자동 발급** (credId/credSecret이 있고 token이 없을 때)

환경별 IAM URL:
- dev/stage: `https://iam.kakaocloud-stg.com/identity/v3/auth/tokens`
- prod: `https://iam.kakaocloud.com/identity/v3/auth/tokens`

```bash
TOKEN=$(curl -si -X POST "{iam_url}" \
  -H 'Content-Type: application/json' \
  -d '{"auth":{"identity":{"methods":["application_credential"],"application_credential":{"id":"{cred_id}","secret":"{cred_secret}"}}}}' \
  | grep -i 'x-subject-token' | awk '{print $2}' | tr -d '\r\n')
echo "발급된 토큰: ${TOKEN:0:20}..."
```

---

## Phase 3: 시나리오 선택

사용자가 시나리오를 지정했으면 해당 ID만 실행. 없으면 목록 제시 후 선택 요청.

| ID | 이름 | 대상 파일 |
|----|------|----------|
| s1 | 서비스 상태 확인 | console_v1/v2 |
| s2 | 토픽 생성·조회 | pubsub_v1/v2 |
| s3 | 서브스크립션 생성·조회 | pubsub_v1/v2 |
| s4 | 메시지 E2E 플로우 | pubsub_v1/v2 |
| s5 | Ack 예외 케이스 | pubsub_v1/v2 |
| s6 | Seek 재처리 | pubsub_v1/v2 |
| s8 | 리소스 정리 | pubsub_v1/v2 |
| sq1 | [QA] 잘못된 파라미터 검증 | pubsub_v1/v2 |
| sq2 | [QA] 응답 스키마 검증 | pubsub_v1/v2 |
| sq3 | [QA] SLA 응답시간 검증 | console_v1/v2 |

**권장 실행 순서 (전체):** s1 → s2 → s3 → s4 → s5 → s6 → sq2 → sq3 → s8

---

## Phase 4: 테스트 실행

api-test-runner 에이전트를 서브 에이전트로 호출한다.

독립 시나리오(s1, sq1, sq2, sq3)는 `run_in_background=True`로 병렬 실행 가능.
상태 의존 시나리오(s2→s3→s4→s5→s6→s8)는 순차 실행.

```
Agent(
  description="시나리오 {id} 실행",
  subagent_type="general-purpose",
  model="opus",
  prompt="""
  api-test-runner 에이전트 역할로 작업한다.
  [에이전트 정의 내용 삽입]

  실행할 시나리오: {scenario_id}
  스펙 파일: _workspace/spec-analysis.json
  Auth: domain_id={domain_id}, project_id={project_id}, token={token}, base_url={base_url}
  결과를 _workspace/test-results.json에 누적 저장
  """
)
```

---

## Phase 5: 리포트 생성

result-reporter 에이전트를 호출한다.

```
Agent(
  description="테스트 결과 리포트 생성",
  subagent_type="general-purpose",
  model="opus",
  prompt="""
  result-reporter 에이전트 역할로 작업한다.
  [에이전트 정의 내용 삽입]

  결과 파일: _workspace/test-results.json
  환경: {env}, Base URL: {base_url}
  _workspace/report.md 생성 후 콘솔에 요약 출력
  """
)
```

완료 후 `_workspace/report.md` 경로를 사용자에게 안내한다.

---

## 에러 핸들링

| 오류 상황 | 처리 |
|----------|------|
| `./swagger/*.json` 없음 | swagger/ 폴더 파일 목록 확인 후 안내 |
| 인증 정보 없음 | 필요 항목 목록 출력 후 입력 요청 |
| IAM 토큰 발급 실패 | 오류 메시지 출력, X-Auth-Token 직접 입력 요청 |
| 시나리오 실행 중 스텝 실패 | 실패 기록 후 teardown 실행, 리포트에 포함 |
| 401 응답 | 토큰 만료 가능성 안내, 재발급 제안 |

---

## 테스트 시나리오

**정상 흐름:** s2(토픽 생성) → s3(서브스크립션 생성) → s4(메시지 E2E) → s8(리소스 정리) 순서 실행 시 모든 assert 통과
**에러 흐름:** sq1([QA] 잘못된 파라미터) 실행 → 404 응답 → assert `status == 404` pass 확인
