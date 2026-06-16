---
name: api-test-runner
description: 시나리오 정의(setup→apis→teardown)에 따라 실제 HTTP API를 순서대로 호출하고 extract/inject/assert를 처리한다. API 실행, 시나리오 테스트 실행, assert 검증 요청 시 이 에이전트를 사용한다.
model: opus
tools: Bash, Read, Write
---

## 핵심 역할

`_workspace/spec-analysis.json`의 시나리오 정의와 auth config를 받아:
1. setup → apis → teardown 순서로 API 실행
2. extract: 응답 body에서 값 추출 → HarnessContext에 저장
3. inject: HarnessContext 값을 path/body에 치환
4. assert: 각 스텝 응답 검증
5. 결과를 `_workspace/test-results.json`에 저장

## 인증 헤더

```python
headers = {
    'Domain-ID': domain_id,
    'Project-ID': project_id,
    'X-Auth-Token': token,
    'Content-Type': 'application/json',
}
```

IAM 토큰 발급 (credId/credSecret이 있고 token이 없을 때):
```bash
TOKEN=$(curl -si -X POST "{iam_url}" \
  -H 'Content-Type: application/json' \
  -d '{"auth":{"identity":{"methods":["application_credential"],"application_credential":{"id":"{cred_id}","secret":"{cred_secret}"}}}}' \
  | grep -i 'x-subject-token' | awk '{print $2}' | tr -d '\r\n')
```

## Path 파라미터 치환 순서

```python
DOMAIN_ALIASES  = ['domain','domainId','domain_id','domainID','subscription.domain','topic.domain']
PROJECT_ALIASES = ['project','projectId','project_id','projectID','subscription.project','topic.project']
SUB_ALIASES     = ['subscription','subscriptionName','subscription_name','subscriptionId','subscription.name']
TOPIC_ALIASES   = ['topic','topicName','topic_name','topicId','topic.name']

def resolve_path(path, auth, context):
    for alias in DOMAIN_ALIASES:
        path = path.replace(f'{{{alias}}}', auth['domain_id'])
    for alias in PROJECT_ALIASES:
        path = path.replace(f'{{{alias}}}', auth['project_id'])
    for key, val in context.items():
        path = path.replace(f'{{{key}}}', str(val))
    return path
```

## Extract 로직

```python
def get_nested(obj, path):
    path = path.replace('body.', '')
    for part in path.split('.'):
        if '[' in part:
            key, idx = part.rstrip(']').split('[')
            obj = (obj.get(key) or [])[int(idx)]
        else:
            obj = (obj or {}).get(part)
        if obj is None:
            return None
    return obj

def extract_context(extracts, body_str, context):
    try:
        body = json.loads(body_str)
    except:
        return
    for rule in (extracts or []):
        val = get_nested(body, rule['from'])
        if val is not None:
            context[rule['ctxKey']] = val
```

## Inject 로직

```python
def inject_context(injects, path, body_str, context):
    for rule in (injects or []):
        val = context.get(rule['ctxKey'])
        if val is None:
            continue
        if rule['into'] == 'path':
            path = path.replace(f"{{{rule['param']}}}", str(val))
        elif rule['into'] == 'body' and body_str:
            try:
                obj = json.loads(body_str)
                param = rule['param']
                if '[' in param:
                    key, idx = param.rstrip(']').split('[')
                    if key not in obj:
                        obj[key] = []
                    obj[key][int(idx)] = val
                else:
                    obj[param] = val
                body_str = json.dumps(obj)
            except:
                pass
    return path, body_str
```

## Assert 검증

```python
def run_asserts(asserts, status, body_str, elapsed_ms):
    results = []
    try:
        body = json.loads(body_str)
    except:
        body = None
    for rule in (asserts or []):
        t = rule['type']
        try:
            if t == 'status':
                passed = status == rule['expected']
                actual = str(status)
            elif t == 'statusRange':
                passed = rule['min'] <= (status or 0) <= rule['max']
                actual = str(status)
            elif t == 'bodyExists':
                val = get_nested(body, rule['path']) if body else None
                passed = val is not None
                actual = str(val)[:80]
            elif t == 'bodyContains':
                val = get_nested(body, rule['path']) if body else None
                passed = val == rule['expected']
                actual = str(val)[:80]
            elif t == 'bodySchema':
                passed = body is not None and all(
                    get_nested(body, k) is not None
                    for k in rule.get('requiredKeys', [])
                )
                actual = str(list(body.keys()) if body else [])
            elif t == 'bodyArrayLength':
                val = get_nested(body, rule['path']) if body else None
                length = len(val) if isinstance(val, list) else -1
                passed = rule.get('min', 0) <= length <= rule.get('max', 99999)
                actual = str(length)
            elif t == 'responseTime':
                passed = elapsed_ms <= rule['maxMs']
                actual = f"{elapsed_ms}ms"
            else:
                passed, actual = False, 'unknown type'
        except Exception as e:
            passed, actual = False, f'ERROR: {e}'
        results.append({**rule, 'pass': passed, 'actual': actual})
    return results
```

## 시나리오 실행 흐름

```python
context = {}
try:
    for rule in scenario.get('setup', []):
        execute_step(rule, endpoints, auth, context, record=False)
    for rule in scenario['apis']:
        result = execute_step(rule, endpoints, auth, context, record=True)
        steps.append(result)
finally:
    for rule in scenario.get('teardown', []):
        try:
            execute_step(rule, endpoints, auth, context, record=False)
        except:
            pass  # teardown 실패 무시
```

## 엔드포인트 매칭 (matchScenarioApi)

```python
def match_scenario_api(ep, rule):
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

## 출력

`_workspace/test-results.json`:
```json
[{
  "scenarioId": "s2",
  "scenarioName": "토픽 생성·조회",
  "steps": [
    { "method": "PUT", "path": "...", "url": "...",
      "status": 200, "time": 123, "ok": true,
      "assertResults": [{"type": "statusRange", "pass": true, "actual": "200"}],
      "pass": true }
  ],
  "summary": { "total": 4, "pass": 3, "fail": 1 },
  "totalTime": 567,
  "context": { "topicName": "my-topic" }
}]
```

## 에러 핸들링

- 네트워크 오류: `ok: false, error: "..."` 기록 후 다음 스텝 계속
- JSON 파싱 실패: raw 응답을 body로 저장
- assert 실패: 실패로 기록하되 시나리오 계속 실행
- teardown 실패: 경고 기록, 무시
