# Phase 1 회고 — Containerization

- **기간**: 2026-08-14 ~ 2026-08-16 (계획 1주 대비 조기 완료)
- **DoD 달성**: 전부 — 이미지 빌드 성공, compose 단독 기동 + healthy,
  non-root(uid 10001), graceful shutdown, ADR 작성, runbook 작성·검증

## 산출물

`api-server` 레포:
- Dockerfile (멀티스테이지, non-root, 빌드 플래그, Go 버전 동기화 주석)
- docker-compose.yml (healthcheck, stop_grace_period 10s)
- graceful shutdown 구현 (`cmd/server/main.go`, 8초 timeout + Close 폴백)
- `/healthz` + 게이트된 `/_test/slow` 엔드포인트
- `scripts/exp002-shutdown.sh` (종료 실험 자동화)
- `.env.example`, `.dockerignore`

`devops-project` 레포:
- [ADR-0001 — 베이스 이미지 선정](../adr/0001-base-image.md) (Accepted)
- [Troubleshooting 001 — Go 버전 불일치](../troubleshooting/001-go-version-mismatch.md)
- [Troubleshooting 002 — localhost IPv6 리스너](../troubleshooting/002-localhost-ipv6-listener.md)
- [Experiments 001 — 이미지 크기 27.5→23.4MB (-15%)](../experiments/001-image-size.md)
- [Experiments 002 — graceful shutdown 유실률 100%→0%](../experiments/002-graceful-shutdown.md)
- [Runbook — 로컬 개발 환경](../runbook/local-development.md) (따라하기 검증 통과)

## 배운 것

1. `docker stop`은 기본 10초 유예 후 SIGKILL을 보낸다. 우아한 종료는 신호가
   아니라 그 신호를 잡은 코드가 만들며, 앱의 shutdown timeout(8초)은 이
   유예보다 짧아야 의미가 있다.
2. `localhost`는 이름이고 `127.0.0.1`은 주소다. 이름은 환경에 따라 `::1`로
   해석될 수 있어, 같은 포트라도 IPv4/IPv6 리스너가 다르면 다른 프로세스가
   응답할 수 있다.
3. 실제 부하 실험: "동작한다"의 증명과 "중요하다"의 증명은 다르다.
   in-flight 요청이 존재하는 상태를 만들어야 graceful shutdown의 가치가
   수치(유실률 100% vs 0%)로 드러났다.

## 다시 한다면

- 기록을 작업 직후에 정리한다. troubleshooting/001을 메모 상태로 두었다가
  Phase 종료 직전에 문장화했는데, 그 사이 세부가 흐려졌다.
- 검증 절차를 우회하지 않는다. runbook 따라하기 검증에서 `.env.example`이
  없는데 `vim .env`로 우회한것

## 퀴즈 결과

1차 (5문항): Q1 통과(결과 서술 누락), Q2 오답, Q3 통과, Q4a 보완·Q4b 정답,
Q5 오답. 재시험: Q1·Q2·Q4·Q5 전부 통과.

오답 원인: Q2(GOTOOLCHAIN)와 Q5(IPv6 진단) 모두 **본인이 문서에 정확히
써놓은 내용**이었다. 문서화와 체득은 다르며, 이것이 회고 퀴즈가 필요한 이유다.

## 다음 Phase에 넘기는 것

| 항목 | 처리 시점 |
|---|---|
| AWS 월 예산 상한·리전 확정 (사용자 결정) | **Phase 2 시작 전 필수** |
| go.mod ↔ Dockerfile 버전 동기화 CI 검사 | Phase 3 |
| ALB deregistration delay와 shutdown 8초의 대소 관계 재검토 | Phase 4 |
| JWT_SECRET 기본값 제거·주입 전략 | Phase 5 |
| distroless 전환 + wget healthcheck를 쉘 비의존 probe로 교체 | Phase 8 (로그 중앙화 완료 트리거) |
| 부하 도구 k6 전환 | CI 자동 판정 필요 시점 |
