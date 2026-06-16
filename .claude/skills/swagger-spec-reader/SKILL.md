---
name: swagger-spec-reader
description: Swagger 2.x 또는 OpenAPI 3.x JSON 파일을 파싱해 엔드포인트 목록, 파라미터, 요청/응답 스키마를 추출하는 가이드. swagger 파일 읽기, 엔드포인트 파악, 스키마 분석, $ref 해석, 예제값 생성 요청 시 사용.
---

## 버전 확인

```python
import json
with open('./swagger/pubsub_v1.json') as f:
    spec = json.load(f)
version = spec.get('swagger') or spec.get('openapi')  # "2.0" or "3.x.x"
```

## 엔드포인트 추출

```python
base_path = spec.get('basePath', '')  # Swagger 2.x
endpoints = []
for path, methods in spec.get('paths', {}).items():
    for method, op in methods.items():
        if method not in ['get','post','put','delete','patch']:
            continue
        endpoints.append({
            'method': method.upper(),
            'path': path,
            'fullPath': base_path + path,
            'summary': op.get('summary', op.get('operationId', '')),
            'parameters': op.get('parameters', []),
            'tags': op.get('tags', []),
            'operationId': op.get('operationId', ''),
        })
```

## $ref 해석

```python
def resolve_ref(schema, spec):
    if not schema or '$ref' not in schema:
        return schema
    # "#/definitions/TopicRequest" → spec['definitions']['TopicRequest']
    parts = schema['$ref'].replace('#/', '').split('/')
    obj = spec
    for p in parts:
        obj = obj.get(p, {})
    return obj
```

## Request Body 스키마 추출

```python
def get_body_schema(op, spec):
    # OpenAPI 3.x
    if 'requestBody' in op:
        content = op['requestBody'].get('content', {})
        mt = content.get('application/json') or next(iter(content.values()), None)
        return resolve_ref(mt.get('schema') if mt else None, spec)
    # Swagger 2.x body parameter
    body_param = next(
        (p for p in op.get('parameters', []) if p.get('in') == 'body'), None
    )
    return resolve_ref(body_param.get('schema') if body_param else None, spec)
```

## 예제값 생성

```python
def build_example(schema, spec, depth=0):
    if not schema or depth > 3:
        return {}
    schema = resolve_ref(schema, spec)
    if 'example' in schema:
        return schema['example']
    t = schema.get('type')
    if t == 'object' or 'properties' in schema:
        return {
            k: build_example(v, spec, depth + 1)
            for k, v in schema.get('properties', {}).items()
        }
    if t == 'array':
        return [build_example(schema.get('items', {}), spec, depth + 1)]
    if t == 'string':  return schema.get('example', 'string')
    if t in ('integer', 'number'): return schema.get('example', 0)
    if t == 'boolean': return False
    return None
```

## path 파라미터 추출

```python
import re
def get_path_params(path):
    # "/v1/domains/{domain}/projects/{project}/topics/{topic.name}"
    return re.findall(r'\{([^}]+)\}', path)
```

## 파일별 처리 예시

```python
SWAGGER_FILES = ['console_v1', 'console_v2', 'pubsub_v1', 'pubsub_v2']
results = {}
for name in SWAGGER_FILES:
    try:
        with open(f'./swagger/{name}.json') as f:
            spec = json.load(f)
        results[name] = {
            'spec': spec,
            'endpoints': extract_endpoints(spec),
        }
    except Exception as e:
        results[name] = {'error': str(e), 'endpoints': []}
```
