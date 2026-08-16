# Experiments — 파괴 검증·성능 측정 기록

만든 것을 일부러 깨뜨리거나 부하를 걸어 검증한 결과를 기록한다.
"동작한다"는 주장은 실행 결과로만 인정한다 (COLLABORATION.md).

## 규칙

- **개선 작업은 before 수치를 먼저 측정하고 시작한다** — after만 있으면 성과 주장이 안 된다
- 수치와 스크린샷 필수. 스크린샷은 `../images/`에 두고 링크
- 파일명: `NNN-짧은-제목.md` (예: `001-image-size-optimization.md`)
- 예시: 이미지 크기 before/after, 빌드 시간, 컨테이너 kill 후 복구 시간,
  부하 테스트 p95 latency, 롤백 소요 시간, DB 복구(RTO/RPO) 측정

## 목록

| # | 실험 | Phase | 핵심 수치 (before → after) |
|---|---|---|---|
| [001](./001-image-size.md) | 이미지 크기 (빌드 플래그) | 1 | 27.5MB → 23.4MB (-15%) |
| [002](./002-graceful-shutdown.md) | graceful shutdown in-flight 보존 | 1 | 유실률 100% (SIGKILL) → 0% (SIGTERM), 5회 반복 |
