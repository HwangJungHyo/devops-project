# ADR-0003: IaC 도구 선정 — 선언형 DSL (Terraform)

- **상태**: Accepted
- **날짜**: 2026-08-21
- **관련 Phase**: Phase 2
- **영향받는 후속 Phase**: Phase 3 (CI에서 plan/apply 실행)
- **관련 ADR**: ADR-0002 (EKS 선정) / ADR-0004 (state 저장·잠금 — 이 결정에 의존한다)

## Context

`devops-infra` 레포에서 AWS 인프라(VPC, EKS, 노드그룹, ALB)를 코드로 관리해야 한다.
결정 시점의 조건은 다음과 같다.

- 1인 랩. 단일 AWS 계정, 세션마다 `apply`/`destroy`를 반복하는 학습용 환경이다.
- **목표가 "동작하는 인프라"가 아니라 "인프라를 다루는 역량의 습득과 증명"이다.**
  따라서 선택 기준에 *채용 시장에서 통용되는가*가 정당하게 포함된다.
- 로컬에 Terraform v1.15.8이 설치되어 있다 

후보군은 두 계열로 나뉜다.

- **선언형 DSL**: Terraform (HCL), OpenTofu (HCL), CloudFormation (YAML/JSON)
- **범용 언어형**: AWS CDK (TypeScript 등), Pulumi (TypeScript/Python 등)

시점 사실 — 2023년 HashiCorp가 Terraform 라이선스를 MPL 2.0에서 BSL 1.1로 전환했고,
그 반발로 커뮤니티 포크 OpenTofu가 Linux Foundation 산하로 출범했다. BSL은
"Terraform과 경쟁하는 제품을 만들어 판매하는 행위"를 제한하며, 자기 인프라를 관리하는
최종 사용자에게는 제약이 발생하지 않는다.

## Decision

**선언형 DSL 계열을 채택하고, 구현체로 Terraform을 사용한다.**

결정은 되돌리는 비용이 다른 두 층으로 구성된다.

| 층 | 결정 | 되돌리는 비용 |
|---|---|---|
| 계열 | 선언형 DSL — 원하는 상태를 기술하면 도구가 현재 상태와의 diff를 계산하는 모델 | **높음** (전체 코드 재작성) |
| 구현체 | Terraform | 낮음 (OpenTofu와 state·provider·모듈 호환) |

이 ADR의 무게는 첫 번째 층에 있다. 두 번째 층은 사실상 가역적이다.

**왜 선언형 계열인가** — 인프라 직무에서 지배적 모델이고, "원하는 상태 ↔
현재 상태의 diff(plan)"라는 상태 모델 자체가 이 프로젝트의 학습 대상이다.
범용 언어형(CDK/Pulumi)은 결국 내부에서 같은 선언형 엔진을 구동하며,
애플리케이션 개발자 경험에 최적화된 계층이라 이 프로젝트의 학습 목표와
초점이 다르다.

**왜 Terraform인가 (vs OpenTofu, CloudFormation)** — 학습 자료와 채용
시장의 사실상 표준이고, BSL은 자기 인프라를 관리하는 최종 사용자에게 실질
제약이 없으며(위 시점 사실), 로컬에 이미 설치·검증되어 있다. OpenTofu는
state·provider 호환으로 전환 비용이 낮아 라이선스 정책 리스크의 탈출구로
남는다. CloudFormation은 AWS 종속이라 계열 선택의 이식성 근거와 충돌한다.

## 버전 고정

- `required_version = "~> 1.15.0"` — 1.15.x 허용, 1.16 이상 차단.
  근거: 다른 마이너 버전의 Terraform(CI, 다른 머신)이 state를 건드려
  형식이 앞서 나가는 사고 방지. ADR-0004의 `use_lockfile`이 1.15에서
  지원되는 것도 하한의 근거다.
- AWS provider는 스캐폴드 시점의 최신 안정 버전을 `~>`(마이너 허용)로
  고정하고 `.terraform.lock.hcl`을 커밋한다.

## 결과 (Consequences)

- HCL이라는 DSL을 추가로 학습한다 — 단, 이 비용은 곧 학습 목표와 겹친다.
- HashiCorp의 라이선스·정책 변경 리스크를 안는다. 2층 구조 덕에
  OpenTofu 전환이라는 탈출구가 있어 리스크는 제한적이다.
- CI(Phase 3)는 Terraform 바이너리 설치·버전 일치를 전제로 설계해야 한다.

## 참고

- [HashiCorp License FAQ](https://www.hashicorp.com/license-faq) — BSL 적용 범위
- [OpenTofu](https://opentofu.org/) — 커뮤니티 포크, Linux Foundation
