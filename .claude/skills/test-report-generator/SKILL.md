---
name: test-report-generator
description: API 테스트 결과 JSON을 읽어 Markdown 리포트를 생성한다. "결과 정리해줘", "리포트 만들어줘", "테스트 결과 분석", "어디서 실패했어", "결과 보여줘" 요청 시 사용.
---

## 리포트 템플릿

```markdown
# Pub/Sub API 테스트 리포트

**실행 일시:** 2026-06-15 14:30:00  
**환경:** stage | **Base URL:** https://pub-sub.kr-central-2.kakaocloud.com  

---

## 전체 요약

| 항목 | 값 |
|------|-----|
| 전체 스텝 | 12 |
| 성공 | ✅ 10 |
| 실패 | ❌ 1 |
| Assert 실패 | ⚠️ 1 |
| 총 소요시간 | 3,241ms |

---

## 시나리오별 결과

### 📤 시나리오 2. 토픽 생성·조회 — ✅ PASS (4/4)

| # | Method | 경로 | 상태 | 시간 | 결과 |
|---|--------|------|------|------|------|
| 1 | PUT | /v1/.../topics | 200 | 234ms | ✅ |
| 2 | GET | /v1/.../topics | 200 | 89ms | ✅ |
| 3 | GET | /v1/.../topics/my-topic | 200 | 102ms | ✅ |
| 4 | PATCH | /v1/.../topics/my-topic | 200 | 156ms | ✅ |

**컨텍스트 추출값:** `topicName = my-topic`

---

### 📥 시나리오 3. 서브스크립션 생성·조회 — ⚠️ PARTIAL (4/5)

| # | Method | 경로 | 상태 | 시간 | 결과 |
|---|--------|------|------|------|------|
| 1 | PUT | /v1/.../subscriptions | 200 | 198ms | ✅ |
| 2 | GET | /v1/.../subscriptions | 200 | 77ms | ✅ |
| 3 | GET | /v1/.../subscriptions/my-sub | 404 | 45ms | ❌ |

**Assert 상세 (Step 3):**
- ❌ status == 200 (실제: 404)

---

## 실패 상세

### ❌ 시나리오 3 — Step 3 GET /subscriptions/{subscription.name}

- **URL:** `https://.../subscriptions/my-sub`
- **응답 코드:** 404
- **응답 body:**
  ```json
  {"code": 5, "message": "subscription not found"}
  ```
- **실패 assert:** `status == 200` (실제: 404)
- **가능한 원인:** subscription.name 값이 올바르지 않거나 해당 리소스가 아직 생성되지 않음
- **권고:** Step 1(서브스크립션 생성)의 응답에서 name 값을 확인하세요

---

## 권고 사항

- ⚠️ SLA 초과 없음
- ⚠️ teardown 토픽 삭제 성공 — 리소스 정리 완료
- ❌ 시나리오 3 Step 3 실패 — 서브스크립션 이름 주입(inject) 값 확인 필요
```

## 생성 로직

```python
from datetime import datetime
import json

def generate_report(results_path, env, base_url, output_path='_workspace/report.md'):
    with open(results_path) as f:
        results = json.load(f)

    total = sum(len(r['steps']) for r in results)
    passed = sum(s['pass'] for r in results for s in r['steps'])
    failed = total - passed
    assert_fails = sum(
        1 for r in results for s in r['steps']
        if s.get('ok') and not s.get('pass')
    )
    total_time = sum(r.get('totalTime', 0) for r in results)

    lines = [
        f"# Pub/Sub API 테스트 리포트\n",
        f"**실행 일시:** {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}  ",
        f"**환경:** {env} | **Base URL:** {base_url}\n",
        "---\n## 전체 요약\n",
        "| 항목 | 값 |", "|------|-----|",
        f"| 전체 스텝 | {total} |",
        f"| 성공 | ✅ {passed} |",
        f"| 실패 | ❌ {failed} |",
        f"| Assert 실패 | ⚠️ {assert_fails} |",
        f"| 총 소요시간 | {total_time:,}ms |",
        "\n---\n## 시나리오별 결과\n",
    ]
    # 시나리오별 테이블 + 실패 상세 추가
    ...
    with open(output_path, 'w') as f:
        f.write('\n'.join(lines))
```

## 상태 아이콘 규칙

```python
def step_icon(step):
    if step.get('error'):       return '🔌'
    if not step.get('ok'):      return '❌'
    if not step.get('pass'):    return '⚠️'
    return '✅'

def scenario_status(steps):
    if all(s.get('pass') for s in steps): return '✅ PASS'
    if any(s.get('pass') for s in steps): return '⚠️ PARTIAL'
    return '❌ FAIL'
```

## SLA 초과 감지

```python
SLA_MS = {
    'dashboard': 2000, 'quota': 2000, 'service': 2000,
    'default': 3000
}

def check_sla(step):
    limit = SLA_MS.get(next(
        (k for k in SLA_MS if k in step.get('path','').lower()), 'default'
    ))
    if step.get('time', 0) > limit:
        return f"⚠️ SLA 초과: {step['time']}ms (기준: {limit}ms)"
    return None
```

## 에러 원인 분류

```python
def classify_error(step):
    status = step.get('status')
    if status == 401: return "X-Auth-Token 만료 가능성 — 재발급 필요"
    if status == 403: return "권한 없음 — Domain-ID/Project-ID 확인"
    if status == 404: return "리소스 없음 — path 파라미터 값 확인"
    if status == 400: return "잘못된 요청 — Request Body 스키마 확인"
    if status and status >= 500: return "서버 오류 — 백엔드 로그 확인"
    if step.get('error'): return f"네트워크 오류: {step['error']}"
    return "원인 불명"
```
