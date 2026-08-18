# devops-project

Go 백엔드 API(`api-server`)를 컨테이너화부터 운영(모니터링·스케일링·DR)까지
단계적으로 구축하는 프로젝트. 인프라 코드, CI/CD, 의사결정 기록을 이 레포에서 관리한다.

> 이 README는 포트폴리오 표지다. Phase가 끝날 때마다 갱신한다.

## 아키텍처

(Phase 진행에 따라 다이어그램 추가 — `docs/images/`)

## 핵심 성과

(Phase 진행에 따라 정량 수치로 기록. 예: 이미지 크기, 배포 시간, 복구 시간)

| 항목 | Before | After | 근거 |
|---|---|---|---|
| 배포 중 요청 유실률 (SIGKILL → graceful shutdown) | 100% | 0% (5회 반복 재현) | [experiments/002](./docs/experiments/002-graceful-shutdown.md) |
| 런타임 이미지 크기 (빌드 플래그 적용) | 27.5MB | 23.4MB (-15%) | [experiments/001](./docs/experiments/001-image-size.md) |

## 진행 상태

로드맵: [docs/roadmap.md](./docs/roadmap.md)

| Phase | 주제 | 상태 |
|---|---|---|
| 1 | Containerization | **완료** ([회고](./docs/retrospective/phase-01.md)) |
| 2 | IaC + AWS 인프라 | 다음 |
| 3~12 | — | 대기 |

## 문서

- [의사결정 기록 (ADR)](./docs/adr/) — 왜 이렇게 만들었는가
- [트러블슈팅](./docs/troubleshooting/) — 무엇을 겪고 어떻게 풀었는가
- [실험·검증](./docs/experiments/) — 수치로 증명한 것
- [회고](./docs/retrospective/) — Phase별 배운 것
- [Runbook](./docs/runbook/) — 운영 절차 (Phase 진행하며 누적)
- [협업 규칙](./COLLABORATION.md) — AI 멘토와의 진행 방식
