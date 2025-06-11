# Google A2A (Agent to Agent) Protocol 전체 스펙 조사 보고서

## 1. Google A2A Protocol 개요와 목적

Google Agent-to-Agent (A2A) Protocol은 2025년 4월 9일 Google Cloud Next 2025에서 발표된 개방형 AI 에이전트 간 통신 표준입니다. 다양한 프레임워크와 벤더에서 개발된 독립적인 AI 에이전트들이 효과적으로 통신하고 협업할 수 있도록 설계되었습니다.

**핵심 목적:**
- 서로 다른 프레임워크로 구축된 에이전트 간의 상호 운용성 제공
- 에이전트가 단순한 도구가 아닌 독립적인 개체로서 협업 가능
- 내부 상태나 메모리를 노출하지 않고도 안전한 협업 지원
- 50개 이상의 기술 파트너(Atlassian, Box, Salesforce, SAP 등)가 참여하는 표준화된 에코시스템 구축

**현재 버전:** v0.2.1 (Apache License 2.0 오픈소스)

## 2. A2A 프로토콜의 핵심 구성 요소와 아키텍처

### 핵심 구성 요소:

**1) Agent Card**
- 에이전트의 메타데이터를 담은 JSON 문서
- 일반적으로 `/.well-known/agent.json` 경로에 호스팅
- 에이전트의 ID, 기능, 서비스 엔드포인트, 인증 요구사항 포함

**2) Task Management System**
- Task 객체: 고유 ID로 식별되는 작업 단위
- 생명주기 상태: submitted, working, input-required, completed, canceled, failed 등
- 컨텍스트 보존과 히스토리 관리

**3) Message System**
- 역할 기반 통신 (user/agent)
- 다중 파트 콘텐츠 지원 (TextPart, FilePart, DataPart)
- 태스크 ID와 컨텍스트 ID 참조를 통한 연속성 보장

**4) Artifact System**
- 에이전트가 생성한 불변의 최종 결과물
- 텍스트, 구조화된 데이터, 파일, 미디어 포함 가능

**5) Communication Transport**
- JSON-RPC 2.0 over HTTP(S) 기반
- 동기식 요청/응답, Server-Sent Events(SSE)를 통한 스트리밍, 비동기 푸시 알림 지원

### 설계 원칙:
- **간단하고 표준 기반**: HTTP, JSON-RPC 2.0, SSE 등 기존 표준 재사용
- **엔터프라이즈 준비**: 인증, 권한, 보안, 프라이버시, 추적, 모니터링 지원
- **비동기 우선 설계**: 장시간 실행되는 작업과 human-in-the-loop 상호작용 지원
- **모달리티 중립적**: 텍스트, 오디오/비디오, 구조화된 데이터, UI 컴포넌트 교환 지원
- **불투명한 실행**: 내부 구현을 노출하지 않고 협업

## 3. 메시지 포맷과 구조

### JSON-RPC 2.0 기본 구조:
```json
{
  "jsonrpc": "2.0",
  "id": "unique-identifier",
  "method": "method-name",
  "params": {
    // 메서드별 파라미터
  }
}
```

### Task Send 요청 예시:
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tasks/send",
  "params": {
    "id": "de38c76d-d54c-436c-8b9f-4c2703648d64",
    "sessionId": "c295ea44-7543-4f78-b524-7a38915ad6e4",
    "message": {
      "role": "user",
      "parts": [
        {
          "type": "text",
          "text": "두 자동차를 비교해주세요",
          "metadata": {}
        }
      ]
    },
    "metadata": {}
  }
}
```

### 메시지 파트 유형:

**텍스트 파트:**
```json
{
  "type": "text",
  "text": "안녕하세요!",
  "metadata": {
    "mimeType": "text/plain"
  }
}
```

**파일 파트 (인라인):**
```json
{
  "type": "file",
  "file": {
    "mimeType": "application/pdf",
    "data": "<base64-encoded-content>",
    "name": "document.pdf"
  }
}
```

**구조화된 데이터 파트:**
```json
{
  "type": "data",
  "data": {
    "user_id": 123,
    "preferences": {
      "theme": "dark",
      "notifications": true
    }
  },
  "metadata": {
    "schema": {
      "type": "object",
      "properties": {
        "user_id": {"type": "number"},
        "preferences": {"type": "object"}
      }
    }
  }
}
```

## 4. 인증 및 보안 메커니즘

### 지원되는 인증 방식:
- **Bearer Token**: OAuth 2.0 Bearer 토큰
- **API Key**: 헤더, 쿼리 파라미터, 쿠키 기반 API 키
- **Basic Auth**: HTTP Basic 인증
- **Service Accounts**: 클라우드 서비스 계정 통합
- **mTLS (상호 TLS)**: 양방향 인증서 기반 인증

### 보안 요구사항:
- **필수 HTTPS**: 프로덕션 환경에서는 반드시 HTTPS 사용
- **TLS 1.2+**: 강력한 암호화 스위트와 함께 권장
- **인증서 검증**: 신뢰할 수 있는 CA에 대한 인증서 유효성 검사
- **토큰 관리**: 자동 토큰 갱신, 취소, 무효화 지원

### Agent Card 인증 섹션 예시:
```json
{
  "authentication": {
    "schemes": ["bearer", "oauth2"],
    "credentials": "https://example.com/auth"
  }
}
```

## 5. API 엔드포인트와 메서드들

### 표준 A2A 엔드포인트:

**에이전트 발견:**
- 엔드포인트: `/.well-known/agent.json`
- HTTP 메서드: `GET`
- 목적: 에이전트 기능 및 메타데이터 조회

### JSON-RPC 메서드:

| 메서드 | 목적 | 파라미터 |
|--------|------|----------|
| `tasks/send` | 에이전트에 작업 전송 | `id`, `message`, `sessionId`, `metadata` |
| `tasks/get` | 작업 상태 및 결과 조회 | `id`, `historyLength`, `metadata` |
| `tasks/cancel` | 실행 중인 작업 취소 | `id`, `metadata` |
| `tasks/sendSubscribe` | 스트리밍 응답과 함께 작업 전송 | `tasks/send`와 동일 |
| `tasks/pushNotification/set` | 푸시 알림 구성 | `id`, `url`, `authentication`, `metadata` |

### 표준 에러 코드:

| 코드 | 설명 |
|------|------|
| -32700 | 파싱 에러 |
| -32600 | 잘못된 요청 |
| -32601 | 메서드를 찾을 수 없음 |
| -32602 | 잘못된 파라미터 |
| -32603 | 내부 에러 |
| -32000 | 작업을 찾을 수 없음 |
| -32001 | 작업이 이미 존재함 |

## 6. 에이전트 간 통신 플로우와 시퀀스

### 전체 통신 플로우:

**1. 발견 단계:**
```
- /.well-known/agent.json에서 공개 Agent Card 조회
- 인증 요구사항 및 기능 파싱
- 선택적으로 인증된 확장 카드 조회
```

**2. 인증 단계:**
```
- 필요한 자격 증명 획득 (OAuth 토큰, API 키, 인증서)
- 인증 스키마 호환성 검증
- 대상 에이전트와 보안 채널 수립
```

**3. 작업 실행 단계:**
```
- 고유 식별자로 Task 객체 생성
- 사용자 요구사항이 담긴 초기 메시지 전송
- 상호작용 모달리티 협상
- 다중 턴 메시지 교환
- SSE를 통한 실시간 상태 업데이트
- 최종 아티팩트 전달 및 세션 정리
```

### 스트리밍 통신 (SSE):
```
Content-Type: text/event-stream

data: {"jsonrpc": "2.0", "id": 1, "result": {"status": {"state": "working"}}}

data: {"jsonrpc": "2.0", "id": 1, "result": {"artifacts": [{"name": "result", "parts": [{"type": "text", "text": "부분 결과..."}]}]}}

data: {"jsonrpc": "2.0", "id": 1, "result": {"status": {"state": "completed"}}}
```

## 7. 에러 처리와 재시도 메커니즘

### 재시도 전략:
- **지수 백오프**: 기본 지연 1-2초, 지수 배수 2.0, 최대 지연 300초
- **지터**: ±25% 랜덤화로 동시 재시도 방지
- **재시도 제한**: 빠른 작업 3-5회, 장시간 작업 7-10회

### 서킷 브레이커 패턴:
- **실패 임계값**: 60초 동안 10개 요청 중 50% 에러율
- **복구 타임아웃**: 30-60초 후 half-open 상태로 전환
- **성공 임계값**: 3회 연속 성공 시 서킷 닫기

### 타임아웃 구성:
- **연결 타임아웃**: 5-15초
- **읽기 타임아웃**: 30-120초
- **Agent Card 조회**: 최대 10초
- **작업 생명주기**: 빠른 작업 30-60초, 장시간 작업 1-24시간

## 8. 성능 최적화 가이드라인

### A2A 특화 최적화:
- **Agent Card 캐싱**: 5-15분 동안 에이전트 기능 캐시
- **연결 재사용**: 자주 통신하는 에이전트 간 지속적 HTTP 연결 유지
- **페이로드 최적화**: 대용량 아티팩트와 데이터 전송을 위한 스트리밍 사용
- **HTTP/2 활용**: 동시 A2A 통신을 위한 멀티플렉싱 활용

### 성능 목표:
- **에이전트 발견**: <500ms
- **작업 시작**: <200ms
- **스트리밍 지연시간**: <100ms
- **처리량**: 서버당 1000-10000 요청/초

### 모니터링 핵심 지표:
- **골든 시그널**: 지연시간(P50, P95, P99), 에러율, 처리량, 포화도
- **A2A 특화 지표**: 에이전트 발견 성공률, 작업 완료율, 에이전트 가용성

## 8. 베스트 프랙티스와 주의사항

### 구현 베스트 프랙티스:

**1. 에이전트 설계:**
- 단일 목적 에이전트 원칙 준수
- 명확한 기능 경계 정의
- 상태 비저장 작업 우선 고려

**2. 보안:**
- 처음부터 인증/권한 구현
- mTLS를 고보안 환경에서 사용
- 단기 수명 토큰과 동적 프로비저닝 활용

**3. 성능:**
- 연결 풀링과 재사용 구현
- Agent Card 캐싱 전략 수립
- 수평적 확장 설계

**4. 에러 처리:**
- 포괄적 에러 관리 구현
- 서킷 브레이커 패턴 적용
- 멱등성 보장으로 안전한 재시도

### 일반적인 함정과 해결 방법:

**1. 확장성 문제:**
- 문제: O(N²) 통합 복잡도
- 해결: 조기에 A2A 표준화, 중앙 집중식 에이전트 발견 구현

**2. 벤더 종속성:**
- 문제: 독점 프레임워크 종속
- 해결: 개방형 표준 우선, 프로토콜 중립적 구현 유지

**3. 네트워크 오버헤드:**
- 문제: JSON-RPC over HTTP 오버헤드
- 해결: HTTP/2 사용, 효율적인 페이로드 설계

## 9. 최신 업데이트 사항 (2024-2025년)

### A2A Protocol v0.2 주요 변경사항:
- **무상태 상호작용**: 세션 관리가 필요 없는 시나리오를 위한 간소화된 개발
- **표준화된 인증**: OpenAPI 스키마 기반의 공식화된 인증 체계
- **향상된 보안**: 에이전트 간 인증 요구사항의 명확한 통신

### 2025년 로드맵:
- **v1.0 안정화 릴리스**: 2025년 하반기 예정
- **향상된 권한 부여**: Agent Card에 자격 증명 포함 공식화
- **동적 스킬 쿼리**: 런타임 기능 확인을 위한 QuerySkill() 메서드
- **UX 협상**: 대화 중간 모달리티 변경 지원

### 파트너 생태계:
- **Microsoft**: Azure AI Foundry 지원, Copilot Studio 통합
- **SAP**: Joule AI 어시스턴트 A2A 통합
- **50개 이상 파트너**: Atlassian, Salesforce, ServiceNow, Deloitte, Accenture 등

### 주요 GitHub 리포지토리:
- **A2A 프로토콜 사양**: https://github.com/google-a2a/A2A
- **a2ajava**: https://github.com/vishalmysore/a2ajava
- **SpringActions**: https://github.com/vishalmysore/SpringActions
- **Spring AI 공식**: https://github.com/spring-projects/spring-ai

## 결론

Google A2A Protocol은 AI 에이전트 간의 표준화된 통신을 위한 포괄적인 프레임워크로, 다음과 같은 핵심 강점을 제공합니다:

1. **표준화**: HTTP, JSON-RPC 2.0 등 검증된 웹 표준 기반
2. **유연성**: 다양한 콘텐츠 유형과 상호작용 패턴 지원
3. **보안**: 엔터프라이즈급 인증 및 권한 부여
4. **확장성**: 장시간 실행 작업과 스트리밍 응답을 위한 설계
5. **상호 운용성**: 벤더 중립적 접근으로 크로스 플랫폼 협업 가능

Spring AI와의 통합은 커뮤니티 개발 a2ajava 라이브러리를 통해 실현되며, 애노테이션 기반의 간단한 접근 방식으로 Spring Boot 애플리케이션을 AI 액션 가능한 서비스로 변환할 수 있습니다. 

A2A 프로토콜은 아직 초기 단계(v0.2)이지만, 50개 이상의 주요 기술 파트너의 지원과 활발한 오픈소스 커뮤니티를 통해 엔터프라이즈 환경에서 멀티 에이전트 AI 시스템의 표준이 될 가능성이 높습니다.
