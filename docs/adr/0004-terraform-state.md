# ADR-0004: Terraform state 저장·잠금 — S3 backend + native lockfile

- **상태**: Accepted
- **날짜**: 2026-08-21
- **관련 Phase**: Phase 2
- **영향받는 후속 Phase**: Phase 3 (CI가 이 버킷에 접근해 plan/apply를 수행한다)
- **관련 ADR**: ADR-0003 (IaC 도구 선정 — 이 결정의 전제) / ADR-0002 (EKS 선정)

## Context

Terraform은 관리 대상 리소스의 실제 상태를 state 파일에 기록한다. state는
**"Terraform이 아는 현실"**이며, 코드와 실제 인프라를 잇는 유일한 연결점이다.
따라서 state의 저장 위치와 동시 접근 제어는 선택이 아니라 필수 결정이다.

결정 시점의 조건:

- 1인 랩이지만 **Phase 3에서 CI가 apply를 수행할 예정이다.** 즉 동시 접근자가
  로컬·CI 2개가 되는 것이 이미 로드맵에 있다. 잠금은 가정이 아니라 예정된 요구다.
- Terraform v1.15.8 (ADR-0003에서 `~> 1.15.0`으로 고정) — `use_lockfile` 지원 버전이다.
- AWS 단일 계정, 서울 리전(`ap-northeast-2`) 기준.

로컬 state를 배제하는 근거:

1. **유실 = 리소스 고아화.** state를 잃으면 Terraform은 자신이 만든 리소스를
   인식하지 못한다. 리소스는 살아서 과금되지만 코드로는 손댈 수 없다.
2. **잠금이 없다.** 두 곳에서 동시에 apply하면 state가 손상된다.
3. **git 커밋 위험.** state에는 리소스 속성이 평문으로 들어가며, 일부 리소스는
   민감값을 포함한다. 로컬 파일은 실수로 커밋될 경로 위에 있다.

시점 사실 두 가지 — 오래된 블로그를 따라가면 구식 구성을 만드는 지점이다.

- **DynamoDB 기반 잠금은 deprecated이다.** 공식 문서 표현: *"DynamoDB-based locking
  is deprecated and will be removed in a future minor version."* Terraform은 S3의
  조건부 쓰기를 이용한 native locking(`use_lockfile`)을 제공하며, 별도 테이블이 필요 없다.
- **S3 버킷의 암호화와 public 차단은 이미 기본값이다.** 2023년 1월 이후 신규 버킷은
  SSE-S3가 기본 적용되고, 2023년 4월 이후 신규 버킷은 Block Public Access가
  기본 활성화되며 ACL이 비활성화된다. 따라서 이 두 항목은 *설정*이 아니라 **검증** 작업이다.
  명시적 활성화가 실제로 필요한 것은 versioning 하나다.

## Decision

**S3 원격 백엔드를 사용하고, 잠금은 `use_lockfile`(S3 native)로 한다.**

결정은 되돌리는 비용이 크게 다른 두 층으로 구성된다.

| 층 | 결정 | 되돌리는 비용 |
|---|---|---|
| **저장 위치의 정체성** — 원격 백엔드 채택 + 버킷 이름 + 리전 + 키 스키마 | S3 / `hangar-tfstate-{계정ID}` / `ap-northeast-2` / `dev/main/terraform.tfstate` | **높음** — S3 버킷은 이름 변경이 불가능하다. 새 버킷 생성 → 객체 복사 → CI 설정·IAM 정책·`terraform_remote_state` 참조를 모두 갱신해야 한다 |
| **잠금 구현 방식** | `use_lockfile = true` | 낮음 — 한 줄 변경 + `terraform init -reconfigure`. state 내용에 영향 없음 |

**이 ADR의 무게는 첫 번째 층에 있다.** 두 번째 층은 공식 문서가 이미 답을 정해둔
사안이라(deprecated) 실질적으로 결정이 아니다. 따라서 검토 시간은 "DynamoDB냐
lockfile이냐"가 아니라 **버킷 이름·리전·키 스키마**에 써야 한다. 이 세 값은
코드·CI 설정·IAM 정책·문서로 퍼지며, 하나를 바꾸면 전부 따라 움직인다.

**왜 원격 백엔드(S3)인가** — 위 Context의 세 가지 로컬 배제 근거를 그대로 해소한다.
버킷 versioning이 유실에 대한 안전망을 제공하고, 조건부 쓰기가 잠금을 제공하며,
state가 애초에 작업 트리 밖에 있어 커밋 사고 경로가 사라진다. Terraform Cloud도
같은 문제를 해결하지만 외부 SaaS 의존을 추가하고, 이 프로젝트의 학습 대상인
"백엔드를 직접 구성하는 경험"을 건너뛰게 만든다.

**왜 `use_lockfile`인가** — DynamoDB 방식이 공식 deprecated이고 제거가 예고되어 있다.
관리할 리소스가 하나 줄고(테이블 불필요), IAM 권한 표면도 줄어든다. 잠금 객체는
state 객체 옆에 `<key>.tflock`으로 생성되므로 별도 리소스의 수명 관리가 없다.

### 버킷 사양

| 항목 | 값 | 성격 |
|---|---|---|
| 이름 | `hangar-tfstate-{계정ID}` | 결정 — 전역 유일해야 하므로 계정 ID를 접미. 용도(`tfstate`)를 이름에 박아 6개월 뒤 "지워도 되나" 판단 비용을 없앤다 |
| 리전 | `ap-northeast-2` | 결정 — 작업 리전과 일치시켜 지연·혼동을 줄인다 |
| 키 | `dev/main/terraform.tfstate` | 결정 — `{환경}/{스택}` 축. 후술 |
| Versioning | Enabled | **명시적 활성화 필요** |
| 암호화 | SSE-S3 (기본값 유지) | 검증 |
| Block Public Access | 전체 차단 (기본값 유지) | 검증 |
| Lifecycle | noncurrent version 90일 만료 | 결정 |

**키 스키마를 `dev/main/`으로 두는 이유** — 리소스 이름이 아니라 `{환경}/{스택}` 축이다.
현재 flat 시작 구조는 지금 state가 하나임을 뜻하지만, EKS 생성에 10~15분이 걸려
"VPC는 남기고 클러스터만 내리고 싶다"는 요구가 곧 발생한다. 그때 `dev/network/`,
`dev/cluster/`로 분할하면 **모든 state가 같은 depth에 놓여** IAM 정책 패턴
(`dev/*/terraform.tfstate`)과 lifecycle 규칙을 고치지 않아도 된다. 지금 한 세그먼트를
더 두는 비용은 0이고, 나중에 추가하려면 state 이사가 필요하다.

**SSE-KMS를 기각한 이유** — 요청당 과금과 키 정책 관리 부담이 추가되는데,
1인 랩에서 얻는 것은 "키 회전 이력"뿐이다. 감사·컴플라이언스 요구가 생기면
그때 승격한다(재검토 트리거 참조).

**MFA Delete를 기각한 이유** — 설정과 해제에 root 자격증명이 필요하다.
이 프로젝트는 "root는 MFA 활성화 후 금고행, 이후 사용하지 않는다"는 원칙을
세웠으므로 MFA Delete는 그 원칙과 정면 충돌한다. versioning이 실수 삭제에 대한
1차 안전망 역할을 대신한다.

### backend.tf

```hcl
terraform {
  backend "s3" {
    bucket       = "hangar-tfstate-<ACCOUNT_ID>"
    key          = "dev/main/terraform.tfstate"
    region       = "ap-northeast-2"
    encrypt      = true
    use_lockfile = true
  }
}
```

- `encrypt = true`는 버킷 기본 암호화와 중복이지만 유지한다. 백엔드 블록만 읽고도
  의도를 알 수 있어야 한다.
- **backend 블록에는 변수를 쓸 수 없다.** 계정 ID는 하드코딩하거나
  `-backend-config="bucket=..."`로 주입해야 한다. 계정 ID는 비밀값이 아니므로
  이 랩에서는 하드코딩한다.

## 부트스트랩 (닭과 달걀)

**state를 담을 버킷 자체는 Terraform으로 만들 수 없다.** 버킷을 만드는 apply에
필요한 state를 저장할 곳이 아직 없기 때문이다. 따라서 이 버킷만 CLI로 수동
생성하고, 사용한 명령을 이 문서에 기록한다.

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET="hangar-tfstate-${ACCOUNT_ID}"
REGION="ap-northeast-2"

# 1) 버킷 생성 (us-east-1 외 리전은 LocationConstraint 필수)
aws s3api create-bucket \
  --bucket "$BUCKET" \
  --region "$REGION" \
  --create-bucket-configuration LocationConstraint="$REGION"

# 2) Versioning — 유일하게 명시적 활성화가 필요한 항목
aws s3api put-bucket-versioning \
  --bucket "$BUCKET" \
  --versioning-configuration Status=Enabled

# 3) 기본값 검증 (설정이 아니라 확인)
aws s3api get-bucket-encryption   --bucket "$BUCKET"
aws s3api get-public-access-block --bucket "$BUCKET"

# 4) Lifecycle — state 이력과 lock 객체 잔해의 무한 증가 방지
aws s3api put-bucket-lifecycle-configuration \
  --bucket "$BUCKET" \
  --lifecycle-configuration '{
    "Rules": [{
      "ID": "expire-noncurrent-state-versions",
      "Status": "Enabled",
      "Filter": {"Prefix": ""},
      "NoncurrentVersionExpiration": {"NoncurrentDays": 90},
      "Expiration": {"ExpiredObjectDeleteMarker": true}
    }]
  }'
```

3번의 두 명령은 출력이 나오면 통과다. 오류가 나면 기본값이 적용되지 않은
버킷이므로 명시적으로 설정해야 한다.

이 버킷은 Terraform 관리 대상이 **아니다.** 따라서 `default_tags`의
공통 태그가 붙지 않는다. 계정에 태그 없는 리소스가 하나 존재하는 상태가 정상이며,
그 예외의 근거가 이 문서다.

## 결과 (Consequences)

- **이 버킷의 가용성이 apply 가능성의 전제조건이 된다.** 버킷에 접근할 수 없으면
  인프라를 변경할 수 없다. 단일 장애점을 하나 만든 것이며, 그 대가로 state 안전성을 얻는다.
- **부트스트랩이 코드 밖에 남는다.** "인프라 전체가 코드로 재현된다"는 목표에 구멍이
  하나 생기고, 이 ADR이 그 구멍의 유일한 문서다. 계정을 새로 만들 때 이 문서의
  명령 블록이 첫 단계가 된다.
- **Phase 3 CI에 필요한 IAM 권한이 확정된다** — `s3:ListBucket`,
  `s3:GetObject`, `s3:PutObject`, `s3:DeleteObject`를 이 버킷 범위로.
  `use_lockfile`은 조건부 쓰기를 사용하므로 DynamoDB 권한은 불필요하다.
- **버킷 삭제는 versioning으로 막을 수 없다.** versioning은 객체 삭제의 안전망이지,
  버킷 삭제의 안전망이 아니다. 랩에서는 감수하되, 협업 단계에서는 버킷 정책이나
  SCP로 `s3:DeleteBucket`을 차단해야 한다.
- **state 이력이 90일 뒤 사라진다.** 그보다 오래된 상태로 되돌릴 필요가 생기는
  시나리오는 이 랩에 없다고 판단했다. 판단이 틀리면 lifecycle 값만 늘리면 된다(가역적).

## 재검토 트리거

- **CI(Phase 3)에서 apply를 실행할 때** — 잠금이 실제 경합 상황에서 동작하는지 검증하고,
  CI용 IAM 권한을 위 목록으로 최소화한다. 로컬과 CI가 같은 자격증명을 쓰지 않도록 분리한다.
- **협업자가 생길 때** — state 분리(`dev/network/`, `dev/cluster/`)와 접근 권한 분화.
  이 시점에 버킷 삭제 차단 정책도 함께 도입한다.
- **환경이 2개째 생길 때** — 키 스키마를 `prod/`로 확장. 이때 환경 간 state 접근을
  IAM으로 격리할지, 계정을 분리할지 결정해야 한다.
- **감사·컴플라이언스 요구가 생길 때** — SSE-KMS 승격, MFA Delete, state 전용 계정 격리 재검토.
- **`plan`이 체감될 정도로 느려질 때** — state 크기 문제이므로 분할을 검토한다.

## 참고

- [Backend Type: s3 — Terraform](https://developer.hashicorp.com/terraform/language/backend/s3) — `use_lockfile`, `dynamodb_table` deprecated 명시
- [Deprecation of dynamodb_table in Terraform S3 Backend — HashiCorp Discuss](https://discuss.hashicorp.com/t/deprecation-of-dynamodb-table-in-terraform-s3-backend/77060)
- [Amazon S3 now applies two security best practices to all new buckets by default (2023-04)](https://aws.amazon.com/about-aws/whats-new/2023/04/amazon-s3-security-best-practices-buckets-default) — Block Public Access·ACL 기본값
- [Configuring default encryption — Amazon S3 User Guide](https://docs.aws.amazon.com/AmazonS3/latest/userguide/default-bucket-encryption.html) — SSE-S3 기본 적용
