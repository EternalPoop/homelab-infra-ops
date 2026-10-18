# homelab-infra-ops
VirtualBox 가상화 기반으로 Linux·Windows 서버 4대를 구축하고,
모니터링·백업과 장애 재현·조치까지 직접 수행하는 인프라 운영 실습 기록

## 구성

| 서버 | OS | 역할 | IP | 상태 |
| --- | --- | --- | --- | --- |
| lin01 | Rocky Linux 9.8 | 웹서버 | 10.10.10.20 | 구축 완료 |
| dc01 | Windows Server 2022 | AD, DNS, 파일서버 | 10.10.10.10 | 구축 완료 |
| cli01 | Windows 11 Enterprise | 직원 PC | 10.10.10.100 | 도메인 가입 완료 |
| mon01 | Rocky Linux 9.8 | 모니터링, 백업 저장소 | 10.10.10.30 | 예정 |

## 진행 현황

| 단계 | 내용 | 상태 |
| --- | --- | --- |
| [1. 가상화](01-virtualization/) | VirtualBox, NAT 네트워크, 포트 포워딩 | 완료 |
| [2. Linux](02-linux/) | 설치, 계정, SSH, 방화벽, SELinux, LVM, Nginx, 점검 자동화 | 완료 |
| [3. Windows](03-windows/) | AD, DNS, GPO, 공유 폴더, 원격 접속, 이벤트 로그 | 완료 |
| [4. 모니터링](04-monitoring/) | Zabbix, 장애 알림 | 예정 |
| [5. 백업](05-backup/) | 백업 자동화, 복구 테스트 | 예정 |
| [6. 장애 대응](06-incidents/) | 장애 재현·조치·보고서 | 예정 |
| [7. 하드웨어](07-hardware/) | 서버 부품, RAID, 메모리 교체 절차 | 예정 |
| [운영 매뉴얼](runbook.md) | 점검표, 자주 하는 작업 | 예정 |

## 참고
학습 과정에서 AI(Claude)를 학습 보조로 사용했고, 모든 설정은 직접 입력하며 진행했습니다.
