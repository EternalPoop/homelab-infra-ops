# 5단계 백업과 복구

## 목표
백업은 "복구해 봤다"까지가 한 세트다. 백업을 만들고, 일부러 지운 뒤 실제로 살려서 복구 시간을 측정한다.

## 구성

| 대상 | 백업 내용 | 방식 | 저장 위치 | 주기 |
| --- | --- | --- | --- | --- |
| lin01 | /etc, /srv/web, /srv/share | tar 압축 + 복제 클러스터로 복사 | lin01 로컬 /var/backup + GlusterFS 클러스터 bak01~03 (3대 복제) | 매일 02:00 (cron) |
| dc01 | 시스템 상태 (AD DB, 레지스트리, 부팅 파일) | Windows Server Backup (wbadmin) | 별도 디스크 E: (30GB) | 매일 03:00 |
| dc01 | 삭제된 AD 개체 | AD 휴지통 | AD 내부 | 상시 |

- 5-1~5-2는 mon01 한 대에 원격 보관했으나, mon01이 단일 장애점이라는 피드백을 받고 5-5에서 3대 복제 클러스터로 개선함.

## 진행 순서

### 5-1. Linux 백업 (lin01 → mon01)

**[mon01] 백업 받을 곳**

```bash
sudo useradd -m backup                     # 백업 전용 계정 (비밀번호 없음, 키로만 접속)
sudo mkdir -p /backup/lin01
sudo chown backup:backup /backup/lin01
sudo dnf -y install rsync
```

**[lin01] 백업 전용 키 생성**

```bash
sudo dnf -y install rsync tar
sudo ssh-keygen -t ed25519 -f /root/.ssh/id_backup -N ''   # 자동 실행을 위해 암호 없는 키
sudo cat /root/.ssh/id_backup.pub                           # 공개키 복사
```

**[mon01] 공개키 등록**

```bash
sudo mkdir -p /home/backup/.ssh
echo '<lin01 공개키 한 줄>' | sudo tee /home/backup/.ssh/authorized_keys
sudo chown -R backup:backup /home/backup/.ssh
sudo chmod 700 /home/backup/.ssh
sudo chmod 600 /home/backup/.ssh/authorized_keys
sudo restorecon -Rv /home/backup/.ssh
```

**[lin01] 키 접속 확인** (첫 접속 시 yes → cron 실행 시 질문에 막히지 않게)

```bash
sudo ssh -i /root/.ssh/id_backup backup@10.10.10.30 hostname   # → mon01.lab.local
```

**[lin01] 백업 스크립트** `/usr/local/bin/backup.sh`

```bash
#!/bin/bash
DATE=$(date +%F)
SRC="/etc /srv/web /srv/share"
LOCAL=/var/backup
LOG=/var/log/backup.log
mkdir -p $LOCAL
# 압축 실패 시 TAR FAIL 기록 후 종료 (빈 백업을 OK로 기록하지 않음)
if ! tar czf $LOCAL/lin01-$DATE.tar.gz $SRC 2>>$LOG; then
  echo "$DATE TAR FAIL" >> $LOG
  exit 1
fi
# mon01로 원격 복사
rsync -a -e "ssh -i /root/.ssh/id_backup" $LOCAL/ backup@10.10.10.30:/backup/lin01/ \
  && echo "$DATE OK" >> $LOG \
  || echo "$DATE RSYNC FAIL" >> $LOG
# 7일 지난 로컬 백업 삭제
find $LOCAL -name '*.tar.gz' -mtime +7 -delete
```

```bash
sudo chmod 700 /usr/local/bin/backup.sh
sudo /usr/local/bin/backup.sh       # 수동 실행
sudo tail /var/log/backup.log       # OK 확인
ls -lh /var/backup                  # lin01: lin01-2026-10-21.tar.gz 4.7M
# [mon01] ls -lh /backup/lin01      # mon01: 같은 파일 4.7M

# 매일 02:00 자동 실행 (기존 점검 스크립트 줄 유지)
(sudo crontab -l 2>/dev/null; echo "0 2 * * * /usr/local/bin/backup.sh") | sudo crontab -
sudo crontab -l
```

lin01(왼쪽)과 mon01(오른쪽) 양쪽에 같은 백업 파일이 생성됨

![lin01·mon01 백업 파일]
<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/f293561a-653a-41d7-982f-84f5ee32996d" />

### 5-2. Linux 복구 테스트 (lin01)

**테스트 1: 로컬 백업으로 복구**

```bash
sudo rm /srv/web/index.html                                        # 사고 재현
curl http://localhost                                              # 403 Forbidden
sudo tar tzf /var/backup/lin01-2026-10-21.tar.gz | grep index      # 백업 안에 파일 확인
sudo tar xzf /var/backup/lin01-2026-10-21.tar.gz -C / srv/web/index.html
sudo restorecon -Rv /srv/web                                       # SELinux 라벨 복원
curl http://localhost                                              # 정상
```

![로컬 백업 복구]
<img width="966" height="1031" alt="image" src="https://github.com/user-attachments/assets/147e2e7f-1c45-4943-a6d7-194db4753634" />

**테스트 2: 로컬 백업까지 삭제 → mon01 원격 백업으로 복구**

```bash
sudo rm /srv/web/index.html
sudo rm /var/backup/lin01-2026-10-21.tar.gz
sudo rsync -a -e "ssh -i /root/.ssh/id_backup" backup@10.10.10.30:/backup/lin01/lin01-2026-10-21.tar.gz /tmp/
sudo tar xzf /tmp/lin01-2026-10-21.tar.gz -C / srv/web/index.html
sudo restorecon -Rv /srv/web
curl http://localhost
sudo /usr/local/bin/backup.sh && sudo rm /tmp/lin01-2026-10-21.tar.gz   # 로컬 백업 재생성, 임시 파일 정리
```

![원격 백업 복구]
<img width="968" height="1036" alt="image" src="https://github.com/user-attachments/assets/c29349fb-8d71-48d6-aa14-d95d55f6cbae" />

### 5-3. Windows 시스템 상태 백업 (dc01, PowerShell 관리자)

- VirtualBox에서 dc01에 30GB 디스크(`dc01-backup.vdi`, 동적 할당) 추가

```powershell
Get-Disk                                                     # 1번 디스크 30GB RAW 확인
Initialize-Disk -Number 1 -PartitionStyle GPT
New-Partition -DiskNumber 1 -UseMaximumSize -DriveLetter E
Format-Volume -DriveLetter E -FileSystem NTFS -NewFileSystemLabel "Backup" -Confirm:$false

Install-WindowsFeature Windows-Server-Backup
wbadmin start systemstatebackup -backupTarget:E: -quiet      # 시스템 상태(AD 포함) 백업
wbadmin get versions                                         # Can recover: System State 확인
```

![시스템 상태 백업 성공]
<img width="1103" height="650" alt="image" src="https://github.com/user-attachments/assets/4095bee7-6470-44b4-89fd-86dcadf4a15d" />

- 일정 등록: `wbadmin.msc` → Local Backup → Backup Schedule → Custom → System state → Once a day 3:00 AM → Back up to a volume → E:

![백업 일정 등록]
<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/1b096046-4bc4-48ae-a379-ba2fc25e259b" />

### 5-4. 삭제된 AD 계정 복구 (dc01, PowerShell 관리자)

```powershell
Get-ADUser kim.it -Properties MemberOf                       # 삭제 전 그룹 소속 기록
Enable-ADOptionalFeature 'Recycle Bin Feature' -Scope ForestOrConfigurationSet -Target 'lab.local'
Get-ADOptionalFeature -Filter 'Name -like "Recycle*"' | Select-Object Name, EnabledScopes

Remove-ADUser kim.it -Confirm:$false                         # 사고 재현
Get-ADObject -Filter 'samAccountName -eq "kim.it"' -IncludeDeletedObjects | Restore-ADObject
Get-ADUser kim.it -Properties MemberOf, Enabled              # 그룹 소속까지 복구 확인
```

![AD 계정 복구 후 그룹 소속 확인]
<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/d5ba84ca-9a80-4261-948e-502b2d452543" />

### 5-5. 백업 클러스터 (GlusterFS 3대 복제)

5-1의 원격 백업은 mon01 한 대에만 보관해서, mon01이 고장 나면 감시와 백업이 함께 사라지는 **단일 장애점(SPOF)**이었다. 인프라 엔지니어에게 받은 피드백을 반영해, 백업 전용 노드 3대를 묶어 **같은 데이터를 3대에 자동 복제**하는 클러스터로 바꿨다.

```text
lin01 ──백업──▶ /mnt/backupvol (GlusterFS 볼륨 backupvol, replica 3)
                    │  같은 파일을 3대에 동시 저장
          ┌─────────┼─────────┐
        bak01     bak02     bak03
      10.10.10.71  .72       .73
```

| 호스트 | IP | vCPU | 메모리 | 디스크 |
| --- | --- | --- | --- | --- |
| bak01 | 10.10.10.71 | 1 | 1GB | OS 20GB + 브릭 5GB (XFS) |
| bak02 | 10.10.10.72 | 1 | 1GB | OS 20GB + 브릭 5GB (XFS) |
| bak03 | 10.10.10.73 | 1 | 1GB | OS 20GB + 브릭 5GB (XFS) |

- **브릭(brick)**: 각 노드에서 실제 데이터가 저장되는 폴더. 노드마다 별도 디스크에 둠
- dc01을 끈 상태로 실습해서, 이름 찾기는 각 서버의 `/etc/hosts`에 등록 (서버는 DNS보다 `/etc/hosts`를 먼저 봄)

**[bak01] 기본 설정, 브릭 디스크, 설치** (lin01 `00-clean-install` 복제 + 5GB 디스크 추가)

```bash
sudo hostnamectl set-hostname bak01
sudo nmcli con mod enp0s3 ipv4.method manual ipv4.addresses 10.10.10.71/24 ipv4.gateway 10.10.10.1 ipv4.dns 8.8.8.8
sudo nmcli con up enp0s3

sudo tee -a /etc/hosts > /dev/null << 'EOF'
10.10.10.20 lin01
10.10.10.71 bak01
10.10.10.72 bak02
10.10.10.73 bak03
EOF

lsblk                                                   # 5G sdb 확인 후 포맷 (sda는 OS 디스크)
sudo mkfs.xfs /dev/sdb
sudo mkdir -p /data/brick1
echo '/dev/sdb /data/brick1 xfs defaults 0 0' | sudo tee -a /etc/fstab
sudo mount -a && sudo systemctl daemon-reload

sudo dnf -y install centos-release-gluster11            # Rocky 9 extras 저장소의 GlusterFS 11 저장소 등록
sudo dnf -y install glusterfs-server
sudo firewall-cmd --permanent --add-service=glusterfs && sudo firewall-cmd --reload
sudo poweroff                                           # glusterd 시작 전에 종료 후 복제
```

- bak01을 완전한 복제로 bak02, bak03 생성 (새 MAC, 브릭 디스크 포함) → 한 대씩 켜서 호스트명·IP만 변경

**[bak01~03] 클러스터 구성**

```bash
sudo systemctl enable --now glusterd            # 3대 모두
sudo gluster system:: uuid get                  # 3대 모두: 고유번호가 서로 달라야 함

sudo gluster peer probe bak02                   # bak01에서
sudo gluster peer probe bak03
sudo gluster peer status                        # Number of Peers: 2, Peer in Cluster (Connected)

sudo gluster volume create backupvol replica 3 bak01:/data/brick1/backupvol bak02:/data/brick1/backupvol bak03:/data/brick1/backupvol
sudo gluster volume start backupvol
sudo gluster volume info backupvol              # Type: Replicate, Number of Bricks: 1 x 3 = 3
sudo gluster volume status backupvol            # 브릭 3개 Online Y
```

**[lin01] 클러스터 연결**

```bash
sudo tee -a /etc/hosts > /dev/null << 'EOF'
10.10.10.71 bak01
10.10.10.72 bak02
10.10.10.73 bak03
EOF

sudo dnf -y install centos-release-gluster11
sudo dnf -y install glusterfs-fuse
sudo mkdir -p /mnt/backupvol
# _netdev: 네트워크가 켜진 뒤 연결 / backup-volfile-servers: bak01이 죽어도 bak02·bak03으로 연결
echo 'bak01:/backupvol /mnt/backupvol glusterfs defaults,_netdev,backup-volfile-servers=bak02:bak03 0 0' | sudo tee -a /etc/fstab
sudo mount -a && sudo systemctl daemon-reload
df -h /mnt/backupvol                            # 약 5G (3대 복제라 용량은 1대분)
```

**[lin01] 백업 스크립트 변경** `/usr/local/bin/backup.sh`

```bash
#!/bin/bash
DATE=$(date +%F)
SRC="/etc /srv/web /srv/share"
LOCAL=/var/backup
DEST=/mnt/backupvol/lin01
LOG=/var/log/backup.log
mkdir -p $LOCAL
# 압축 실패 시 기록 후 종료
if ! tar czf $LOCAL/lin01-$DATE.tar.gz $SRC 2>>$LOG; then
  echo "$DATE TAR FAIL" >> $LOG
  exit 1
fi
# 클러스터 연결 확인: 끊긴 상태로 복사하면 lin01 자기 디스크에 쌓이고 OK가 찍히므로 먼저 검사
if ! mountpoint -q /mnt/backupvol; then
  echo "$DATE MOUNT FAIL" >> $LOG
  exit 1
fi
# 클러스터로 복사 → 3대에 자동 복제
mkdir -p $DEST
rsync -a $LOCAL/ $DEST/ \
  && echo "$DATE OK" >> $LOG \
  || echo "$DATE COPY FAIL" >> $LOG
find $LOCAL -name '*.tar.gz' -mtime +7 -delete
```

**장애 테스트 1: 노드 1대(bak02) 강제 종료 상태에서 백업·복구**

```bash
# VirtualBox에서 bak02 전원 끄기 후 [lin01]
sudo /usr/local/bin/backup.sh && sudo tail -1 /var/log/backup.log   # 2026-10-22 OK
sudo rm /srv/web/index.html
ls /mnt/backupvol/lin01                                              # 10-21, 10-22 백업 확인
sudo tar xzf /mnt/backupvol/lin01/lin01-2026-10-21.tar.gz -C / srv/web/index.html
sudo restorecon -Rv /srv/web
curl http://localhost                                                # 정상
```

![노드 1대 장애 중 백업·복구]
<img width="843" height="309" alt="image" src="https://github.com/user-attachments/assets/b92cc42c-0568-49b2-8905-017efbdd1bb3" />

**장애 테스트 2: 자가 치유 (bak02 다시 켜기)**

```bash
sudo gluster volume heal backupvol info          # [bak01] 3개 브릭 모두 Connected, Number of entries: 0
ls -lh /data/brick1/backupvol/lin01              # [bak02] 꺼져 있던 동안 만든 10-22 백업이 자동으로 채워짐
```

![자가 치유 확인]
<img width="576" height="308" alt="image" src="https://github.com/user-attachments/assets/e4ca0ee1-6695-422f-8e4d-29f05b1574ac" />
<img width="518" height="318" alt="image" src="https://github.com/user-attachments/assets/45722a59-bd31-4fde-b15c-cbf14a9b7844" />

## 복구 시간 기록

| 실습 | 상황 | 걸린 시간 |
| --- | --- | --- |
| 5-2 테스트 1 | lin01 웹 파일 삭제 → 로컬 백업으로 복구 | 약 4분 18초 |
| 5-2 테스트 2 | 로컬 백업까지 삭제 → mon01 원격 백업으로 복구 | 약 3분 4초 |
| 5-3 | dc01 시스템 상태 백업 (백업 소요) | 약 11분 |
| 5-4 | AD 계정 삭제 → AD 휴지통으로 복구 | 약 3분 1초 |
| 5-5 테스트 1 | 클러스터 노드 1대 강제 종료 상태에서 백업 → 복구 | 약 2분 53초 |

- 테스트 1에는 명령어 실수가 포함됨. 절차를 익힌 테스트 2는 원격 복사 단계가 추가됐는데도 더 빨랐음.

## 배운 점
- **백업 로그의 OK를 믿지 말 것**: tar가 없어 압축이 실패했는데도 rsync가 빈 폴더를 복사하고 OK를 기록함. 단계마다 실패를 검사하고, 파일 크기까지 확인해야 함.
- **로컬 + 원격 두 곳 보관**: 테스트 2처럼 서버 디스크가 통째로 사라져도 원격 백업으로 복구 가능. (3-2-1 원칙의 축소판)
- **백업 전용 계정과 키**: 비밀번호 없는 backup 계정에 공개키로만 접속하게 해서, 자동 실행이 가능하면서도 비밀번호 유출 위험을 없앰.
- **AD 복구는 휴지통이 먼저**: 시스템 상태 복원은 DSRM 재부팅이 필요해 도메인 전체가 멈춤. AD 휴지통은 서비스 중단 없이 그룹 소속까지 복구.
- **복구 후 SELinux 라벨 확인**: 복원한 파일은 라벨이 틀릴 수 있어 restorecon으로 다시 붙임.
- **백업 보관소도 단일 장애점이 될 수 있음**: 원격 백업 서버 한 대는 그 서버가 죽으면 끝. 3대 복제 클러스터로 바꿔 노드 1대가 죽어도 백업·복구가 계속됨을 확인함.
- **복제와 백업은 다름**: 복제는 실수로 지운 파일도 3대에서 동시에 지워짐. 그래서 클러스터 위에 날짜별 백업 파일을 쌓는 구조로 둠.
- **자가 치유**: 꺼졌던 노드가 돌아오면 빠진 데이터를 다른 노드에서 자동으로 채움 (`gluster volume heal ... info`로 확인).
- **왜 3대인가**: 과반(2대)이 살아 있어야 쓰기를 허용해, 노드끼리 데이터가 엇갈리는 split-brain을 막음. 2대로는 과반 판단이 불가능.

## 막힌 점과 해결

| # | 증상 | 원인 | 해결 |
| --- | --- | --- | --- |
| 1 | lin01 `dnf install` 시 `Could not resolve host: mirrors.rockylinux.org` | DNS가 dc01 하나뿐인데 dc01이 외부 이름에 응답하지 못함 (부팅 직후 또는 전달자 불안정) | 보조 DNS 8.8.8.8 추가 (`nmcli con mod +ipv4.dns`). dc01 전달자 점검은 추후 과제 |
| 2 | mon01에서 할 공개키 등록 명령을 lin01에서 실행 | 두 서버 창을 혼동 | lin01에 만든 폴더 삭제 후 mon01에서 재실행. 명령 전 프롬프트의 호스트명 확인 습관 |
| 3 | 백업 키로 접속 시 비밀번호 요구 → `Permission denied` | mon01에 공개키가 등록되지 않음 (자리표시 글자를 그대로 입력). backup 계정은 비밀번호가 없음 | 실제 공개키를 mon01 authorized_keys에 등록, 권한 700/600, restorecon |
| 4 | 첫 접속 시 `Host key verification failed` | 호스트 키 확인 질문에 응답하지 않음 | 수동 접속으로 yes 입력 → known_hosts 등록. cron 자동 실행 전 반드시 필요 |
| 5 | 백업 로그는 OK인데 백업 파일 0개 | Minimal 설치라 tar 미설치. 스크립트가 rsync 성공만 확인 | tar 설치, 압축 실패 시 TAR FAIL 기록 후 종료하도록 스크립트 수정 |
| 6 | `-bash: 날짜: No such file or directory` | 기술서의 `<날짜>`를 그대로 입력. bash에서 `<`는 입력 재지정 기호 | 실제 파일명 입력, Tab 자동완성 사용 |
| 7 | `tar: /srv/web/index.html: Cannot open` | `-C /`와 `srv/web/index.html` 사이 띄어쓰기 누락 → 경로를 풀 위치로 인식 | `-C / srv/web/index.html`로 수정 |
| 8 | dc01 `Get-Disk`에 새 디스크가 없음 | 디스크를 만들기만 하고 VM에 연결하지 않음 (Not Attached) | VirtualBox 저장소에서 기존 디스크 선택 후 연결 |
| 9 | wbadmin `backup location could not be found`, `-quiet` 명령 없음 | `E:` 대신 `E;` 입력. PowerShell에서 `;`는 명령 구분자 | `-backupTarget:E:`로 수정 |
| 10 | `Enable-ADOptionalFeatures` 명령 없음 | 명령 이름 오타 (끝에 s) | `Enable-ADOptionalFeature`로 수정 |
| 11 | dc01 시각이 한국보다 16시간 느림 (오전 5시대) | 시간대가 미국 태평양으로 설정됨 | `Set-TimeZone -Id "Korea Standard Time"` |
| 12 | bak03 SSH 포트 포워딩이 동작하지 않을 뻔함 | 호스트 IP를 `127.0.01`로 입력 (점 누락) | `127.0.0.1`로 수정 후 적용 |
| 13 | `/var/lib/glusterd/glusterd.info: No such file or directory` | 최신 GlusterFS는 고유번호(UUID)를 처음 필요할 때 생성 (정상 동작) | `gluster system:: uuid get`으로 3대의 UUID가 서로 다른지 확인 |
| 14 | 오타 폴더 `/mnt/bakcupvol` 삭제 시 `Permission denied` | `sudo mkdir`로 만들어 주인이 root | `sudo rmdir`로 삭제 (빈 폴더만 지워서 안전) |

막힌 점 5번 증거: 압축 실패(`tar: command not found`) 뒤에도 로그에 OK가 기록됨

![로그는 OK인데 tar 실패]
<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/fa74b434-3cf7-47ac-a130-7e6173863832" />
