# 2단계 Linux 서버 (Rocky Linux 9)

## 2-0. lin01 설치

작업일: 2026-10-17

| 항목 | 선택 | 이유 |
| --- | --- | --- |
| ISO | Rocky-9.8-x86_64-minimal | 실무 서버처럼 GUI 없이 최소 구성 |
| 언어 | English (US) | 에러 메시지를 그대로 검색하기 쉬움 |
| 시간대 | Asia/Seoul | 로그 시각을 실제 시간과 맞추기 위해 |
| 소프트웨어 | Minimal Install | 필요한 패키지만 직접 설치 |
| 디스크 | 20GB, 자동 파티션 | |
| 호스트명 | lin01 | |
| 네트워크 | 설치 중 비활성 | LabNet은 DHCP가 없어 설치 후 nmcli로 고정 IP 설정 예정 |
| root 계정 | 잠그지 않음, SSH 비밀번호 로그인 불허 | 부팅 장애 시 응급 모드 진입에 root 비밀번호 필요 |
| 관리자 계정 | admin1 (administrator 체크) | root 대신 개인 계정 + sudo로 작업 기록을 남기기 위해 |

설치 후 확인:

```bash
hostnamectl
ip addr
```

<img width="833" height="575" alt="image" src="https://github.com/user-attachments/assets/3cd9432e-af0b-477f-af7c-b6a4bb8c5998" />

설치 직후 스냅샷 `00-clean-install` 생성
