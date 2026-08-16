# 002: Graceful shutdown의 in-flight 요청 보존 효과

- **날짜**: 2026-08-16
- **Phase**: Phase 1
- **유형**: 파괴 검증

## 가설 / 목표

종료 신호를 받는 순간 처리 중이던 요청이 SIGTERM(graceful)에서는 완료되고
SIGKILL에서는 유실된다는 것을 확인한다.

증명 대상이 아닌 것 (과장 방지):
- 무중단 배포 — 종료 후 새 요청은 graceful shutdown도 받지 않는다
- 다중 replica 트래픽 전환 — 로드밸런서와 replica가 없으므로 측정 불가

Go의 http.Server.Shutdown()이 보장하는 범위는 "이미 받은 요청은 가능한 한 완료,
종료 후 새 요청은 거부"까지다.

## 방법

| 항목 | 값 |
|---|---|
| 부하 도구 | hey v0.1.5 (`github.com/rakyll/hey`, go install, go1.26.4 빌드) |
| 부하 조건 | `-n 20 -c 20 -t 15` (지속 부하 아님, 동시 20개 1회) |
| 요청 처리시간 | 5초 (`/_test/slow?duration=5s`) |
| 애플리케이션 shutdown timeout | 8초 (`cmd/server/main.go` shutdownTimeout) |
| docker stop timeout | 15초 |
| hey 요청 timeout | 15초 |
| 신호 시점 | 진입 로그 20개 확인 직후 (스크립트 자동 판정) |
| 반복 | 각 조건 5회 |
| 성공 판정 | HTTP 200 |
| 실패 판정 | 전송 오류 또는 200 이외 응답 |
| 결과 원본 | hey stdout 전체 + 컨테이너 로그 (CSV 미사용) |

지속 부하(-z)를 쓰지 않은 이유: 종료 이후 새로 발생한 요청의 연결 거부가
in-flight 요청의 유실과 섞여 graceful shutdown 효과만 분리할 수 없다.

-disable-keepalive 미사용: worker당 요청이 1개라 연결 재사용이 발생하지 않아
결과에 영향이 없다. 불필요한 옵션은 통제 변수만 늘린다.

전제: Dockerfile의 ENTRYPOINT가 exec 형식이라 서버 바이너리가 PID 1로
SIGTERM을 직접 수신한다.

실행: `api-server/scripts/exp002-shutdown.sh none|stop|kill`

## 결과

동일 이미지 ID (전 11회 실행):
`sha256:65ecd9eba8b7f0be2ee784ef92cb8bdb02eaa5b1700e10338167d3dab7426aea`

무효 실행: 없음 (INVALID 마킹 0건)

| 조건 | 실행 | 총 요청 | 성공 | 실패 | 실패율 |
|---|---|---|---|---|---|
| 기준선 (종료 없음) | 1회 | 20 | 20 | 0 | 0% |
| docker stop (SIGTERM) | 5회 | 100 | 100 | 0 | **0%** |
| docker kill (SIGKILL) | 5회 | 100 | 0 | 100 | **100%** |

신호 시점 in-flight 수 (컨테이너 로그 `in-flight=` 최대값):

| 조건 | 회차별 in-flight |
|---|---|
| stop | 20 / 20 / 20 / 20 / 20 |
| kill | 20 / 20 / 20 / 20 / 20 |

graceful shutdown 소요 시간 (Shutdown signal received → Server stopped,
로그 초 단위 기준):

| 회차 | 소요 |
|---|---|
| 1~5회 | 약 4~5초 (전 회차 8초 timeout 미도달, "Graceful shutdown failed" 로그 없음) |

증적 (EVIDENCE.txt 요약 — 원본은 실행 시 `api-server/results/`에 생성, 미커밋):

```text
########## 2. 성공/실패 집계
RUN                                 OK   ERR
20260816-233212-none                20     0
20260816-233220-stop                20     0
20260816-233229-stop                20     0
20260816-233237-stop                20     0
20260816-233245-stop                20     0
20260816-233253-stop                20     0
20260816-233302-kill                 0    20
20260816-233305-kill                 0    20
20260816-233309-kill                 0    20
20260816-233312-kill                 0    20
20260816-233315-kill                 0    20
```

stop 조건 대표 1회분 (신호 → 5초 요청 완료 → 종료, timeout 미도달):

```text
--- 20260816-233220-stop
  max in-flight: 20
  signal-at: 2026-08-16 23:32:22.978939200+09:00
  2026/08/16 14:32:23 Shutdown signal received
  2026/08/16 14:32:23 Shutting down server (timeout=8s)...
  2026/08/16 14:32:28 Server stopped
```

hey 상태 코드 분포 (stop 대표 1회분): `[200] 20 responses`

## 해석

(작성 전 확인할 것)
- stop에서 실패 발생 → shutdown 구현, timeout, 신호 전달 중 하나의 문제
- kill에서 성공 발생 → 신호가 늦었거나 요청 진입 판정 오류
- 총 요청이 20이 아님 → 해당 실행 무효
- 기준선부터 실패 → 종료 실험 이전에 부하 조건 재조정
- stop 소요 시간이 8초에 근접 → shutdown timeout이 실제로 걸린 것이므로
  "Graceful shutdown failed" 로그 유무를 반드시 확인

이 실험은 스스로 정한 무효 판정 기준 네 가지 — stop에서의 실패, kill에서의 성공,
총 요청 수 20 미달, 기준선 실패 — 가 11회 전 실행에서 하나도 발생하지 않았으므로
결과를 유효한 것으로 인정한다. 동일 이미지·동일 부하·동일 신호 시점에서 성공률이
100%와 0%로 갈렸고, 두 조건의 유일한 차이는 신호 종류다. shutdown 소요 시간이
요청 처리시간과 일치했다는 점에서 SIGTERM 조건의 성공은 우연이 아니라 서버가
실제로 진행 중인 요청을 기다린 결과다.
SIGKILL 조건의 100% 유실은 예외 상황이 아니라 **graceful shutdown이 없을 때의
기본값**이다. 배포는 곧 기존 컨테이너의 종료이므로, 이 처리가 없으면 그 순간
처리 중이던 요청은 매 배포마다 전부 끊긴다. 이번에 확인한 것은 그 손실을
막는 코드가 실제로 동작한다는 사실 하나다.
### 이 실험이 증명하지 못한 것
(기존 3개 항목 유지)

## 후속 조치

- 테스트 엔드포인트는 ENABLE_TEST_ENDPOINTS=true일 때만 등록 (기본 false)
- CI 자동 합격/불합격 판정과 지속 QPS 실험이 필요해지는 시점에 k6로 전환

## 재검토 시점 (Phase 4)
- ALB 연결 draining 시간과 앱 shutdown timeout(8초)의
  대소 관계를 재검토한다. deregistration delay가 8초보다 길면 ALB가 아직
  트래픽을 보내는 동안 앱이 먼저 종료될 수 있다.
