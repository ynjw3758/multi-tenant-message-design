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

📌 메시지 전송 업체 멀티테넌트 확장 설계서 (보강본)
1. 개요

본 문서는 기존 단일 메시지 전송 업체 구조를
조직(테넌트)별로 서로 다른 메시지 전송 업체를 사용할 수 있도록 확장하기 위한 설계 문서이다.

설계 목표

조직별 메시지 업체 설정 지원

외부 업체 장애 격리

보안 강화 (API Key/Secret 보호)

운영자 실수 방지

확장 가능한 구조 확보

본 설계는 단순 기능 추가가 아닌,
멀티테넌트 환경에서 외부 의존성을 안전하게 분리하는 것을 핵심 목표로 한다.

Part A. 서버 설계
A-1. 작업 단계
1단계. 요구사항 분석 및 가정 정의
🎯 목표

멀티테넌트 환경에서 조직별 메시지 업체 분리 구조 정의

조직 단위 API Key/Secret 관리 방식 확정

📌 가정

한 조직은 하나의 ACTIVE 업체 사용

각 조직은 외부 업체 계정을 별도로 보유

메시지 발송은 외부 HTTP API 호출 기반

메시지 발송 내역은 DB 저장

메시지 발송은 대량 트래픽 가능성 존재

💡 설계 의도

구현에 앞서 조직 단위 계약/과금 모델을 명확히 정의해야
데이터 모델과 발송 구조를 일관되게 설계할 수 있기 때문에
요구사항 분석을 최우선 단계로 배치하였다.

2단계. 데이터 모델 확장

데이터 모델을 먼저 확정한 이유는
Provider 추상화 및 발송 파이프라인 변경이
조직별 설정 저장 구조를 전제로 하기 때문이다.

📌 OrganizationMessageConfig
컬럼명	타입	설명
id	PK	설정 ID
org_id	FK	조직 ID
provider_type	ENUM	CLOUD_MESSAGE / SEND_TALK
status	ENUM	DRAFT / ACTIVE
credentials_enc	TEXT	암호화된 API Key/Secret
version	INT	낙관적 락
updated_at	DATETIME	수정 시각
updated_by	USER_ID	수정자
🔎 설계 의도

status 분리는 운영 실수 방지를 위한 안전 장치

credentials_enc는 보안 사고 예방을 위한 암호화 저장

version은 동시 수정 충돌 방지

📌 Message 테이블 변경
컬럼명	타입	설명
id	PK	메시지 ID
org_id	FK	조직 ID
provider_snapshot	ENUM	생성 시점 업체
status	ENUM	PENDING / SENT / FAILED
fail_reason	TEXT	실패 사유
🔎 provider_snapshot 설계 이유

업체 변경 시에도
이미 생성된 메시지의 발송 경로가 바뀌지 않도록
일관성을 보장하기 위함이다.

이는 재시도/장애 복구 시에도 동일 provider를 사용하도록 보장한다.

3단계. Provider 추상화 구조 도입
❌ 기존 방식: if-else 분기

문제점:

업체 추가 시 기존 코드 수정 필요 (OCP 위반)

서비스 레이어 책임 과다

테스트/장애 처리 분리 어려움

✅ 선택: 전략 패턴 기반 Provider 추상화
interface MessageProvider {
    fun send(request: MessageRequest): SendResult
}
class CloudMessageProvider : MessageProvider
class SendTalkProvider : MessageProvider
val provider = providerResolver.resolve(org.providerType)
provider.send(request)
🎯 선택 근거

OCP 준수 (확장 시 수정 최소화)

SRP 준수 (서비스 레이어는 발송 흐름만 책임)

업체별 예외 처리 독립 가능

단위 테스트/Mocking 용이

4단계. 발송 파이프라인 비동기화
📌 메시지 발송 흐름
🎯 선택 이유 (심화)

동기 호출을 사용할 경우:

외부 API 평균 응답시간 변동이 사용자 응답 지연으로 직결

대량 발송 시 스파이크 트래픽 발생

외부 장애가 내부 서비스 장애처럼 보일 위험

따라서 비동기 큐 기반으로 분리하여:

장애 격리

재시도 전략 적용

DLQ 처리 가능

Worker 수평 확장 가능

5단계. 설정 API 설계
API	설명
GET /org/provider	현재 ACTIVE 설정 조회
PUT /org/provider	DRAFT 저장
POST /org/provider/test	연결 테스트
POST /org/provider/activate	ACTIVE 전환
A-2. 기술적 결정 사항 (요약)
항목	선택	근거
업체 분기	Provider 추상화	확장성, 유지보수성
발송 처리	비동기 큐	장애 격리
설정 반영	DRAFT→TEST→ACTIVE	운영 실수 방지
변경 일관성	provider_snapshot	안정성 확보
A-3. 리스크 및 대응
위험 요소	영향 범위	완화 전략
외부 API 장애	SLA 위반, 고객 불만	타임아웃 + 재시도 + DLQ
관리자 오입력	조직 단위 발송 중단	TEST 필수화
테넌트 침해	타 조직 과금/법적 문제	JWT 기반 org 검증
Secret 유출	비용 손실/신뢰 하락	암호화 + 마스킹
동시 수정	설정 꼬임	optimistic lock
Part B. 화면 설계
B-1. 화면 흐름

설정 화면 진입 (조직명/현재 ACTIVE 표시)

업체 선택

Key/Secret 입력

저장(DRAFT)

연결 테스트

활성화(ACTIVE)

B-2. UI 결정 사항
항목	선택	이유
저장/활성화 분리	분리	실수 방지
테스트 필수화	성공 후 활성화	운영 안전성
Secret 표시	마스킹 + 재조회 불가	보안 강화
조직 정보 강조	상단 고정	테넌트 실수 방지
B-3. 예외/엣지 케이스
상황	UI 대응
필수값 누락	인라인 에러
테스트 실패	실패 사유 명확 표시
외부 장애	재시도 안내
동시 수정	충돌 메시지
업체 변경	“신규 메시지부터 적용” 안내
AI 활용 내역
사용 목적

멀티테넌트 메시지 설계 구조 검토

전략 패턴 기반 구조 설계 정제

외부 API 장애 대응 전략 정리

UI 실수 방지 패턴 설계

Mermaid 다이어그램 생성

활용 의도

정답이 정해진 문제가 아니므로
설계 대안 비교와 리스크 도출 과정에서 AI를 활용하였다.
최종 설계 선택은 요구사항과 운영 안정성 기준으로 판단하였다.
