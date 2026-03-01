# 📌 Multi-Tenant Message Provider Architecture Design

> 사전 설계 과제 제출 문서  
> 작성자: 윤지완  
> 작성일: 2026-03-01  

본 문서는 기존 단일 메시지 전송 업체 구조를  
조직(테넌트) 단위로 서로 다른 메시지 전송 업체를 사용할 수 있도록 확장하기 위한 설계 문서이다.

본 설계는 단순 기능 추가가 아닌,  
멀티테넌트 환경에서 외부 의존성을 안전하게 분리하는 것을 핵심 목표로 한다.

---

## 📑 Table of Contents

- [Part A. 서버 설계](#part-a-서버-설계)
  - [A-1. 작업 단계](#a-1-작업-단계)
  - [A-2. 기술적 결정 사항](#a-2-기술적-결정-사항)
  - [A-3. 리스크 및 대응](#a-3-리스크-및-대응)
- [Part B. 화면 설계](#part-b-화면-설계)
- [AI 활용 내역](#ai-활용-내역)
- [최종 요약](#최종-요약)

---

# Part A. 서버 설계

## A-1. 작업 단계

---

### 1단계. 요구사항 분석 및 가정 정의

#### 🎯 목표
- 멀티테넌트 환경에서 조직별 메시지 업체 분리 구조 정의
- 조직 단위 API Key/Secret 관리 방식 확정
- 업체 변경 시 발송 일관성 정책 정의

#### 📌 가정
- 한 조직은 동시에 1개의 ACTIVE 업체만 사용
- 각 조직은 외부 업체 계정을 별도로 보유
- 메시지 발송은 외부 HTTP API 호출 기반
- 메시지 발송 내역은 DB 저장
- 메시지 발송은 대량 트래픽 가능성 존재
- org_id는 JWT 기반으로 서버에서 강제 추출

#### 💡 설계 의도
구현 이전에 조직 단위 계약/과금 모델을 명확히 정의해야  
데이터 모델과 발송 구조를 일관되게 설계할 수 있기 때문에  
요구사항 분석을 최우선 단계로 배치하였다.

---

### 2단계. 데이터 모델 확장

Provider 추상화 및 발송 파이프라인 변경은  
조직별 설정 저장 구조를 전제로 하므로 데이터 모델을 먼저 확정하였다.

---

#### 📌 OrganizationMessageConfig

| 컬럼명 | 타입 | 설명 |
|--------|------|------|
| id | PK | 설정 ID |
| org_id | FK | 조직 ID |
| provider_type | ENUM | CLOUD_MESSAGE / SEND_TALK |
| status | ENUM | DRAFT / ACTIVE |
| credentials_enc | TEXT | 암호화된 API Key/Secret |
| version | INT | 낙관적 락 |
| updated_at | DATETIME | 수정 시각 |
| updated_by | USER_ID | 수정자 |

#### 🔎 설계 의도
- status 분리는 운영 실수 방지를 위한 안전 장치
- credentials_enc는 보안 사고 예방을 위한 암호화 저장
- version은 동시 수정 충돌 방지

---

#### 📌 Message 테이블 변경

| 컬럼명 | 타입 | 설명 |
|--------|------|------|
| id | PK | 메시지 ID |
| org_id | FK | 조직 ID |
| provider_snapshot | ENUM | 생성 시점 업체 |
| status | ENUM | PENDING / SENT / FAILED |
| fail_reason | TEXT | 실패 사유 |

#### 🔎 provider_snapshot 설계 이유

업체 변경 이후에도  
이미 생성된 메시지의 발송 경로가 변경되지 않도록  
일관성을 보장하기 위함이다.

이는 재시도 및 장애 복구 시 동일 Provider를 사용하도록 보장한다.

---

### 3단계. Provider 추상화 구조 도입

#### ❌ 기존 방식: if-else 분기

- 업체 추가 시 기존 코드 수정 필요 (OCP 위반)
- 서비스 레이어 책임 과다
- 테스트 및 장애 처리 분리 어려움

#### ✅ 선택: 전략 패턴 기반 Provider 추상화

```kotlin
interface MessageProvider {
    fun send(request: MessageRequest): SendResult
}

class CloudMessageProvider : MessageProvider
class SendTalkProvider : MessageProvider

val provider = providerResolver.resolve(org.providerType)
provider.send(request)
```
🔎 ProviderResolver의 역할

조직의 ACTIVE 설정을 조회하여 provider_type을 결정한다.

provider_type에 따라 적절한 MessageProvider 구현체를 반환한다.

서비스 레이어는 Provider 구현체에 대한 구체적인 의존 없이 send()만 호출한다.

이를 통해 서비스 레이어는 발송 흐름만 책임지고,
업체별 구현 세부사항은 Provider 구현체로 격리된다.

```mermaid
flowchart LR
    Org[Organization]
    Config[OrganizationMessageConfig]
    Resolver[ProviderResolver]
    Cloud[CloudMessageProvider]
    Send[SendTalkProvider]

    Org --> Config
    Config --> Resolver
    Resolver --> Cloud
    Resolver --> Send
