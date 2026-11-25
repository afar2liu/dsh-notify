# Changelog — @nolit/dsh-notify

<!-- skeleton:
version: 2.0
kind: report
status: active
updated: 2026-11-13
summary: Engineering history for Notify for Chrome (dsh-notify).
-->

## Unreleased (2026-11-25)
- **기능**: 합성 알림음 추가 — 완료(상승 아르페지오)·승인/질문(세 번 삑) 음색 구분, 음량·프리셋 설정, 시험 재생 버튼. 알림음이 재생되면 Chrome 알림은 `silent`로 표시되어 소리가 한 번만 납니다. 브라우저 자동재생 정책으로 첫 클릭 전에는 시스템 알림음이 대신 재생됩니다.
- **기능**: 계획 검토는 "계획 검토 필요", user-questions로 들어온 승인은 "승인 필요"로 표시합니다.
- **기능**: "이 탭이 백그라운드일 때만 알림"(onlyWhenHidden) 설정 추가 — 모든 이벤트에 적용됩니다.
- **제거**: 峰谷 타이머와 공휴일 조회(네트워크 요청 0건), Windows 프로토타입(prototype-windows/).
- **검증 필요**: Windows에서 `silent`로 인한 이중 알림 여부는 실기기 확인 전입니다.

## 0.0.1-rc.6 (2026-11-13)
- **개선**: 에러 후 완료 알림 이중 발생 이슈 해결 (쿨다운 로직)
- **실측**: 테스트 72개 → 77개 (이중 알림 방지 테스트 5건 추가)
- **지표**: 'Turn failed' 상태 생존 확인

## 0.0.1-rc.5 (2026-11-11)
- **개선**: 크로스 플랫폼(macOS/Linux) 알림 최적화
- **실측**: 72개 테스트 통과

## 0.0.1-rc.4 (2026-11-04)
- **기능**: 네이티브 설정 섹션 및 DSH 아이콘 연동
