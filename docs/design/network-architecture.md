# Network Architecture Design — EKS Lab (ap-northeast-2)

> 과제 2 산출물 / 결정 기록(Decision Record)
> 작성 기준일: 2026-08-19
> 전제: EKS 학습용 클러스터, 워커 노드 3개, 목표 처리량 10 req/s, **매 세션 종료 시 `terraform destroy`**

---

## 결정 요약

| # | 항목 | 결정 |
|---|---|---|
| 1 | VPC CIDR | `10.60.0.0/16` |
| 2 | AZ | `ap-northeast-2a / 2b / 2c` (단일 계정 — AZ 이름 사용, 멀티 계정 시 AZ ID 고정) |
| 3 | 서브넷 | Public 3 + Private 3 (AZ당 1쌍) |
| 4 | 노드 타입 | `t3.medium` × 3 (ON_DEMAND) |
| 5 | Private 서브넷 | **`/24` × 3** (계산상 최소 `/25`, 의도적 상향) |
| 6 | Public 서브넷 | **`/27` × 3** |
| 7 | NAT Gateway | **1개** (단일 AZ 공유) |
| 8 | Route Table | Public 1개 + Private 1개 |

![Network Architecture Diagram](../images/network-architecture.svg)

---

## 1. VPC CIDR — `10.60.0.0/16`

### 결정

```
VPC:  10.60.0.0/16   (65,536 IP)
```

### 근거: "겹치면 안 되는 대역" 목록에서 역산

VPN(Site-to-Site / Client VPN)으로 연결하는 순간, 양쪽 대역이 겹치면 라우팅이 성립하지 않는다. 겹친 뒤에는 **VPC CIDR을 바꿀 수 없으므로 VPC를 재생성해야 한다.** 그래서 "안 쓰는 대역"이 아니라 "**남들이 쓰는 대역**"부터 배제한다.

| 배제 대역 | 누가 쓰나 | 충돌 시 증상 |
|---|---|---|
| `192.168.0.0/24`, `192.168.1.0/24` | 가정용 공유기 기본값 (거의 모든 제조사) | 집에서 VPN 접속 시 라우팅 실패 |
| `172.17.0.0/16` | **Docker 기본 bridge (`docker0`)** | 노드/로컬에서 컨테이너 ↔ VPC 통신 깨짐 |
| `172.31.0.0/16` | **AWS Default VPC** | 다른 계정/리전 default VPC와 peering 불가 |
| `10.0.0.0/16`, `10.1.0.0/16` | 튜토리얼·Terraform 예제 기본값, 사내망 첫 블록 | 회사망 VPN 연결 시 충돌 확률 최상위 |
| `10.100.0.0/16`, `172.20.0.0/16` | **EKS Service CIDR 기본 후보** | Service ClusterIP ↔ VPC 주소 충돌 |

`10.60.0.0/16`은 위 5개 어디에도 걸리지 않는다.

### 왜 `/16`인가

- `/16` = 65,536 IP. 이 랩에는 100배 이상 과잉이다.
- 그럼에도 `/16`으로 잡는 이유: **VPC 내부 IP는 과금 대상이 아니다.** 아껴서 얻는 게 없다.
- 반대로 좁게 잡으면 잃는 게 크다 — VPC CIDR은 생성 후 축소/변경 불가. (Secondary CIDR 추가는 가능하지만 비연속 대역이 생겨 관리가 나빠진다.)
- 실무 제약이 붙는 경우(사내 IPAM에서 `/20`만 할당) 서브넷 설계는 그대로 두고 프리픽스만 좁히면 된다 → §5의 계산이 그 판단 근거가 된다.

### 같이 결정해야 하는 것: Kubernetes Service CIDR

Pod IP는 VPC 서브넷에서 나오지만, **Service ClusterIP는 VPC와 무관한 별도 대역**이다. 지정하지 않으면 EKS가 `10.100.0.0/16` 또는 `172.20.0.0/16` 중 하나를 자동 선택한다.

```hcl
# aws_eks_cluster
kubernetes_network_config {
  service_ipv4_cidr = "172.24.0.0/16"
}
```

- 제약: RFC1918 내, `/12`~`/24` 사이, **VPC CIDR과 겹치면 안 됨**, **클러스터 생성 후 변경 불가**
- 명시 지정을 권장한다. 자동 선택에 맡기면 나중에 VPN 상대편 대역과 겹쳤을 때 클러스터 재생성 외에 방법이 없다.

> 출처: [EKS — KubernetesNetworkConfig](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-properties-eks-cluster-kubernetesnetworkconfig.html)

### 최종 대역 배치도

```
10.60.0.0/16
├── 10.60.0.0/24    Private-a   (nodes/pods, X-ENI)
├── 10.60.1.0/24    Private-b
├── 10.60.2.0/24    Private-c
├── 10.60.3.0/24 ~ 10.60.99.0/24   ← Private 확장 예약 (97개)
├── 10.60.100.0/27  Public-a    (NAT, ALB)
├── 10.60.100.32/27 Public-b
├── 10.60.100.64/27 Public-c
└── 10.60.101.0/24 ~            ← Public / 기타(DB, VPC Endpoint) 확장 예약
```

Private을 앞쪽 연속 블록에 배치했다. 다만 route summarization(경로 요약)은
이 랩에 적용하지 않는다. 요약이 이점을 갖는 것은 VPN·Transit Gateway 등
외부 사설망에 경로를 광고할 때인데, 이 랩에는 그 연결이 없다. 인터넷 경유
접근(ALB 공인 주소)만 존재하므로 내부 사설 대역을 외부에 알릴 일이 없다.
VPN/TGW 도입 시 /18 경계 정렬을 재검토한다.

---

## 2. AZ 선택 — `ap-northeast-2a / 2b / 2c`

### 선택 기준 (우선순위 순)

1. **EKS 최소 요건**: 클러스터 서브넷은 최소 2개 AZ. ALB(Application Load Balancer)도 최소 2개 AZ 서브넷을 요구한다. → 2개가 하한, 3개는 쿼럼 감각(etcd/HA 패턴)을 익히기 위한 선택.
2. **인스턴스 타입 가용성**: AZ마다 제공되는 인스턴스 타입이 다르다. 선택 전 반드시 확인:
   ```bash
   aws ec2 describe-instance-type-offerings \
     --location-type availability-zone \
     --filters Name=instance-type,Values=t3.medium \
     --region ap-northeast-2 --output table
   ```
3. **AZ 이름 vs AZ ID**: AWS는 계정마다 `ap-northeast-2a`라는 이름이
   가리키는 실제 물리 AZ를 다르게 매핑한다. 계정 간 VPC Peering·Transit
   Gateway를 연결할 때 물리 AZ가 어긋나면 cross-AZ 비용과 지연이
   발생하므로, 그때는 AZ ID(`apne2-az1` 등) 기준 정렬이 필수다.
   이 랩은 단일 계정(IAM만 분리)이므로 AZ 이름으로 충분하다.
   멀티 계정 전환(과제 1의 엔터프라이즈 설계 적용) 시 AZ ID 고정으로
   전환한다.
   ```bash
   aws ec2 describe-availability-zones --region ap-northeast-2 \
     --query 'AvailabilityZones[].[ZoneName,ZoneId,State]' --output table
   ```

4. **Cross-AZ 데이터 전송 요금**: AZ 간 트래픽은 양방향 과금. AZ를 늘릴수록 노드 간 통신 요금이 붙는다. 랩 규모에선 무시 가능하지만, 3개를 넘길 이유가 없다는 근거는 된다.

> ap-northeast-2는 AZ 4개(2a/2b/2c/2d)로 알고 있으나, **위 CLI로 직접 확인 후 확정할 것.** 리전 AZ 구성은 증설된다.

### 4개가 아니라 3개인 이유

- 홀수(3)여야 향후 etcd/Zookeeper류 쿼럼 기반 워크로드를 얹을 때 자연스럽다.
- AZ 1개 추가 = 서브넷 2개 + NAT 검토 + IP 대역 추가. 얻는 가용성 대비 관리 복잡도가 랩 목적에 맞지 않는다.

---

## 3. 서브넷 구성

### 구성

| 서브넷 | AZ | CIDR | 인터넷 경로 | 들어가는 것 |
|---|---|---|---|---|
| `public-a` | 2a | `10.60.100.0/27` | IGW | **NAT Gateway**, Internet-facing ALB ENI |
| `public-b` | 2b | `10.60.100.32/27` | IGW | Internet-facing ALB ENI |
| `public-c` | 2c | `10.60.100.64/27` | IGW | Internet-facing ALB ENI |
| `private-a` | 2a | `10.60.0.0/24` | NAT | **워커 노드**, **Pod IP**, **EKS X-ENI**, Internal ALB ENI |
| `private-b` | 2b | `10.60.1.0/24` | NAT | 〃 |
| `private-c` | 2c | `10.60.2.0/24` | NAT | 〃 |

### 배치 근거

**노드는 Private에 둔다.** 노드에 공인 IP를 붙이지 않는 것이 기본. 인바운드는 ALB만 받고, 아웃바운드(ECR pull, yum, API 호출)는 NAT을 경유한다.

**ALB는 Public / Internal ALB는 Private.** AWS Load Balancer Controller가 태그로 서브넷을 자동 탐색하므로 태그가 필수다:

```
public-*   →  kubernetes.io/role/elb           = 1
private-*  →  kubernetes.io/role/internal-elb  = 1
```

**EKS 컨트롤 플레인 ENI(X-ENI)는 Private 서브넷에 배치, 노드와 공유.**

- 클러스터 생성 시 지정한 서브넷마다 EKS가 Cross-Account ENI를 만든다. API server ↔ 노드 통신 경로다.
- EKS는 **X-ENI를 최대 4개까지 생성**할 수 있고, **클러스터 업그레이드 중에는 새 X-ENI를 만든 뒤 기존 것을 삭제**한다 → 업그레이드 순간 IP 소요가 일시적으로 2배.
- AWS 권장은 클러스터 연결 서브넷 최소 `/28`. IP가 빠듯한 환경에서는 **X-ENI 전용 `/28` 서브넷 분리**를 권한다.

> **대안 검토 — X-ENI 전용 서브넷 분리 (기각)**
> 프로덕션에서 Private 서브넷이 `/22` 이하로 좁아지면, 노드/Pod가 IP를 먹어치워 업그레이드 시점에 X-ENI를 만들 IP가 없어 **업그레이드가 실패**하는 사고가 난다. 이때는 전용 `/28` 3개로 분리하는 게 정석이다.
> 이 랩은 Private을 `/24`로 잡아 X-ENI 8개분(업그레이드 포함) 여유가 §5 계산에 이미 반영돼 있으므로, 서브넷 3개를 더 만드는 복잡도가 이득보다 크다고 판단해 **공유**로 간다.
> **전환 트리거: Private 서브넷을 `/24`보다 좁히기로 결정하는 순간, 이 결정을 재검토한다.**

> 출처: [EKS Best Practices — VPC and Subnet Considerations](https://docs.aws.amazon.com/eks/latest/best-practices/subnets.html), [Optimizing IP Address Utilization](https://docs.aws.amazon.com/eks/latest/best-practices/ip-opt.html)

---

## 4. 노드 인스턴스 타입 — `t3.medium` × 3

### 선택

```
t3.medium  (2 vCPU / 4 GiB / ENI 3개 / ENI당 IPv4 6개)
ON_DEMAND, Managed Node Group, desired 3 / min 3 / max 6
```

### 근거

**10 req/s는 CPU 제약이 아니다.** 응답시간 200ms를 가정해도 평균 동시 처리 2건. vCPU 1개로 남는다. 즉 이 워크로드에서 인스턴스 타입을 결정하는 건 CPU·메모리가 아니라 **Pod 슬롯 수와 시간당 비용**이다.

**max-pods 검증** (과제 1 공식):

```
max Pods = ENI 수 × (ENI당 IP 수 − 1) + 2
         = 3 × (6 − 1) + 2 = 17
```

노드당 17 슬롯 중 시스템 몫을 빼면:

| 항목 | 슬롯 소모 |
|---|---|
| `aws-node` (VPC CNI) | 0 — hostNetwork (공식의 `+2` 몫) |
| `kube-proxy` | 0 — hostNetwork (공식의 `+2` 몫) |
| `ebs-csi-node` | 0 — hostNetwork |
| CoreDNS | 클러스터 전체 2개 (노드당 아님) |
| metrics-server, AWS LB Controller | 클러스터 전체 소수 |

→ 노드당 **애플리케이션 Pod 약 14개 확보**, 3노드 기준 **약 42개**. 10 req/s 워크로드에 충분하다.

### 기각한 대안

| 후보 | max-pods | 기각 사유 |
|---|---|---|
| `t3.small` (2 vCPU / 2 GiB, ENI 3 × IP 4) | `3×3+2 = 11` | 시스템 Pod 제하면 앱 Pod 8개대. 메모리 2 GiB는 kubelet + 시스템 컴포넌트로 상당 부분 소진되어 학습 중 OOM 위험 |
| `t3.large` (2 vCPU / 8 GiB, ENI 3 × IP 12) | `3×11+2 = 35` | 이 워크로드엔 과잉. **단, Pod 밀도가 병목이 되면 1순위 승격 후보** |
| `m5.large` (2 vCPU / 8 GiB, ENI 3 × IP 10) | `3×9+2 = 29` | burstable 대비 시간당 비용 높음. 랩에서 baseline 성능 보장이 필요 없음 |

### 부가 결정

- **ON_DEMAND**: SPOT이 더 싸지만 중단 이벤트가 학습 중 노이즈가 된다. 나중에 "노드 중단 시 Pod 재스케줄" 실습을 할 때 SPOT 노드그룹을 별도로 추가하는 게 낫다.
- **t3는 credit 기반(burstable)**: 10 req/s는 baseline CPU를 넘지 않으므로 문제없다. 다만 `unlimited` 모드에서 baseline 초과 시 추가 과금이 붙으므로, 부하 테스트를 돌릴 계획이면 `standard` 모드 또는 `c5.large`로 전환 검토.
- **이 선택이 §5의 입력값이다**: `ENI 3 × IP 6 = 노드당 18 IP`.

---

## 5. Private 서브넷 크기 계산 ★ 핵심 산출물

### Step 1 — 노드 1대가 서브넷에서 가져갈 수 있는 IP 상한

과제 1에서 확인한 대로 IP는 **Pod 단위가 아니라 ENI 단위 덩어리로 예약**된다. warm ENI가 붙는 순간 Pod가 0개여도 그 ENI의 IP 전량이 서브넷에서 사라진다.

```
t3.medium: ENI 3개 × ENI당 IPv4 6개 = 18 IP / 노드
```

이 18개는 Pod가 몇 개 뜨든 관계없이 **CNI가 최대로 예약할 수 있는 양**이다.
(내역: Pod 사용 가능 15 + ENI primary IP 3)

> 과제 1의 3배 증폭 결론이 여기서 그대로 계산에 들어간다. Pod 15개를 위해 18 IP를 잡아두는 것이며, `WARM_ENI_TARGET=1` 기본값이 이를 강제한다.

### Step 2 — AZ 1개에 몰릴 수 있는 노드 수

Managed Node Group `desired 3 / max 6`, 3 AZ 분산.

**평균(6÷3=2)이 아니라 worst case로 잡아야 한다.** 이유:

- ASG의 AZ 리밸런싱은 즉시가 아니다. 스케일아웃 직후 한쪽에 몰린 상태가 지속될 수 있다.
- 특정 AZ에 인스턴스 용량이 없으면 나머지 AZ로 몰린다.
- 노드 롤링 업데이트 중에는 old/new 노드가 **동시에 존재**한다.

```
AZ당 노드 예산 = 4대  (max 6 중 편중 + 롤링 중 중복 흡수)
```

### Step 3 — Private 서브넷 1개당 IP 수요

| 항목 | 계산 | IP |
|---|---|---:|
| 워커 노드 + Pod | 4대 × 18 | **72** |
| EKS X-ENI | 최대 4 × 업그레이드 시 2배 | **8** |
| Internal ALB | ALB 최소 요구 8 free IP | **8** |
| AWS 예약 (`.0` / `.1` / `.2` / `.3` / 마지막) | 고정 | **5** |
| | **소계** | **93** |

> AWS 예약 5개: 네트워크 주소, VPC 라우터(`.1`), Amazon DNS(`.2`), 향후 예약(`.3`), 브로드캐스트(마지막). 서브넷마다 무조건 빠진다.

> **X-ENI 8개 계상 — 가정 명시**
> X-ENI 최대 4개, 업그레이드 시 순간 최대 8개는 **클러스터 전체 기준**이다.
> EKS는 클러스터 생성 시 지정한 서브넷 중 일부를 골라 X-ENI를 분산 배치하는데,
> **어느 서브넷에 만들지는 EKS(AWS 관리 영역)가 정하며 사용자가 제어할 수 없다.**
> Pod를 어느 노드에 띄울지 정하는 쿠버네티스 스케줄러와는 별개 영역이다.
> 배치 결과를 사전에 알 수 없으므로, 한 서브넷에 몰리는 경우를 흡수하려고
> 서브넷당 8로 계상했다. 의도적 과다 계상이며 이 항목을 3으로 낮춰도
> 소계는 88, 결정(`/24`)은 변하지 않는다.

### Step 4 — 넷마스크 결정

| 넷마스크 | 총 IP | 사용 가능 | 93 충족? | 여유 |
|---|---:|---:|---|---:|
| `/26` | 64 | 59 | ✗ **부족** | — |
| `/25` | 128 | 123 | ✓ | 30 (24%) |
| **`/24`** | **256** | **251** | **✓** | **158 (63%)** |

### 결정: `/24` × 3 — 계산상 최소치(`/25`)보다 한 단계 위

계산 결과는 `/25`가 최소다. 그럼에도 `/24`를 택한다:

**① 서브넷 CIDR은 생성 후 변경 불가.**
늘리려면 서브넷을 지우고 다시 만들어야 하고, 그러려면 그 안의 노드그룹·ENI를 전부 걷어내야 한다. **틀렸을 때의 비용이 비대칭적으로 크다.**

**② VPC 내부 IP는 무료다.**
`/16` VPC에서 `/24` 3개는 전체의 1.2%. 아껴서 얻는 게 0이다.

**③ 인스턴스 타입을 바꿔도 서브넷 재설계를 강제하지 않는다.** ← 실질적으로 가장 큰 이유
`t3.large`(ENI 3 × IP 12 = 노드당 **36 IP**)로 올린다고 가정하면:

```
4대 × 36 = 144
+ X-ENI 8 + ALB 8 + 예약 5
= 165 IP
```

- `/25`(123) → **터진다.** 서브넷 재생성 필요.
- `/24`(251) → 버틴다. 노드 타입만 바꾸면 끝.

즉 `/24`의 가치는 "IP가 넉넉하다"가 아니라 **"§4의 결정을 나중에 번복할 수 있게 만든다"**는 데 있다.

**④ warm pool 소모량은 추정치다.**
`WARM_ENI_TARGET` / `WARM_IP_TARGET` 설정, Pod 삭제 후 30초 cool-down 캐시, DaemonSet 추가 등에 따라 실제 소모는 변한다. 계산 자체에 오차가 있으므로 흡수 여유가 필요하다.

> 이것이 멘토가 말한 **"프라이빗 서브넷은 넉넉히"의 정량적 근거**다.
> "넉넉히"가 감이 아니라 `노드 수 × ENI 수 × ENI당 IP + X-ENI + LB + 예약 5`라는 식으로 환산되고, 여기에 **"타입 변경을 흡수할 수 있는가"**라는 조건을 하나 더 걸어 한 단계 상향한 결과가 `/24`다.

### 검증 도구

AWS가 이 계산을 대신 해주는 스프레드시트를 배포한다. `WARM_IP_TARGET` / `WARM_ENI_TARGET` 조합별 IP 소모를 시뮬레이션한다.
→ [subnet-calc.xlsx (aws-eks-best-practices)](https://github.com/aws/aws-eks-best-practices/blob/master/latest/bpg/networking/subnet-calc/subnet-calc.xlsx)

**위 계산을 이 계산기에 넣어 교차 검증할 것.**

### 실제 소모량 확인 명령

```bash
# 서브넷별 잔여 IP
aws ec2 describe-subnets --region ap-northeast-2 \
  --filters Name=vpc-id,Values=<vpc-id> \
  --query 'Subnets[].[Tags[?Key==`Name`].Value|[0],CidrBlock,AvailableIpAddressCount]' \
  --output table

# 노드별 CNI 실제 할당 현황
kubectl get cninode -o wide          # VPC CNI v1.18+
kubectl exec -n kube-system <aws-node-pod> -- \
  curl -s http://localhost:61679/v1/enis | jq
```

---

## 6. Public 서브넷 크기 — `/27` × 3

### 서브넷 1개당 수요

| 항목 | IP |
|---|---:|
| AWS 예약 | 5 |
| NAT Gateway ENI | 1 |
| Internet-facing ALB (AWS 최소 요구) | 8 |
| Bastion / 예비 | 1 |
| **소계** | **15** |

### 넷마스크 비교

| 넷마스크 | 사용 가능 | 판정 |
|---|---:|---|
| `/28` | 11 | ✗ **AWS가 명시적으로 금지.** ALB는 서브넷당 `/27` 이상 + free IP 8개를 요구한다 |
| **`/27`** | **27** | **✓ 채택** — 15 사용, 12 여유 |
| `/26` | 59 | 과잉 — 단, ALB를 2개 이상 띄울 계획이면 이쪽 |

> ALB 요건 원문: 각 AZ 서브넷은 최소 `/27` 비트마스크와 free IP 8개를 가져야 하며, 이 8개는 로드밸런서 스케일아웃에 사용된다. 부족하면 노드 교체가 실패하고 5xx/타임아웃이 발생한다.
> 출처: [ELB — Application Load Balancers](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/application-load-balancers.html)

### §5와 다른 논리를 적용한 이유

Private에는 "계산 최소치보다 한 단계 위"를 적용했는데 Public에는 안 했다. 일관성 없어 보이지만 의도적이다.

- **Private의 IP 소모는 탄력적이다.** 노드 수, 인스턴스 타입, warm pool 설정에 따라 배수로 튄다. 예측 오차가 크므로 여유가 필요하다.
- **Public의 내용물은 유한하고 예측 가능하다.** NAT ENI 1개 + LB ENI 8개. Pod처럼 늘어나지 않는다.

**상향 트리거**: Ingress를 2개 이상(= ALB 2개 이상) 만들기로 하면 ALB당 8 IP가 추가되므로 `/26`으로 상향한다. 현재 전제는 Ingress 1개다.

---

## 7. NAT Gateway — 1개 (단일 AZ 공유)

### 옵션 비교

| 옵션 | 시간당 비용 | 가용성 | Cross-AZ 요금 | apply/destroy 시간 |
|---|---|---|---|---|
| AZ당 1개 (3개) | ×3 | AZ 장애 격리 | 없음 | 가장 느림 |
| **1개 공유** | **×1** | **단일 장애점** | **발생** | **빠름** |
| 없음 (VPC Endpoint만) | NAT 0 / Interface Endpoint는 시간당 과금 | — | 없음 | 설정 복잡 |

### 결정: 1개

**① 학습 환경에 HA 요구가 없다.** AZ 장애로 실습이 중단되는 리스크는 수용 가능하다. 프로덕션이라면 정반대 결론이다.

**② "매 세션 destroy" 제약의 실제 의미는 비용이 아니라 시간이다.**
직관적으로는 "destroy하니까 비용 걱정 없다 → 3개 써도 된다"로 갈 수 있는데, 그게 아니다. NAT Gateway는 **생성/삭제가 느린 리소스**(각각 수 분)다. 3개면 매 세션 `apply`/`destroy`가 그만큼 길어지고, 그 대기 시간이 세션마다 반복된다. **세션당 반복 비용이 시간당 요금보다 크다.**

**③ 트레이드오프를 명시한다 (수용):**

- `public-a`가 속한 AZ 장애 시 **전 노드의 아웃바운드가 끊긴다.** ECR pull 실패 → 새 Pod 기동 불가.
- `private-b`, `private-c` 노드의 아웃바운드는 AZ를 넘어가므로 cross-AZ 데이터 전송 요금이 붙는다. 랩 트래픽 규모에서는 무시 가능.

**④ 프로덕션 전환 경로를 코드에 남긴다.**

```hcl
# terraform-aws-modules/vpc 기준
enable_nat_gateway     = true
single_nat_gateway     = true   # ← lab
one_nat_gateway_per_az = false  # ← prod에서 true로 토글
```

변수 하나로 뒤집히도록 짜둔다. **단, 토글 시 Private route table도 3개로 분리해야 한다** (§8 참조).

### 함께 결정: S3 Gateway Endpoint는 무조건 추가

```hcl
resource "aws_vpc_endpoint" "s3" {
  vpc_id            = aws_vpc.this.id
  service_name      = "com.amazonaws.ap-northeast-2.s3"
  vpc_endpoint_type = "Gateway"
  route_table_ids   = [aws_route_table.private.id]
}
```

- **Gateway Endpoint는 시간당 과금이 없다** (Interface Endpoint와 다름).
- **ECR 이미지 레이어는 실제로 S3에서 내려온다.** 이걸 붙이면 이미지 pull 트래픽이 NAT을 우회 → NAT data processing 요금($/GB)이 크게 준다.
- 랩에서도 붙일 이유가 충분한, 사실상 유일한 "공짜 최적화".

반대로 ECR API/DKR Interface Endpoint는 시간당 과금이 있으므로 랩에서는 생략하고 NAT을 경유시킨다.

> 비용 수치는 리전·시점에 따라 변한다. 실제 금액은 [AWS Pricing Calculator](https://calculator.aws/)에서 `ap-northeast-2` 기준으로 확인할 것. 이 문서는 **상대 비교와 과금 구조**(NAT = 시간당 + 처리 GB당, Gateway Endpoint = 무료)만 근거로 삼는다.

---

## 8. 라우팅

### Public Route Table (1개, Public 서브넷 3개 공유)

| Destination | Target | 비고 |
|---|---|---|
| `10.60.0.0/16` | `local` | 자동 생성. **삭제·수정 불가** |
| `0.0.0.0/0` | `igw-xxxxx` | 이 한 줄이 "퍼블릭 서브넷"의 정의 |

Public 서브넷 3개가 동일한 라우팅을 쓰므로 라우트 테이블은 1개면 충분하다.

### Private Route Table (1개, Private 서브넷 3개 공유)

| Destination | Target | 비고 |
|---|---|---|
| `10.60.0.0/16` | `local` | 자동 생성 |
| `0.0.0.0/0` | `nat-xxxxx` (AZ-a) | NAT 1개 구성이므로 3개 서브넷이 동일 NAT 참조 |
| `pl-xxxxx` (S3 prefix list) | `vpce-xxxxx` | Endpoint를 route table에 연결하면 **자동 추가** |

### 핵심 개념

- **퍼블릭/프라이빗을 가르는 것은 서브넷의 속성이 아니라 라우트 테이블의 `0.0.0.0/0` 대상이다.** IGW를 가리키면 퍼블릭, NAT을 가리키면 프라이빗. 서브넷 자체에는 public/private 플래그가 없다.
- **NAT Gateway는 반드시 퍼블릭 서브넷에 있어야 한다.** NAT 자신이 IGW로 나가는 경로를 가져야 하기 때문. NAT을 프라이빗 서브넷에 두는 건 가장 흔한 초보 실수이며, 증상은 "노드가 인터넷이 안 됨"으로 나타난다.
- **`local` 경로는 삭제·변경할 수 없다.** VPC 내부 통신은 라우팅 설정과 무관하게 항상 가능하며, 차단은 Security Group / NACL의 몫이다.

### 흔한 함정 (배포 전 체크)

| 함정 | 증상 | 확인 |
|---|---|---|
| NAT을 AZ당 1개로 바꾸면서 route table을 1개로 방치 | NAT 3개 요금 내면서 실제로는 1개만 쓰이고 cross-AZ 요금도 그대로 | Private RT 개수 = NAT 개수인지 |
| Public 서브넷에 `map_public_ip_on_launch` 미설정 | 퍼블릭 서브넷에 띄운 인스턴스가 공인 IP 없이 격리됨 | `aws ec2 describe-subnets` 의 `MapPublicIpOnLaunch` |
| 서브넷 태그 누락 | AWS LB Controller가 서브넷을 못 찾아 Ingress 생성 실패 | `kubernetes.io/role/elb`, `kubernetes.io/role/internal-elb` |

### 검증

```bash
# 라우트 테이블 ↔ 서브넷 연결 확인
aws ec2 describe-route-tables --region ap-northeast-2 \
  --filters Name=vpc-id,Values=<vpc-id> \
  --query 'RouteTables[].{RT:RouteTableId,Assoc:Associations[].SubnetId,Routes:Routes[].[DestinationCidrBlock,GatewayId,NatGatewayId]}'

# 노드에서 아웃바운드 실증
kubectl run nettest --rm -it --image=public.ecr.aws/amazonlinux/amazonlinux:2023 -- \
  sh -c "curl -sI https://ecr.ap-northeast-2.amazonaws.com | head -1"
```

---

## 9. 미확정 / 배포 전 검증 항목

이 문서에서 **확인하지 못했거나 시점에 따라 달라지는 것들.** 그대로 믿지 말고 검증할 것.

| # | 항목 | 확인 방법 |
|---|---|---|
| 1 | ap-northeast-2의 실제 AZ 개수 및 AZ ID 매핑 | `aws ec2 describe-availability-zones` |
| 2 | 선택한 AZ 3개 모두에서 `t3.medium` 제공 여부 | `aws ec2 describe-instance-type-offerings` |
| 3 | NAT / EC2 / Endpoint 실제 단가 | AWS Pricing Calculator (본문에 금액을 쓰지 않은 이유) |
| 4 | `kubernetes.io/cluster/<name> = shared` 태그가 여전히 필요한지 | AWS Load Balancer Controller 버전에 따라 요구사항이 달라졌다. 사용 중인 컨트롤러 버전 문서에서 확인 |
| 5 | `10.60.0.0/16`이 회사망/VPN 상대 대역과 겹치지 않는지 | **사내 IPAM·네트워크 담당 확인이 최종 근거.** 본문 §1은 "일반적으로 흔한 대역 회피" 논리일 뿐 |
| 6 | Private `/24` 계산의 교차 검증 | `subnet-calc.xlsx`에 §5 파라미터 입력 |

---

## 10. 참고 문서

- [EKS Best Practices — Amazon VPC CNI](https://docs.aws.amazon.com/eks/latest/best-practices/vpc-cni.html) — max-pods 공식, `WARM_ENI_TARGET`
- [EKS Best Practices — VPC and Subnet Considerations](https://docs.aws.amazon.com/eks/latest/best-practices/subnets.html) — X-ENI, 서브넷 태그
- [EKS Best Practices — Optimizing IP Address Utilization](https://docs.aws.amazon.com/eks/latest/best-practices/ip-opt.html) — X-ENI 최대 4개, `/28` 권장
- [ELB — Application Load Balancers](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/application-load-balancers.html) — `/27` + free IP 8개
- [subnet-calc.xlsx](https://github.com/aws/aws-eks-best-practices/blob/master/latest/bpg/networking/subnet-calc/subnet-calc.xlsx) — IP 소모 시뮬레이터
- [eni-max-pods.txt](https://github.com/aws/amazon-vpc-cni-k8s/blob/master/misc/eni-max-pods.txt) — 인스턴스 타입별 max-pods 원본 표
