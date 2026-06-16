# CLAUDE.md — Pub/Sub Swagger API 자동화 테스트 하네스

## 하네스: Pub/Sub Swagger API 자동화 테스트

**목표:** index.html의 시나리오·Swagger 스펙을 기반으로 Kakao Cloud Pub/Sub API를 CLI에서 자동 실행·검증·리포트

**트리거:** "테스트 실행", "시나리오 실행", "API 검증", "전체 실행", "s1/s2/s3 실행", "결과 리포트", "API 테스트해줘", "시나리오 돌려줘" 등 테스트 관련 요청 시 `pubsub-test-orchestrator` 스킬을 사용하라. 단순 질문은 직접 응답 가능.

**변경 이력:**
| 날짜 | 변경 내용 | 대상 | 사유 |
|------|----------|------|------|
| 2026-06-15 | 초기 구성 | 전체 | index.html 분석 기반 신규 구축 |

---

## 프로젝트 개요

Kakao Cloud Pub/Sub API Swagger 자동화 테스트 도구.

- `index.html`: 브라우저 기반 UI 테스트 도구 (수정 금지)
- `swagger/`: API 스펙 파일 4종 (console_v1/v2, pubsub_v1/v2)
- `.claude/`: 하네스 에이전트·스킬 (CLI 기반 자동화)
- `_workspace/`: 테스트 실행 시 생성되는 중간/결과 파일 (런타임 생성)

## 에이전트 구성

| 에이전트 | 역할 |
|---------|------|
| `spec-analyst` | Swagger JSON + index.html 시나리오 파싱 |
| `api-test-runner` | HTTP API 실행, extract/inject/assert 처리 |
| `result-reporter` | 결과 분석 및 Markdown 리포트 생성 |

## 스킬 구성

| 스킬 | 역할 |
|------|------|
| `pubsub-test-orchestrator` | 전체 테스트 워크플로우 조율 (오케스트레이터) |
| `swagger-spec-reader` | Swagger JSON 파싱 패턴 |
| `scenario-executor` | API 시나리오 실행 패턴 (extract/inject/assert) |
| `test-report-generator` | 결과 리포트 생성 패턴 |
