# 07. SE 작업 실습 (상주 IDC SOP 순서)

## 목적

IDC 상주 근무에서 SE분들이 하던 계획 작업(네트워크 검증, 디스크 교체·마이그레이션, RAID, OS·펌웨어 업데이트, 메모리 점검)을 실제 상주 사이트 SOP(표준 작업 절차)와 같은 순서로 홈랩에서 직접 해 봤습니다.

실제 SOP는 어떤 작업이든 아래 순서를 따릅니다. 이 실습도 모든 작업을 이 틀 안에서 진행했습니다.

> **작업 전 점검 → 감시 알람 정지 → 작업 → 네트워크 검증 → 알람 재개 → 이력 기록**

- 실습일: 2026-10-23
- 대상: lin01(Rocky 9), 확인용 mon01(Zabbix 7.0)
- 진행 상황: **A. 본딩·부하 테스트 ✅ / B. 디스크 마이그레이션 ✅** / C. RAID, D. OS 업데이트, E. 펌웨어 절차, F. 메모리는 예정

## 0. 공통 절차

| 단계 | 한 일 |
|---|---|
| 작업 전 | `ss -tn state established`로 접속 중인 연결 확인, 게이트웨이 ping |
| 알람 정지 | Zabbix **Maintenance**(With data collection)로 lin01 알림 일시정지. 실무의 PRTG 일시정지와 같은 작업 |
| 원복 준비 | VirtualBox 스냅샷 |
| 작업 후 | 서비스(nginx, zabbix-agent2), 마운트(`/srv/web`, `/mnt/backupvol`), 웹 응답 확인 → Maintenance 삭제(알람 재개) |

---

## A. 네트워크 본딩 + 부하 테스트

랜카드(NIC) 2개를 하나로 묶어 하나가 끊겨도 통신이 계속되게 하는 **본딩**을 구성하고, 실제 SOP처럼 **부하 테스트와 절체(Failover) 테스트**로 검증했습니다. 실무 SOP에서 OS 업데이트, 펌웨어, NIC 교체, 장애처리가 모두 이 검증으로 끝납니다.

### 구성

| 항목 | 값 |
|---|---|
| 방식 | `active-backup` (하나만 쓰고 나머지는 대기) |
| 랜카드 | enp0s3(primary), enp0s8(추가) — 둘 다 LabNet |
| 옵션 | `miimon=100`(0.1초마다 링크 감시), `primary=enp0s3`, `fail_over_mac=active` |
| IP | bond0 = 10.10.10.20/24 (기존 IP 그대로) |

```bash
# [lin01 콘솔] IP가 enp0s3 → bond0 으로 옮겨 가므로 SSH 가 아니라 콘솔에서 작업
sudo nmcli con add type bond con-name bond0 ifname bond0 \
  bond.options "mode=active-backup,miimon=100,primary=enp0s3,fail_over_mac=active" \
  ipv4.method manual ipv4.addresses 10.10.10.20/24 ipv4.gateway 10.10.10.1 \
  ipv4.dns "10.10.10.10 8.8.8.8" ipv6.method disabled
sudo nmcli con add type ethernet slave-type bond con-name bond0-port1 ifname enp0s3 master bond0
sudo nmcli con add type ethernet slave-type bond con-name bond0-port2 ifname enp0s8 master bond0
sudo nmcli con mod enp0s3 connection.autoconnect no
sudo nmcli con down enp0s3
sudo nmcli con up bond0

# [lin01] 확인
cat /proc/net/bonding/bond0
ip -br addr show bond0
```

![본딩 구성 중 실수: port2를 enp0s3으로 만듦, 자동 생성 프로필 남아 있음]
<img width="1120" height="821" alt="image" src="https://github.com/user-attachments/assets/9bc1c978-e0a6-450b-a6a6-f0a293a56c55" />

![본딩 완료: active-backup, enp0s3·enp0s8 둘 다 up, bond0 10.10.10.20]
<img width="1056" height="840" alt="image" src="https://github.com/user-attachments/assets/38a101c3-6bc8-4fab-aadc-639a1ff2409e" />

### 부하 테스트 (iperf3)

```bash
# [lin01] 받는 쪽
sudo firewall-cmd --add-port=5201/tcp
iperf3 -s

# [mon01] 보내는 쪽 (반대 방향 -R)
iperf3 -c 10.10.10.20 -t 60 -R

# [lin01] 테스트 후 에러 카운터 → 임시 포트 닫기
ip -s link show bond0
sudo firewall-cmd --reload
```

| 항목 | 결과 | 판단 |
|---|---|---|
| 속도 | 1.91 Gbits/sec (60초, 13.4 GBytes) | 정상 |
| errors | RX 0 / TX 0 | 정상 |
| dropped | TX 10 / 약 990만 패킷 | 무시 가능 |
| 재전송(Retr) | 14,760 | 가상 랜카드에 최대 부하를 줄 때 흔한 수준. **실서버였다면 SOP의 네트워크 점검(`ethtool -S`, 케이블·GBIC 교차 테스트) 대상** |

![iperf3 결과: 1.91 Gbits/sec, Retr 14760]
<img width="645" height="810" alt="image" src="https://github.com/user-attachments/assets/5478746f-2248-425f-a322-1ed983d67649" />

![테스트 후 에러 카운터: errors 0, dropped 10]
<img width="971" height="819" alt="image" src="https://github.com/user-attachments/assets/4595183b-e7c2-45ba-926c-1df1c6124ea2" />

### 절체 테스트

VirtualBox에서 어댑터 1의 **케이블 연결을 해제**(= 랜선 뽑기)하고, mon01에서 계속 ping을 보내 손실을 기록했습니다.

```bash
# [mon01] 시각을 붙여 계속 ping, 응답 없으면 "no answer yet" 기록
ping -O 10.10.10.20 | while read line; do echo "$(date +%T) $line"; done | tee ~/bond-ping.log

# [lin01] 절체 확인
sudo cat /proc/net/bonding/bond0 | grep -E "Currently Active|Slave Interface|MII Status"

# [mon01] 손실 집계
grep -c "no answer" ~/bond-ping.log
```

| 순간 | 결과 |
|---|---|
| 랜선(enp0s3) 뽑음 | `enp0s3 NIC Link is Down` → **Currently Active: enp0s8**로 절체 |
| 랜선 다시 꽂음 | `enp0s3 NIC Link is Up` → primary인 **enp0s3로 자동 복귀** |
| 외부 ping (40회) | **1회 손실** (15:44:00, 약 1초) |

![랜선 뽑기 → enp0s8로 절체 → 다시 꽂으면 enp0s3로 복귀]
<img width="1300" height="819" alt="image" src="https://github.com/user-attachments/assets/808dbb40-7e91-45b3-b537-2fec76fc0b98" />

![mon01 ping 40회 중 1회 손실]
<img width="879" height="811" alt="image" src="https://github.com/user-attachments/assets/3367fda7-aaf3-4977-8ccc-11b396676b78" />

---

## B. 디스크 교체·마이그레이션 (rsync 방식)

실무 SOP 방식 그대로 진행했습니다.

> **새 디스크에 rsync 1차 복사 → 작업 시간에 변경분만 2차 복사 → 검증 → fstab 주석 → 전원 끄고 옛 디스크 제거 → 장치 이름 확인 → UUID로 fstab 복구 → mount -a → 쓰기 테스트 → 재부팅 확인**

### 준비: 옛 데이터 디스크

2GB 디스크를 `/data`에 연결하고 운영 데이터처럼 채웠습니다. fstab은 실무에서 흔한 **장치 이름 방식(`/dev/sdc`)**으로 일부러 등록했습니다.

| 원본 기록 | 값 |
|---|---|
| 파일 수 | 549 |
| 용량 | 323M |
| 확인 포인트 | `app` 폴더 소유자 admin1, 숨김 파일 `.hidden-config` |

![원본 기록: 549개, 323M, admin1 소유 폴더, 숨김 파일 / 백업 클러스터 재연결]
<img width="957" height="820" alt="image" src="https://github.com/user-attachments/assets/31d0a367-e1df-4d62-9e65-3f029196a408" />

### 이전

```bash
# [lin01] 새 디스크 포맷 → 임시 경로 연결
sudo mkfs.xfs /dev/sdd
sudo mkdir -p /mnt/newdisk
sudo mount /dev/sdd /mnt/newdisk

# [lin01] 1차 복사 (서비스 중에 미리) - /data/ 끝에 / 를 붙여 숨김 파일까지
sudo rsync -av /data/ /mnt/newdisk/

# [lin01] 2차 복사 (작업 시간, 바뀐 것만)
sudo rsync -av /data/ /mnt/newdisk/

# [lin01] 검증: 파일 수·용량·소유자·내용
sudo find /data -type f | wc -l; sudo find /mnt/newdisk -type f | wc -l
sudo du -sh /data /mnt/newdisk
ls -la /data/app /mnt/newdisk/app
sudo diff -r --no-dereference /data /mnt/newdisk && echo "SAME"
```

| 확인 | 결과 |
|---|---|
| 1차 rsync | 약 336MB 전체 복사 (`speedup is 1.00`) |
| **2차 rsync** | **`app/a.txt` 하나만** 복사 (`speedup is 7,322`) → 서비스 중단 시간 최소화 |
| 파일 수·용량 | 549 = 549, 323M = 323M |
| 소유자 | admin1 그대로 |

![rsync 1차·2차 비교와 검증]
<img width="936" height="816" alt="image" src="https://github.com/user-attachments/assets/fc3680e0-9f5b-4bf1-950a-574b1dd8511a" />

### 교체와 fstab 복구

```bash
# [lin01] fstab 의 "/dev/sdc /data" 줄 주석 처리 → 연결 해제 → 끄기
sudo vim /etc/fstab
sudo umount /data /mnt/newdisk
sudo poweroff
# → VirtualBox 에서 옛 디스크 제거 → 켜기

# [lin01] 옛 디스크가 빠지자 새 디스크 이름이 sdd → sdc 로 바뀜 (lsblk)
# [lin01] vim 에서 옛 줄 삭제(dd) 후 UUID 를 불러와 새 줄 작성
#         :r !blkid -s UUID -o value /dev/sdc
#         → UUID=73dbf605-... /data xfs defaults,nofail 0 0
sudo vim /etc/fstab

# [lin01] 재부팅 전 검사 → 확인 → 쓰기 테스트
sudo systemctl daemon-reload
sudo mount -a
df -h /data
sudo find /data -type f | wc -l
cat /data/app/a.txt
sudo dd if=/dev/zero of=/data/writetest bs=1M count=100 oflag=direct
sudo rm /data/writetest
```

- fstab을 먼저 주석 처리한 이유: 디스크를 빼면 장치 이름이 당겨져서(sdd → sdc) 엉뚱한 디스크를 연결하거나 부팅이 멈출 수 있음
- UUID로 바꾼 이유: 장치 이름과 달리 디스크를 빼고 꽂아도 바뀌지 않음
- `nofail`: 이 디스크가 없어도 부팅은 계속되게 함

![fstab: 장치 이름 대신 UUID + nofail]
<img width="877" height="323" alt="image" src="https://github.com/user-attachments/assets/57753dba-98a8-4894-bd5f-dd7491802608" />

![새 디스크 UUID로 연결, 549개 일치, a.txt 변경분 포함, 쓰기 1.6 GB/s]
<img width="856" height="524" alt="image" src="https://github.com/user-attachments/assets/e9d0a43e-6ba0-48ff-81d6-a0fc77a69980" />

![재부팅 후 /data 자동 연결]
<img width="1010" height="272" alt="image" src="https://github.com/user-attachments/assets/f6ce15fd-580f-406a-b7a7-436f92a686cd" />

---
## C. RAID 구성과 디스크 장애 대응 (mdadm)

실서버의 RAID 컨트롤러(HPE Smart Array, Dell PERC) 대신 리눅스 소프트웨어 RAID(`mdadm`)로 **장애 → Hot Spare 자동 리빌드 → 디스크 교체 → Degraded 수동 교체 → RAID 레벨 변경**을 직접 해 봤습니다. 화면만 다르고 상태 이름(Optimal/Degraded/Rebuild, Hot Spare)과 판단 순서는 실서버와 같습니다.

- 실습일: 2026-10-24
- 디스크: 1G 4개 추가 (sdc, sde, sdf, sdg)

### 작업 전: 디스크 이름 확인 (가장 중요했던 단계)

1G 디스크 4개를 꽂자 장치 이름이 또 밀려서, **B에서 옮긴 `/data` 디스크가 `sdd`**가 됐습니다. 처음엔 문서의 가정(`sdd sde sdf sdg`)대로 `sdd`를 넣었다가 `Device or resource busy`로 막혔고, `lsblk -f`로 보니 운영 데이터 디스크였습니다. 막히지 않았다면 데이터 디스크를 RAID로 덮어썼을 상황입니다.

반대로 `/data`는 이름이 sdc → sdd로 바뀌었는데도 제자리에 붙어 있었습니다. **B에서 fstab을 UUID로 바꿔 둔 덕분**입니다.

```bash
# [lin01] 디스크별 크기·파일시스템·연결 위치 (-f = 파일시스템 정보까지)
lsblk -f
# [lin01] 디스크 이름과 크기만 (-d = 디스크만) → 1G 이고 비어 있는 4개 = sdc sde sdf sdg
lsblk -d -o NAME,SIZE
```

![sdd 를 넣었다가 busy 로 막힘]
<img width="979" height="105" alt="image" src="https://github.com/user-attachments/assets/cecf05cb-aee8-46b8-b7ed-a27c49387ed6" />

![sdd = /data(2G, 사용 중), 빈 1G 디스크 = sdc sde sdf sdg]
<img width="1005" height="754" alt="image" src="https://github.com/user-attachments/assets/077f7186-e42f-4cbc-b19f-66547ef0bb0e" />

### RAID 1 + Hot Spare 구성

| 디스크 | 역할 |
|---|---|
| sdc + sde | 미러 (RAID 1) |
| sdf | 예비 (Hot Spare) |
| sdg | 테스트 3(RAID 5)에서 사용 |

```bash
# [lin01] 소프트웨어 RAID 도구 설치
sudo dnf install -y mdadm
# [lin01] RAID 1 + Hot Spare. 비트맵(write-intent bitmap) 질문은 y = 리빌드 때 바뀐 부분만 다시 맞춤
sudo mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/sdc /dev/sde --spare-devices=1 /dev/sdf
# [lin01] 재부팅해도 같은 이름으로 잡히게 설정 저장
sudo mdadm --detail --scan | sudo tee -a /etc/mdadm.conf
# [lin01] 파일시스템 → 연결 → 테스트 파일과 체크섬(md5) 기록
sudo mkfs.xfs /dev/md0
sudo mkdir -p /mnt/raid && sudo mount /dev/md0 /mnt/raid
sudo dd if=/dev/urandom of=/mnt/raid/test.bin bs=1M count=200
md5sum /mnt/raid/test.bin | tee ~/raid-md5.txt
```

![/dev/md0/ 처럼 끝에 / 를 붙여 생성 실패]
<img width="1005" height="754" alt="image" src="https://github.com/user-attachments/assets/ed1c56bc-eb90-430c-96a2-b2be3e24002b" />

### 테스트 1: 디스크 장애 → Hot Spare 자동 리빌드 → 교체

```bash
# [lin01] sdc 를 고장 처리 (디스크 장애 재현)
sudo mdadm /dev/md0 --fail /dev/sdc
# [lin01] 예비 sdf 가 자동 투입되어 리빌드 → [UU]
watch -n 1 cat /proc/mdstat
# [lin01] 상태와 데이터 확인
sudo mdadm --detail /dev/md0
md5sum /mnt/raid/test.bin; cat ~/raid-md5.txt
# [lin01] 고장 디스크 제거 → RAID 정보 초기화(새 디스크처럼) → 다시 넣기 = 새 예비 디스크
sudo mdadm /dev/md0 --remove /dev/sdc
sudo mdadm --zero-superblock /dev/sdc
sudo mdadm /dev/md0 --add /dev/sdc
```

| 확인 | 결과 |
|---|---|
| `/proc/mdstat` | `sdc[0](F)`, 예비 sdf 투입, `[2/2] [UU]` (1G라 몇 초 만에 리빌드) |
| `--detail` | State `clean`, Failed 1, Spare 0 → 교체 후 Failed 0, Spare 1, sdc = `spare` |
| md5 | 장애 전후 동일 |

![sdc(F) 고장 처리 → 예비 sdf 투입, [UU]]
<img width="1321" height="175" alt="image" src="https://github.com/user-attachments/assets/42a709a4-ed81-4058-ab93-7c95da54a476" />

![sdf 가 0번 자리 active, sdc faulty, md5 동일]
<img width="535" height="619" alt="image" src="https://github.com/user-attachments/assets/646203ad-cb05-4426-956b-fff4d67e8c75" />

![고장 디스크 제거 → 초기화 → 새 예비로 투입 (Spare 1)]
<img width="533" height="643" alt="image" src="https://github.com/user-attachments/assets/b1cae544-d305-42e6-a5b5-ceb5d919ff1a" />

### 테스트 2: 예비 없이 장애 → Degraded → 수동 교체

```bash
# [lin01] 예비 sdc 를 미리 빼고, 동작 중인 sde 를 고장 처리
sudo mdadm /dev/md0 --remove /dev/sdc
sudo mdadm /dev/md0 --fail /dev/sde
# [lin01] [U_] = 한쪽만 남은 Degraded, State : clean, degraded
cat /proc/mdstat
sudo mdadm --detail /dev/md0
# [lin01] 고장 디스크 빼고 새 디스크(sdc) 넣기 → 빈자리가 있어 예비가 아니라 바로 리빌드
sudo mdadm /dev/md0 --remove /dev/sde
sudo mdadm /dev/md0 --add /dev/sdc
```

| 확인 | 결과 |
|---|---|
| 커널 메시지 | `Disk failure on sde, disabling device` / `Operation continuing on 1 devices` (디스크 1개로 서비스 계속) |
| 상태 | `[2/1] [U_]`, `clean, degraded`, 1번 자리 `removed` |
| 교체 후 | `md: recover of RAID array md0` → 약 6초 후 `recover done`, State `clean`, sdf·sdc `active sync` |

테스트 1(예비 있음)은 자동으로 들어가고 넣은 디스크는 **예비**가 되지만, 테스트 2(예비 없음)는 Degraded로 버티다가 넣은 디스크가 **바로 active로 리빌드**됩니다.

![Degraded: Disk failure on sde, [U_], clean, degraded → sdc 투입, 리빌드 시작]
<img width="567" height="824" alt="image" src="https://github.com/user-attachments/assets/10eefba9-2863-4588-936e-cac59ce787d7" />

![리빌드 완료: recover done, clean, sdf·sdc active sync]
<img width="534" height="822" alt="image" src="https://github.com/user-attachments/assets/68b7d043-7d6d-4d79-91cd-ada6ef42f70d" />

![정상 디스크는 fail 처리 없이 remove 되지 않음 (안전장치)]
<img width="562" height="850" alt="image" src="https://github.com/user-attachments/assets/97a768a8-573c-4358-9efa-6ba8be4e4065" />

### 테스트 3: RAID 1 → RAID 5 재구성

RAID 레벨 변경은 대부분 데이터가 지워지는 작업이라 **백업 → 삭제 → 재생성 → 복구 → 검증** 순서로 했습니다.

```bash
# [lin01] 백업 → 연결 해제 → RAID 정지
sudo cp /mnt/raid/test.bin /var/tmp/
sudo umount /mnt/raid
sudo mdadm --stop /dev/md0
# [lin01] 1G 디스크 4개 RAID 정보 초기화 (sdd = /data 는 절대 넣지 않음)
sudo mdadm --zero-superblock /dev/sdc /dev/sde /dev/sdf /dev/sdg
# [lin01] /etc/mdadm.conf 의 옛 ARRAY 줄 삭제 (vim 에서 dd → :wq)
sudo vim /etc/mdadm.conf
# [lin01] RAID 5 (sdc sde sdf) + 예비 sdg
sudo mdadm --create /dev/md0 --level=5 --raid-devices=3 /dev/sdc /dev/sde /dev/sdf --spare-devices=1 /dev/sdg
sudo mdadm --detail --scan | sudo tee -a /etc/mdadm.conf
# [lin01] 파일시스템 → 복구 → 검증
sudo mkfs.xfs -f /dev/md0
sudo mount /dev/md0 /mnt/raid
sudo cp /var/tmp/test.bin /mnt/raid/
md5sum /mnt/raid/test.bin; cat ~/raid-md5.txt
df -h /mnt/raid
```

| 확인 | 결과 |
|---|---|
| mdadm.conf | `ARRAY /dev/md0 metadata=1.2 spares=1 UUID=...` |
| md5 | 원본과 동일 (재구성 후 복구 성공) |
| 용량 | RAID 1 약 1G → RAID 5 **2.0G** (디스크 3개 중 1개분은 패리티) |

![--raid-devices 를 ==raid-devices 로 입력해 생성 실패]
<img width="1052" height="116" alt="image" src="https://github.com/user-attachments/assets/78e4fff0-37b2-4fca-b571-4147514da4d4" />

![RAID 5: mdadm.conf 저장, md5 동일, 2.0G]
<img width="740" height="459" alt="image" src="https://github.com/user-attachments/assets/f1b2de55-e7d2-48ea-b35c-149851112655" />

---

## D. OS 업데이트 (패키지 버전 고정, 옛 커널 정리)

실무 SOP의 OS 업데이트(`yum versionlock`, `package-cleanup --oldkernels`)를 Rocky 9의 `dnf`로 같은 순서로 했습니다. OS 패키지는 iLO/iDRAC가 아니라 OS 안에서 명령으로 하는 작업이라 실무와 같은 방식입니다.

### 사전 기록 → 버전 고정 → 업데이트

```bash
# [lin01] OS·커널 버전, 디스크 여유, 업데이트 대상 수, 실행 중인 서비스 목록
cat /etc/rocky-release
uname -r
df -h /
sudo dnf check-update | wc -l
systemctl list-units --type=service --state=running --no-legend | awk '{print $1}' | sort > ~/svc-before.txt

# [lin01] nginx 를 현재 버전으로 고정 (실무에서는 DB 패키지 등)
sudo dnf install -y python3-dnf-plugin-versionlock
rpm -q nginx
sudo dnf versionlock add nginx
sudo dnf versionlock list

# [lin01] 업데이트 → 고정 유지 확인 → 재부팅 필요 여부
sudo dnf update -y
rpm -q nginx
sudo dnf needs-restarting -r
```

| 확인 | 결과 |
|---|---|
| 사전 기록 | Rocky Linux 9.8, `/` 여유 15G |
| 버전 고정 | `nginx-2:1.20.1-28.el9_8.6.rocky.0.1.*` |
| 업데이트 | `Nothing to do.` (이미 최신) |
| nginx | 업데이트 전과 같은 버전 |
| 재부팅 | `Reboot should not be necessary.` |

이미 최신이라 실제로 받은 패키지는 없었고, "점검 결과 작업 대상 없음"도 결과 보고 대상으로 기록했습니다.

![사전 기록 (check-update 에 te 오타, wc -1 오타 포함)]
<img width="1323" height="825" alt="image" src="https://github.com/user-attachments/assets/b5fe7611-a636-415e-959c-ee489e6c9afe" />

![nginx 버전 고정]
<img width="707" height="287" alt="image" src="https://github.com/user-attachments/assets/13f2c8de-b4ce-4cf4-ae88-823f392c8112" />

![업데이트 대상 없음, nginx 버전 유지, 재부팅 불필요]
<img width="665" height="219" alt="image" src="https://github.com/user-attachments/assets/bdaee2ac-4405-4a7a-bcac-b59e6967938e" />

### 옛 커널 정리 (사고 사례)

```bash
# [lin01] 옛 커널 정리: 최신 포함 2개만 남기려던 명령
sudo dnf remove -y --oldinstallonly --setopt installonly_limit=2 kernel
rpm -q kernel
```

커널이 3개(687.10.1 설치 때 커널, 687.58.1, 687.59.1) 있었는데, 정리 후 **59.1 하나만 남았습니다.** `dnf history`로 보니 실행한 명령에 `kernel`이 아니라 **`kernal` 오타**가 들어가 있었고, 제한(2개 남기기)이 커널에 걸리지 않은 것으로 보입니다. 남은 커널로 정상 부팅되는 것을 확인했지만, 그 전까지는 문제가 생겨도 돌아갈 이전 커널이 없는 상태였습니다.

→ 교훈: **삭제 작업은 `-y` 없이 Removing 목록을 확인하고 진행**, 재부팅 전 **스냅샷**으로 원복 수단 확보, 원인은 `dnf history`로 추적.

![커널 2개가 삭제되고 59.1 하나만 남음]
<img width="1293" height="813" alt="image" src="https://github.com/user-attachments/assets/3585178a-0404-4c14-b083-7d65b6f3872a" />

### 사후 확인

```bash
# [lin01] 버전, 서비스 목록 비교(출력 없으면 동일), 웹, 마운트, 작업 이력
cat /etc/rocky-release
uname -r
systemctl list-units --type=service --state=running --no-legend | awk '{print $1}' | sort > ~/svc-after.txt
diff ~/svc-before.txt ~/svc-after.txt
curl -s localhost | head -1
df -h /srv/web /mnt/backupvol
sudo dnf history | head -5      # 되돌릴 때는 sudo dnf history undo <ID>
# [lin01] 연습 끝, 버전 고정 해제
sudo dnf versionlock delete nginx kernel
```

| 확인 | 결과 |
|---|---|
| 커널 | `5.14.0-687.59.1.el9_8` |
| 서비스 | `diff` 출력 없음 (작업 전후 동일) |
| 웹 | `<h1>lin01 lab web</h1>` |
| 마운트 | `/srv/web`, `/mnt/backupvol` 정상 |
| 이력 | ID 14 versionlock 설치, 15 update, 16 커널 정리 |

![사후 확인: 59.1, 서비스 동일, 웹·마운트 정상, dnf history 로 오타 원인 확인]
<img width="1324" height="568" alt="image" src="https://github.com/user-attachments/assets/3673e76f-fb08-45ee-a001-1a035210ac43" />


## 배운 점

- **작업은 앞뒤 절차가 절반**: 알람 정지, 스냅샷, 작업 후 마운트·서비스 확인, 알람 재개까지가 한 작업
- **본딩은 뽑아 봐야 안다**: 구성 화면이 정상이어도 실제로 랜선을 뽑고 외부 ping으로 손실을 재야 검증이 됨
- **숫자를 보고 다음 절차를 판단**: iperf3 재전송 수치는 실서버라면 네트워크 점검으로 넘길 기준
- **rsync 2차 복사로 중단 시간 단축**: 서비스 중 1차, 작업 시간에 변경분만
- **장치 이름은 바뀐다**: fstab은 UUID로, 재부팅 전엔 `mount -a`
- **재부팅 후엔 마운트까지 확인**: 백업 클러스터가 빠진 걸 두 번 잡아냄. 의존하는 서버(백업 노드)를 먼저 켜 두는 기동 순서가 중요
- **디스크 작업 전엔 `lsblk -f`부터**: 디스크를 추가할 때마다 이름이 밀려서 sdd가 운영 데이터 디스크가 됨. 이름이 아니라 크기·사용처로 대상을 확정
- **예비 디스크 유무에 따라 대응이 다름**: Hot Spare가 있으면 자동 리빌드 후 교체 디스크는 예비로, 없으면 Degraded로 버티다가 교체 디스크가 바로 리빌드
- **RAID 레벨 변경 = 백업부터**: 지우고 다시 만드는 작업이라 md5로 복구 전후 데이터를 검증
- **삭제는 목록을 보고**: `-y`로 커널 정리를 돌렸다가 오타 하나로 커널 2개가 지워짐. `dnf history`로 무엇을 실행했는지 추적 가능

  
## 막힌 점과 해결

| # | 막힌 점 | 원인 | 해결 |
|---|---|---|---|
| 1 | 본딩에 두 번째 랜카드가 안 들어감 | `bond0-port2`를 `ifname enp0s3`으로 잘못 만듦 | 옛 port2 를 UUID로 지정해 삭제(`nmcli con delete <UUID>`) 후 `enp0s8`로 재생성 |
| 2 | `There is another connection with the name 'bond0-port2'` 경고 | 잘못 만든 것을 지우기 전에 같은 이름으로 새로 만듦 | 이름이 겹칠 땐 UUID로 지정해서 삭제 |
| 3 | enp0s8용 `Wired connection 1` 프로필이 남아 있음 | 새 랜카드를 붙이면 자동으로 생성됨 | `nmcli con delete "Wired connection 1"` |
| 4 | 역방향 부하 테스트만 하고 정방향을 빠뜨림 | lin01 RX가 13MB밖에 안 늘어난 것으로 발견 | 테스트 후 카운터로 실제 실행 여부까지 확인 |
| 5 | `ping: invalid option -- '0'`, `'o'` | 옵션 `-O`(대문자 오)를 숫자 0, 소문자 o로 입력 | 대문자 `-O`로 재입력. 콘솔에서는 SSH로 들어가 복사·붙여넣기 권장 |
| 6 | `wc: invalid option -- '1'` | `-l`(소문자 엘)을 숫자 1로 입력 | `wc -l` |
| 7 | 디스크 추가 후 `/mnt/backupvol` 미연결 (2회) | lin01 재부팅 때 bak01~03이 꺼져 있었음 | bak 3대를 켜고 `sudo mount /mnt/backupvol`. 이후 bak 서버를 켜 둔 채로 lin01만 재부팅 |
| 8 | `fstab has been modified ... use 'systemctl daemon-reload'` | fstab 수정 후 systemd 에 다시 읽히지 않음 | `sudo systemctl daemon-reload` |
| 9 | `diff -r` 결과 `No such file or directory` | 복사한 /etc 안의 심볼릭 링크가 원본·사본 양쪽에서 같은 이유로 깨져 있음 | `diff -r --no-dereference`로 링크는 따라가지 않고 비교 |
| 10 | fstab 의 옛 줄을 지우고 UUID 줄을 쓰는 방법을 몰랐음 | UUID를 손으로 치기 어려움 | vim 에서 `dd`로 삭제 후 `:r !blkid -s UUID -o value /dev/sdc`로 UUID 를 불러와 앞뒤만 입력 |
| 11 | 새 디스크가 3G가 아니라 2G로 붙음 | 디스크를 만들 때 크기를 2GB로 지정 | 같은 크기라 용량으로 구분할 수 없어 **파일시스템 UUID**(옛 c1d6de54 / 새 73dbf605)로 새 디스크임을 확인 |
| 12 | 쓰기 테스트 `0 bytes copied` | `if=`를 `of=`로 입력 | `dd if=/dev/zero of=...`로 재실행 |
| 13 | `mdadm: cannot open /dev/sdd: Device or resource busy` | 디스크 4개 추가로 이름이 밀려 sdd = B에서 옮긴 `/data` 디스크 | `lsblk -f`, `lsblk -d -o NAME,SIZE`로 빈 1G 디스크(sdc sde sdf sdg)를 확인해 교체. 기술서 명령도 sdd → sdc로 수정 |
| 14 | `/dev/md0/ is an invalid name for an md device` (2회) | 장치 이름 끝에 `/`를 붙임 | `/dev/md0` |
| 15 | `watch` 화면에서 아무 키도 안 먹음 | `watch`는 1초마다 다시 그리는 동안 입력을 받지 않음 | `Ctrl + C`로 나감 |
| 16 | 콘솔에 `[ 1066.74...] md: recover ...`가 끼어들어 멈춘 것처럼 보임 | 커널 메시지가 VM 콘솔에 바로 출력됨 | Enter 한 번 → 프롬프트 다시 표시. 고장 아님 |
| 17 | 테스트 2 명령을 다시 쳐서 `hot remove failed ... busy`, `No such device` | 테스트가 이미 끝난 상태에서 재실행. sdc는 정상 동작 중, sde는 이미 빠짐 | 상태는 그대로(`[UU]`). 정상 디스크는 `--fail` 없이 못 빼는 안전장치임을 확인 |
| 18 | `mdadm: no raid-devices specified.` | `--raid-devices`를 `==raid-devices`로 입력 | 하이픈 두 개 `--raid-devices=3` |
| 19 | `dnf check-update` 결과가 `1` | 뒤에 `te`가 붙어 "te 패키지"만 조회 + `wc -1`(숫자) 오타 | `sudo dnf check-update \| wc -l` |
| 20 | `package ngnix / kernal is not installed` | 패키지 이름 철자 오타 | `nginx`, `kernel`. "not installed"가 나오면 철자부터 확인 |
| 21 | 옛 커널 정리 후 커널이 2개가 아니라 1개만 남음 (실행 중이던 58.1 포함 삭제) | 정리 명령에 `kernal` 오타 + `-y`로 목록 확인 없이 진행 | `dnf history`로 원인 확인. 59.1로 정상 부팅 확인. 이후 삭제는 `-y` 없이 목록 확인, 재부팅 전 스냅샷 |
| 22 | `head: invalid option -- 'l'` | `head -1`(숫자)을 `-l`(엘)로 입력 | `head -1` (wc는 `-l` 엘, head는 `-1` 숫자) |
