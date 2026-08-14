# DevOps Mentoring 로드맵

## Phase 1: Containerization
- 애플리케이션 Docker 이미지 작성 (Dockerfile)
- 로컬 Docker Compose 환경 구성
- 이미지 최적화 (멀티스테이지 빌드, 레이어 캐싱)

## Phase 2: Infrastructure as Code + AWS 인프라 구성
- Terraform으로 기본 AWS 인프라 프로비저닝 (VPC, Subnet, SG)
- ECR 레지스트리 생성
- RDS/DB 인프라 구성
- IAM 역할 및 정책 설정

## Phase 3: CI/CD Pipeline
- GitHub Actions 기반 CI 구성 (빌드, 테스트, 이미지 푸시)
- CD 파이프라인 구성 (자동 배포)
- 브랜치 전략 정의

## Phase 4: Container Orchestration + Load Balancer
- ECS 또는 EKS 클러스터 구성
- ALB 연동 및 헬스체크 설정
- 서비스 디스커버리

## Phase 5: Secret Management
- AWS Secrets Manager 또는 Parameter Store 활용
- 환경변수/시크릿 주입 전략

## Phase 6: Deployment Strategy
- Blue/Green 또는 Rolling 배포 구현
- 롤백 전략 수립

## Phase 7: Monitoring + Alerting
- CloudWatch 메트릭 수집
- 대시보드 구성
- 알람 설정 (CPU, 메모리, 에러율 등)

## Phase 8: Logging + Observability
- 중앙 집중식 로그 수집 (CloudWatch Logs / ELK)
- 분산 트레이싱 도입
- 로그 기반 알림

## Phase 9: Auto Scaling
- Target Tracking 기반 오토스케일링
- 부하 테스트 및 튜닝

## Phase 10: Backup & Recovery
- DB 스냅샷/백업 정책
- 복구 절차 검증 (DR 시나리오)

## Phase 11: Security
- 네트워크 보안 (SG, NACL, WAF)
- 컨테이너 이미지 취약점 스캔
- HTTPS/TLS 적용

## Phase 12: 운영 문서 (Runbook)
- 배포 가이드
- 장애 대응 절차
- 온보딩 문서

---

**핵심 원칙**: 한번에 모든 것을 구축하는 게 아니라, Phase 1→12 순서대로
서비스가 성장하며 필요한 기능을 점진적으로 추가한다. 각 단계마다
"왜 이 기술을 선택했는지" 의사결정 기록을 남기는 것이 중요하다.
