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

## 배운 점

- **작업은 앞뒤 절차가 절반**: 알람 정지, 스냅샷, 작업 후 마운트·서비스 확인, 알람 재개까지가 한 작업
- **본딩은 뽑아 봐야 안다**: 구성 화면이 정상이어도 실제로 랜선을 뽑고 외부 ping으로 손실을 재야 검증이 됨
- **숫자를 보고 다음 절차를 판단**: iperf3 재전송 수치는 실서버라면 네트워크 점검으로 넘길 기준
- **rsync 2차 복사로 중단 시간 단축**: 서비스 중 1차, 작업 시간에 변경분만
- **장치 이름은 바뀐다**: fstab은 UUID로, 재부팅 전엔 `mount -a`
- **재부팅 후엔 마운트까지 확인**: 백업 클러스터가 빠진 걸 두 번 잡아냄. 의존하는 서버(백업 노드)를 먼저 켜 두는 기동 순서가 중요

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
