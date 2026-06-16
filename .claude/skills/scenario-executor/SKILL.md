---
name: scenario-executor
description: index.html의 시나리오 구조(setup/apis/teardown, assert/extract/inject)를 CLI에서 HTTP 호출로 실행하는 구체적인 방법을 제공한다. API 호출 실행, 시나리오 순서 처리, extract/inject 패턴, assert 검증 로직, IAM 토큰 발급 방법 요청 시 사용.
---

## 시나리오 구조 (index.html SCENARIOS 배열)

```json
{
  "id": "s2",
  "name": "토픽 생성·조회",
  "appliesTo": ["pubsub_v1", "pubsub_v2"],
  "setup": [],
  "apis": [
    {
      "method": "PUT",
      "pathIncludes": "topic",
      "extract": [{ "from": "body.name", "ctxKey": "topicName" }],
      "assert": [
        { "type": "statusRange", "min": 200, "max": 299 },
        { "type": "bodyExists", "path": "name" }
      ]
    },
    {
      "method": "GET",
      "pathIncludes": "topics/",
      "pathNotIncludes": "subscriptions",
      "inject": [{ "ctxKey": "topicName", "into": "path", "param": "topic.name" }],
      "assert": [{ "type": "status", "expected": 200 }]
    }
  ],
  "teardown": [
    {
      "method": "DELETE",
      "pathIncludes": "topics",
      "inject": [{ "ctxKey": "topicName", "into": "path", "param": "topic.name" }]
    }
  ]
}
```

## 엔드포인트 매칭 (matchScenarioApi)

시나리오 rule과 swagger 엔드포인트를 매칭할 때:

```python
def match_endpoint(ep, rule):
    if ep['method'] != rule['method']:
        return False
    path = ep['path']
    if rule.get('pathIncludes') and rule['pathIncludes'] not in path:
        return False
    if rule.get('pathIncludes2') and rule['pathIncludes2'] not in path:
        return False
    if rule.get('pathNotIncludes'):
        for n in rule['pathNotIncludes'].split('|'):
            if n in path:
                return False
    return True
```

## Path 파라미터 치환

```python
DOMAIN_ALIASES  = ['domain','domainId','domain_id','domainID',
                   'subscription.domain','topic.domain']
PROJECT_ALIASES = ['project','projectId','project_id','projectID',
                   'subscription.project','topic.project']

def resolve_path(path, domain_id, project_id, context):
    for a in DOMAIN_ALIASES:
        path = path.replace(f'{{{a}}}', domain_id)
    for a in PROJECT_ALIASES:
        path = path.replace(f'{{{a}}}', project_id)
    # inject rule에 의한 치환
    for key, val in context.items():
        path = path.replace(f'{{{key}}}', str(val))
    return path
```

## HTTP 실행 (Python requests)

```python
import requests, json, time

def call_api(method, url, headers, body_str=None):
    start = time.time()
    try:
        resp = requests.request(
            method, url,
            headers=headers,
            data=body_str,
            timeout=30,
            verify=True
        )
        return {
            'ok': 200 <= resp.status_code < 300,
            'status': resp.status_code,
            'time': int((time.time() - start) * 1000),
            'body': resp.text,
        }
    except Exception as e:
        return {
            'ok': False, 'status': None,
            'time': int((time.time() - start) * 1000),
            'error': str(e)
        }
```

## curl 대안 (bash)

```bash
RESPONSE=$(curl -s -w "\n%{http_code}" -X {METHOD} "{URL}" \
  -H "Domain-ID: {domain_id}" \
  -H "Project-ID: {project_id}" \
  -H "X-Auth-Token: {token}" \
  -H "Content-Type: application/json" \
  -d '{body}')
HTTP_CODE=$(echo "$RESPONSE" | tail -1)
BODY=$(echo "$RESPONSE" | head -n -1)
```

## IAM 토큰 발급

```bash
# 발급
TOKEN=$(curl -si -X POST "https://iam.kakaocloud-stg.com/identity/v3/auth/tokens" \
  -H 'Content-Type: application/json' \
  -d '{
    "auth": {
      "identity": {
        "methods": ["application_credential"],
        "application_credential": {
          "id": "{cred_id}",
          "secret": "{cred_secret}"
        }
      }
    }
  }' | grep -i 'x-subject-token' | awk '{print $2}' | tr -d '\r\n')
```

prod 환경: `https://iam.kakaocloud.com/identity/v3/auth/tokens`

## Assert 타입 참조

| type | 검증 내용 | 필드 |
|------|----------|------|
| `status` | `status == expected` | `expected` |
| `statusRange` | `min <= status <= max` | `min`, `max` |
| `bodyExists` | JSON path 값이 존재 | `path` |
| `bodyContains` | JSON path 값이 expected와 동일 | `path`, `expected` |
| `bodySchema` | requiredKeys가 모두 존재 | `requiredKeys[]` |
| `bodyArrayLength` | 배열 길이 범위 | `path`, `min`, `max` |
| `responseTime` | 응답시간 <= maxMs | `maxMs` |

## 시나리오 실행 순서 패턴

```python
context = {}
results = []
try:
    # 1. setup (결과 기록 없이 실행)
    for rule in scenario.get('setup', []):
        ep = find_endpoint(endpoints, rule)
        if ep:
            path, body = inject_context(rule.get('inject', []), ep['fullPath'], body_str, context)
            call_api(rule['method'], base_url + path, headers, body)

    # 2. apis (결과 기록)
    for rule in scenario.get('apis', []):
        ep = find_endpoint(endpoints, rule)
        if not ep:
            continue
        path, body = inject_context(rule.get('inject', []), ep['fullPath'], body_str, context)
        result = call_api(rule['method'], base_url + path, headers, body)
        assert_results = run_asserts(rule.get('assert', []), result)
        extract_context(rule.get('extract', []), result['body'], context)
        results.append({**result, 'assertResults': assert_results, 'pass': all(a['pass'] for a in assert_results)})

finally:
    # 3. teardown (실패 무시)
    for rule in scenario.get('teardown', []):
        try:
            ep = find_endpoint(endpoints, rule)
            if ep:
                path, body = inject_context(rule.get('inject', []), ep['fullPath'], None, context)
                call_api(rule['method'], base_url + path, headers, body)
        except:
            pass
```
