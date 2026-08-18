# org-structure.md — 엔터프라이즈 AWS Organizations 설계

## 1. Business Context

본 설계에서는 **개발·IT 인력 약 60명 규모의 B2C SaaS 기업**을 가정한다. 웹과 모바일을 통해 고객에게 SaaS 서비스를 제공하며, 서비스 규모가 성장하면서 애플리케이션 개발뿐 아니라 Kubernetes 플랫폼, 네트워크, 보안 영역을 각각 전문적으로 운영할 필요가 생긴 상황을 전제로 한다.

조직은 다음 네 개의 책임 영역으로 구분한다.

* **Application Team**

  * 웹·모바일 애플리케이션 개발
  * 애플리케이션 배포 및 서비스 기능 운영
* **DevOps / Platform Team**

  * EKS 플랫폼 운영
  * Terraform 기반 인프라 관리
  * CI/CD 플랫폼 및 배포 환경 운영
  * Observability 등 공통 플랫폼 운영
* **Network Team**

  * AWS 네트워크 구조 및 연결 관리
  * VPC, Routing 등 네트워크 영역 운영
* **Security Team**

  * 보안 정책 및 탐지 체계 운영
  * 감사 로그 관리
  * 보안 이벤트 조사 및 대응

이와 같이 조직을 구분한 이유는 단순히 인원이 많기 때문이 아니다. 서비스 규모가 증가하면서 각 영역에서 요구되는 **권한, 변경 주기, 운영 책임 및 사고 발생 시 영향 범위가 달라졌기 때문**이다.

AWS 계정 역시 이러한 책임과 격리 요구사항을 반영하여 설계한다. AWS는 계정을 permission, security, cost 및 workload의 자연스러운 경계로 설명하며 multi-account 환경을 권장하고 있다.

---

## 2. 설계 사고 순서

AWS Organizations의 구조를 회사 조직도에서 바로 만들지 않는다.

본 설계에서는 다음 순서로 구조를 결정한다.

```text
Business Context
       ↓
조직별 책임과 필요한 권한
       ↓
격리가 필요한 Resource / Workload 식별
       ↓
AWS Account Boundary 결정
       ↓
동일한 Governance가 필요한 Account를 OU로 Grouping
       ↓
SCP를 통한 Organization-level Guardrail 적용
       ↓
IAM Identity Center 기반 Human Access 설계
```

즉,

> **조직이 네 개이므로 AWS 계정도 네 개를 만든다**

는 방식으로 접근하지 않는다.

먼저 한 영역에서 발생한 장애, 권한 오용 또는 계정 침해가 다른 영역까지 영향을 미쳐도 되는지를 판단하고, 필요한 경우 AWS Account를 별도의 isolation boundary로 사용한다.

그 후 동일한 통제 정책을 적용해야 하는 Account를 OU로 묶는다.

---

## 3. Account 구성

최종적으로 다음 **6개의 AWS Account**를 사용한다.

```text
AWS Organization
│
├── Management Account
│
├── Security OU
│   ├── Log Archive Account
│   └── Security Tooling Account
│
├── Infrastructure OU
│   └── Network Account
│
└── Workloads OU
    ├── EKS NonProd Account
    └── EKS Prod Account
```

---

### 3.1 Management Account

#### 담당 조직

**DevOps / Platform Team**

단, Platform Team 전체에 Management Account 관리자 권한을 제공하지 않고, 조직 관리 역할을 맡은 **제한된 Organization Administrator만 접근**하도록 한다.

#### 존재 이유

Management Account는 다음과 같은 AWS Organization 전체 관리 목적으로 사용한다.

* AWS Organizations 관리
* Account 및 OU 관리
* Organization-level 정책 관리
* Billing 및 Organization-level governance

일반적인 EKS, EC2, 애플리케이션 및 CI/CD workload는 Management Account에 배치하지 않는다.

AWS 역시 Management Account 접근자를 최소화하고, Management Account에서만 수행할 수 있는 작업에 한해 사용하며 일반 workload 배포를 피하도록 권고하고 있다.

---

### 3.2 Log Archive Account

#### 담당 조직

**Security Team**

#### 존재 이유

각 AWS Account에서 생성되는 감사·보안 로그의 원본을 중앙에서 보관한다.

대표적인 대상은 다음과 같다.

* AWS CloudTrail
* AWS Config
* 기타 중앙 보관이 필요한 감사 로그

Security Tooling과 Log Archive를 하나의 계정으로 통합하지 않고 **별도 Account로 분리한다.**

Security Tooling은 보안 탐지와 분석을 수행하는 영역이고, Log Archive는 사고 발생 이후에도 조사에 사용할 수 있는 **감사 증거를 보존하는 영역**이기 때문이다.

예를 들어 Security Tooling Account의 관리자 권한이 탈취되거나 보안 담당자의 운영 실수가 발생하더라도, 동일한 권한으로 감사 로그 원본까지 삭제하거나 변조할 수 있는 구조는 피해야 한다.

AWS Control Tower 역시 landing zone 구성 시 Security OU에 **Log Archive Account와 Audit Account를 각각 별도로 생성**하는 구조를 사용한다.

#### 계정 분리만으로 충분한가?

Account를 분리했다고 해서 로그가 자동으로 보호되는 것은 아니다.

Security Team 구성원이 두 Account 모두에 강한 권한을 가지고 있다면 계정 경계만으로는 충분한 보호가 되지 않는다.

따라서 다음 보호 장치를 함께 적용한다.

#### S3 Object Lock

로그 객체에는 **S3 Object Lock Compliance mode를 적용한다.**

Object Lock은 WORM(Write Once Read Many) 모델을 사용하여 지정된 기간 동안 객체가 삭제되거나 덮어쓰이는 것을 방지할 수 있다.

Governance mode는 `s3:BypassGovernanceRetention` 권한을 가진 주체가 retention을 우회할 수 있기 때문에, 본 설계가 가정한 관리자 권한 탈취 시나리오에서는 공격자가 해당 권한까지 획득하면 로그 보호를 우회할 수 있다. 반면 Compliance mode는 retention 기간 동안 보호된 object version을 root user를 포함한 누구도 삭제하거나 retention을 단축할 수 없으므로 감사 증거 보존이라는 목적에 더 부합한다.

다만 Compliance mode는 잘못 설정한 retention도 만료 전까지 되돌리거나 단축할 수 없기 때문에 운영 실수의 복구 가능성을 희생한다. 따라서 retention 기간 자체는 보존 규정과 운영 요구사항을 별도로 검토해 결정하고, Object Lock 정책 변경 권한도 제한한다.

#### S3 Versioning + MFA Delete

S3 Versioning을 사용하여 실수로 객체가 삭제되거나 덮어써진 경우 이전 버전을 복구할 수 있도록 한다.

추가로 MFA Delete를 사용하면 object version을 영구 삭제하거나 Versioning 상태를 변경할 때 **버킷 소유 Account의 root user와 MFA 인증**이 필요하다.

따라서 Log Archive 보호는 다음과 같이 여러 계층을 조합한다.

```text
Account Isolation
      +
Restricted Permission Set
      +
S3 Object Lock
      +
S3 Versioning / MFA Delete
```

MFA Delete가 root user를 필요로 하기 때문에 **Log Archive Account의 root credential과 MFA에 대한 별도의 강한 접근 통제도 필요하다.**

---

### 3.3 Security Tooling Account

#### 담당 조직

**Security Team**

#### 존재 이유

Organization 전체의 보안 탐지와 분석을 중앙에서 수행하기 위한 계정이다.

예를 들어 다음과 같은 보안 서비스와 운영 기능을 이 영역에서 관리할 수 있다.

* Amazon GuardDuty
* AWS Security Hub
* 보안 이벤트 분석
* 보안 운영 자동화

보안 탐지 기능과 감사 로그 원본을 분리함으로써 Security Tooling 영역이 침해되더라도 Log Archive의 감사 증거까지 동일한 blast radius에 포함시키지 않는 것을 목표로 한다.

---

### 3.4 Network Account

#### 담당 조직

**Network Team**

#### 존재 이유

EKS 및 애플리케이션 workload와 공통 네트워크 영역의 lifecycle과 관리 책임이 다르기 때문에 Network Account를 별도로 구성한다.

Network Team은 네트워크 영역을 담당하고, Platform Team은 EKS와 workload 플랫폼을 담당한다.

따라서 Platform Engineer가 EKS를 관리하기 위해 높은 권한을 가지고 있다고 해서 공통 네트워크까지 변경할 수 있어야 할 이유는 없다.

예를 들어 Platform 영역에서 다음 문제가 발생할 수 있다.

```text
잘못된 Terraform 적용
또는
Platform Credential 탈취
        ↓
EKS / Workload Resource 변경
```

Network와 EKS가 동일한 Account와 광범위한 권한 안에 있다면 장애가 네트워크 영역까지 확대될 수 있다.

따라서 Network Account를 분리해 다음 원칙을 적용한다.

> **EKS 영역에서 발생한 운영 실수 또는 권한 탈취가 공통 네트워크 영역까지 확산되지 않도록 Account와 Permission Boundary를 분리한다.**

#### 분리의 비용

Account 분리는 무료로 복잡도를 제거해 주는 방법은 아니다.

Network와 EKS를 서로 다른 Account로 분리하면 다음 항목의 설계가 추가로 필요하다.

* Cross-account 접근
* Account 간 Network 연결
* IAM Role 설계
* Terraform state 및 배포 경로
* CI/CD의 Cross-account 배포 방식

그럼에도 프로덕션을 가정한 엔터프라이즈 설계에서는 책임 분리와 blast radius 감소 효과가 이 운영 복잡도보다 크다고 판단해 Account를 분리한다.

---

### 3.5 EKS NonProd Account

#### 담당 조직

**DevOps / Platform Team**

Application Team은 필요한 범위의 애플리케이션 개발·배포 권한을 부여받는다.

#### 존재 이유

개발 및 검증을 위한 EKS workload를 Production 환경에서 분리한다.

현재 설계에서는 다음 환경을 NonProd Account에서 운영하는 것으로 가정한다.

```text
DEV
QA
STG
```

DEV, QA, STG 각각을 별도의 AWS Account로 분리할 수도 있지만, 현재 가정한 약 60명 규모에서는 Account 증가에 따른 운영 복잡도 대비 추가 isolation의 이점이 크지 않다고 판단하여 하나의 NonProd Account로 묶는다.

---

### 3.6 EKS Prod Account

#### 담당 조직

**DevOps / Platform Team**

Application Team에는 Production 운영에 필요한 제한된 권한만 제공한다.

#### 존재 이유

Production workload를 NonProd workload로부터 강하게 격리한다.

NonProd에서는 개발과 검증 과정에서 다음과 같은 변경이 상대적으로 빈번하게 발생한다.

* Terraform 변경
* Kubernetes manifest 변경
* 애플리케이션 배포
* 테스트 및 실험
* 개발자의 운영 실수

따라서 NonProd에서 발생한 잘못된 변경이나 credential compromise가 Production workload에 직접 영향을 미치지 않도록 Account boundary를 둔다.

---

### 3.7 CI/CD 배치 결정

현재 단계에서는 **별도의 Deployments Account를 만들지 않는다.**

CI/CD는 Workloads 영역에 배치하되, 구체적으로

* NonProd에서 build 후 Prod로 승격할 것인지
* Account별 pipeline을 둘 것인지
* 중앙 pipeline에서 cross-account deployment를 수행할 것인지

는 이 문서에서 결정하지 않는다.

이 구조는 **Phase 3 CI/CD 설계에서 결정한다.**

별도 Deployments Account를 현재 제외하는 이유는 중앙 배포 플랫폼을 독립 Account로 운영해야 한다는 요구사항이 아직 확정되지 않았기 때문이다.

다음 조건이 발생하면 Deployments Account 분리를 재검토한다.

* 다수의 Workload Account에 공통으로 배포하는 중앙 CI/CD Platform이 필요할 때
* Pipeline credential compromise의 blast radius를 Workload Account와 분리할 필요가 커질 때
* CI/CD 운영 주체와 Workload 운영 주체의 책임이 명확하게 분리될 때

---

## 4. OU 구조

본 설계에서는 다음 세 개의 OU를 사용한다.

```text
AWS Organization Root
│
├── Management Account
│
├── Security OU
│   ├── Log Archive Account
│   │   └── Owner: Security Team
│   │
│   └── Security Tooling Account
│       └── Owner: Security Team
│
├── Infrastructure OU
│   └── Network Account
│       └── Owner: Network Team
│
└── Workloads OU
    ├── EKS NonProd Account
    │   ├── Owner: DevOps / Platform Team
    │   └── User: Application Team
    │
    └── EKS Prod Account
        ├── Owner: DevOps / Platform Team
        └── Limited User: Application Team
```

OU는 회사 조직도를 그대로 복제하기 위한 구조가 아니라 **동일한 governance와 control을 적용할 Account들을 묶기 위한 구조**로 사용한다.

---

### 4.1 Security OU

#### 포함

* Log Archive Account
* Security Tooling Account

#### 존재 이유

보안·감사 관련 Account에 일반 workload보다 강한 보호 정책을 공통 적용하기 위해 분리한다.

특히 Security logging이나 탐지 기능의 임의 비활성화 및 변경을 방지하는 Organization-level guardrail을 적용할 수 있다.

---

### 4.2 Infrastructure OU

#### 포함

* Network Account

#### 존재 이유

여러 Workload가 의존할 수 있는 공통 Infrastructure 영역을 Application workload와 분리한다.

현재 범위에서는 Network Account만 존재하지만 향후 공통 Infrastructure Account가 추가될 경우 같은 governance 기준을 적용할 수 있다.

---

### 4.3 Workloads OU

#### 포함

* EKS NonProd Account
* EKS Prod Account

#### 존재 이유

실제 B2C SaaS 서비스를 실행하는 workload Account를 관리한다.

Production과 NonProduction은 Account level에서 격리하지만, workload라는 공통 성격 때문에 Workloads OU 아래에서 관리한다.

---

### 4.4 제외한 OU

AWS의 Recommended OU를 모두 생성하지 않는다.

현재 Business Context에 필요한 OU만 선택하고, 명확한 요구사항이 없는 OU는 제외한다.

#### Sandbox OU — 제외

개발자가 자유롭게 실험하기 위한 별도 Sandbox Account 운영 요구사항을 현재 가정하지 않았다.

향후 개발자 self-service 실험 환경이 필요하면 추가를 검토한다.

#### Exceptions OU — 제외

표준 SCP 또는 보안 정책에서 예외 처리가 필요한 Account가 현재 존재하지 않는다.

정책 예외 Account가 생길 경우 추가한다.

#### Transitional OU — 제외

인수·합병, 기존 AWS Account 이전 또는 Legacy Account 정리 상황을 현재 Business Context에서 가정하지 않는다.

#### Suspended OU — 제외

폐기 또는 일시 중지된 Account를 별도로 관리해야 하는 요구가 현재 없다.

#### Policy Staging OU — 제외

현재 Account 수와 정책 규모에서는 별도 OU를 두고 SCP 변경을 검증할 필요성이 낮다고 판단한다.

향후 Account 및 SCP 수가 증가하여 policy rollout 자체가 운영 위험이 되면 추가를 검토한다.

#### Deployments OU — 제외

현재 별도 Deployments Account를 만들지 않았기 때문에 Deployments OU 역시 생성하지 않는다.

중앙 CI/CD Platform을 별도 Account로 운영해야 할 요구가 Phase 3에서 확인되면 Account와 OU 구조를 함께 재검토한다.

---

## 5. Service Control Policy

SCP는 사용자에게 권한을 부여하는 IAM Policy가 아니라 Organization 내 Account에 적용할 수 있는 **최대 permission 범위를 제어하는 guardrail**로 사용한다.

본 설계에서는 우선 다음 두 가지 SCP를 적용한다.

---

### 5.1 승인되지 않은 Region 사용 제한

서비스의 기본 Region은 **`ap-northeast-2`(서울)** 로 결정한다. Phase 2 역시 서울 Region 사용이 확정되어 있다.

따라서 특별한 요구가 없는 상태에서 다른 Region에 workload resource를 생성하는 것을 제한한다.

#### 목적

* 관리 대상 Region 축소
* 잘못된 Region의 Resource 생성 방지
* Shadow Infrastructure 방지
* 비용 관리 단순화
* 데이터 위치 관리

실제 SCP를 구현할 때 IAM, AWS Organizations 등 global service에 필요한 예외 action을 별도로 검토한다.

---

### 5.2 Security Control 훼손 방지

Workload 또는 Infrastructure 운영자가 중앙 보안·감사 기능을 임의로 비활성화하거나 훼손하지 못하도록 제한한다.

보호 대상의 예시는 다음과 같다.

* CloudTrail 비활성화 또는 삭제
* AWS Config 관련 핵심 설정 훼손
* 중앙 보안 서비스의 임의 비활성화
* 중앙 Log Archive 보호 설정 변경

이 정책의 목적은 공격자가 Workload Account의 높은 권한을 탈취했더라도 자신의 활동 흔적을 제거하기 위해 Organization의 보안·감사 기능까지 함께 무력화시키는 것을 어렵게 만드는 것이다.

단, SCP만으로 로그 원본 보호를 완성하지 않고 Log Archive Account의 Permission Set과 S3 protection을 함께 적용한다.

---

## 6. Human Access

각 AWS Account에 장기 IAM User를 생성해 사람이 로그인하는 방식을 사용하지 않는다.

사용자 접근은 **AWS IAM Identity Center**를 중심으로 구성하고 Permission Set을 통해 Account별 권한을 부여한다.

IAM Identity Center는 사용자 또는 그룹에 여러 AWS Account의 접근 권한을 중앙에서 부여할 수 있으며 Permission Set을 이용해 Account별 권한 수준을 정의할 수 있다.

전체 흐름은 다음과 같다.

```text
Corporate Identity Provider
        │
        ▼
AWS IAM Identity Center
        │
        ▼
User / Group
        │
        ▼
AWS Account Assignment
        │
        ▼
Permission Set
        │
        ▼
Temporary Role Session
        │
        ▼
AWS Resource
```

---

### 6.1 Identity Center 운영 위치 — delegated administrator

IAM Identity Center의 organization instance는 Management Account에 존재하지만, 일상적인 사용자·그룹·Permission Set 및 Account Assignment 관리를 위해 **Security Tooling Account를 delegated administrator로 지정한다.** 이를 통해 일반적인 Identity Center 운영을 위해 Management Account에 반복적으로 접근하지 않고, Management Account 접근을 Organization 수준의 제한된 작업으로 최소화한다.

```text
Management Account
│
│  IAM Identity Center organization instance
│
└── delegated administration
        │
        ▼
Security Tooling Account
        │
        └── Security Team
             ├── Permission Set 관리
             ├── Account Assignment 관리
             └── 일반 Identity Center 운영
```

여기서도 역할이 구분된다. Management Account 소유는 계속 DevOps/Platform의 제한된 Organization Administrator가 맡고, workforce access governance의 일상 운영은 Security Team이 담당한다. 따라서 3.1의 "Management 접근 최소화" 원칙과 6장의 "Identity Center 중앙 관리"가 충돌하지 않는다.

---

### 6.2 Permission Set과 Account Boundary 정렬

Account를 분리하더라도 한 사용자가 모든 Account의 Administrator 권한을 가진다면 계정 분리의 보안 효과가 크게 감소한다.

따라서 Account의 담당 조직과 Identity Center의 Permission Set을 정렬한다.

예시는 다음과 같다.

#### Application Team

```text
EKS NonProd
└── DeveloperAccess

EKS Prod
└── 제한된 Deployment 또는 ReadOnly 권한
```

#### DevOps / Platform Team

```text
EKS NonProd
└── PlatformAdmin

EKS Prod
├── PlatformOperator
└── 필요 시 제한된 Privileged Access

Management Account
└── Organization Administrator 일부 인원만
```

#### Network Team

```text
Network Account
└── NetworkAdmin
```

Platform Engineer에게 NetworkAdmin 권한을 기본 제공하지 않는다.

#### Security Team

```text
Security Tooling Account
└── SecurityAdmin

Log Archive Account
└── Audit / ReadOnly 중심
```

Security Team 소속이라는 이유만으로 Log Archive의 데이터를 삭제할 수 있는 광범위한 권한을 기본 제공하지 않는다.

AWS 또한 Administrative user에게 관리자 permission set만 부여하기보다, 평상시 사용할 수 있는 보다 제한적인 permission set을 함께 제공하여 필요한 권한만 사용하도록 권고한다.

---

### 6.3 권한 상승

평상시에는 업무 수행에 필요한 최소 권한 Permission Set을 사용한다.

Production 장애 대응이나 Organization 변경 등 강한 권한이 필요한 작업은 별도의 privileged permission set을 사용하도록 설계한다.

```text
Normal Permission
        ↓
Privileged operation 필요
        ↓
별도 Privileged Permission Set
        ↓
제한된 Session
        ↓
작업 수행
        ↓
Audit Log 기록
```

구체적인 JIT 승인 방식, MFA 조건 및 Session Duration 등은 Human Access 세부 설계 단계에서 추가 결정한다.

---

## 7. Phase 2 — 1인 축소 구현 기준

이 문서는 **정식 Production 환경을 가정한 Architecture Design**이다.

반면 Phase 2의 실제 구현은 DevOps 학습을 목적으로 한 1인 Terraform 환경이며, 월 비용 상한과 세션 종료 후 Resource destroy라는 제약이 있다.

따라서 Production 설계 전체를 실제 AWS Account로 구현하지 않는다.

판단 기준은 다음과 같다.

> **해당 구성 요소가 Phase 2에서 학습하려는 기술 원리를 검증하는 데 필요하면 구현하고, 엔터프라이즈 운영 규모 때문에 필요한 요소라면 실제 구현은 생략하되 설계 원칙은 유지한다.**

---

### 7.1 Phase 2에서 실제 구현하지 않는 영역

다음 항목은 실제 multi-account 환경으로 구현하지 않는다.

* AWS Organizations
* Security OU
* Infrastructure OU
* Workloads OU
* Management / Security / Network / Workload의 물리적 Account 분리
* Log Archive Account
* Security Tooling Account
* Prod / NonProd 별도 Account
* Control Tower Landing Zone

Phase 2의 실제 AWS 환경은 **Single Account**로 축소한다.

---

### 7.2 Phase 2에서 유지할 원칙

Account를 하나로 줄인다고 해서 Architecture Design에서 정의한 책임 경계를 모두 제거하지 않는다.

다음 원칙은 그대로 유지한다.

* Network와 EKS의 책임 분리
* 최소 권한 IAM
* IAM User 대신 Role 기반 접근 방향
* Network / EKS / IAM의 논리적 Resource Boundary
* Public / Private Subnet 분리
* Multi-AZ 설계
* EKS
* Terraform 기반 Infrastructure as Code
* Naming / Tagging 기준
* Workload와 Infrastructure의 blast radius를 고려하는 설계

Phase 2 진행 순서 역시 Organizations 설계 후 단순화된 Network Architecture를 만들고, 이후 ADR과 Terraform 구현으로 넘어가도록 정해져 있다.

따라서 Phase 2의 축소는 Production Architecture를 포기하는 것이 아니라,

> **물리적 Account Boundary를 실제 구현하지 않는 대신, 그 경계를 만들었던 책임 분리와 최소 권한 원칙을 Single Account 안의 Terraform 및 IAM 설계에 최대한 보존하는 것**

으로 정의한다.

---

## 8. 최종 구조 요약

```text
B2C SaaS / 개발·IT 약 60명
        │
        ▼
책임 영역 분리
Application / Platform / Network / Security
        │
        ▼
Account Boundary
        │
        ├── Management
        ├── Log Archive
        ├── Security Tooling
        ├── Network
        ├── EKS NonProd
        └── EKS Prod
        │
        ▼
Governance 기준 Grouping
        │
        ├── Security OU
        ├── Infrastructure OU
        └── Workloads OU
        │
        ▼
Organization Guardrail
        │
        ├── Region Restriction
        └── Security Control Protection
        │
        ▼
IAM Identity Center
        │
        ▼
Account별 Permission Set
        │
        ▼
Least Privilege / Temporary Role Session
```

이 구조에서 가장 중요한 설계 원칙은 Account와 OU의 개수 자체가 아니다.

**Business Context에서 책임을 정의하고, 보호해야 할 blast radius와 권한 경계를 Account로 분리한 뒤, 동일한 governance가 필요한 Account를 OU로 묶고, SCP와 Identity Center를 통해 그 경계를 실제 권한 체계와 정렬하는 것**이 본 설계의 핵심이다.


---

## 참고 자료

- [Organizing Your AWS Environment Using Multiple Accounts (AWS Whitepaper)](https://docs.aws.amazon.com/whitepapers/latest/organizing-your-aws-environment/organizing-your-aws-environment.html) — recommended OU 패턴, account를 격리 경계로 보는 근거
- [Best practices for the management account (AWS Organizations)](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_best-practices_mgmt-acct.html) — management account 접근 최소화·workload 배치 금지
- [How AWS Control Tower works — shared accounts](https://docs.aws.amazon.com/controltower/latest/userguide/how-control-tower-works.html) — Log Archive / Audit 계정 분리 기본 제공
- [Service control policies (SCPs)](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps.html) — SCP는 권한 부여가 아닌 guardrail
- [S3 Object Lock](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html) — WORM 보존
- [S3 MFA Delete](https://docs.aws.amazon.com/AmazonS3/latest/userguide/MultiFactorAuthenticationDelete.html) — root + MFA 요구 조건
- [IAM Identity Center — Permission sets](https://docs.aws.amazon.com/singlesignon/latest/userguide/permissionsetsconcept.html) — 계정별 권한의 중앙 관리
- [IAM Identity Center — Delegated administration](https://docs.aws.amazon.com/singlesignon/latest/userguide/delegated-admin.html) — member account로 일상 운영 위임
