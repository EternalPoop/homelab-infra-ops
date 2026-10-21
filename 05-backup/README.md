# 5단계 백업과 복구

## 목표
백업은 "복구해 봤다"까지가 한 세트다. 백업을 만들고, 일부러 지운 뒤 실제로 살려서 복구 시간을 측정한다.

## 구성

| 대상 | 백업 내용 | 방식 | 저장 위치 | 주기 |
| --- | --- | --- | --- | --- |
| lin01 | /etc, /srv/web, /srv/share | tar 압축 + rsync(SSH 키) | lin01 로컬 /var/backup + mon01 원격 /backup/lin01 | 매일 02:00 (cron) |
| dc01 | 시스템 상태 (AD DB, 레지스트리, 부팅 파일) | Windows Server Backup (wbadmin) | 별도 디스크 E: (30GB) | 매일 03:00 |
| dc01 | 삭제된 AD 개체 | AD 휴지통 | AD 내부 | 상시 |

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

![lin01·mon01 백업 파일](images/05-backup-files.png)
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

![로컬 백업 복구](images/05-restore-local.png)
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

![원격 백업 복구](images/05-restore-remote.png)
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

![시스템 상태 백업 성공](images/05-wbadmin-success.png)

- 일정 등록: `wbadmin.msc` → Local Backup → Backup Schedule → Custom → System state → Once a day 3:00 AM → Back up to a volume → E:

![백업 일정 등록](images/05-wbadmin-schedule.png)

### 5-4. 삭제된 AD 계정 복구 (dc01, PowerShell 관리자)

```powershell
Get-ADUser kim.it -Properties MemberOf                       # 삭제 전 그룹 소속 기록
Enable-ADOptionalFeature 'Recycle Bin Feature' -Scope ForestOrConfigurationSet -Target 'lab.local'
Get-ADOptionalFeature -Filter 'Name -like "Recycle*"' | Select-Object Name, EnabledScopes

Remove-ADUser kim.it -Confirm:$false                         # 사고 재현
Get-ADObject -Filter 'samAccountName -eq "kim.it"' -IncludeDeletedObjects | Restore-ADObject
Get-ADUser kim.it -Properties MemberOf, Enabled              # 그룹 소속까지 복구 확인
```

![AD 계정 복구 후 그룹 소속 확인](images/05-ad-recycle-restore.png)

## 복구 시간 기록

| 실습 | 상황 | 걸린 시간 |
| --- | --- | --- |
| 5-2 테스트 1 | lin01 웹 파일 삭제 → 로컬 백업으로 복구 | 약 4분 18초 |
| 5-2 테스트 2 | 로컬 백업까지 삭제 → mon01 원격 백업으로 복구 | 약 3분 4초 |
| 5-3 | dc01 시스템 상태 백업 (백업 소요) | 약 11분 |
| 5-4 | AD 계정 삭제 → AD 휴지통으로 복구 | 약 3분 1초 |

- 테스트 1에는 명령어 실수가 포함됨. 절차를 익힌 테스트 2는 원격 복사 단계가 추가됐는데도 더 빨랐음.

## 배운 점
- **백업 로그의 OK를 믿지 말 것**: tar가 없어 압축이 실패했는데도 rsync가 빈 폴더를 복사하고 OK를 기록함. 단계마다 실패를 검사하고, 파일 크기까지 확인해야 함.
- **로컬 + 원격 두 곳 보관**: 테스트 2처럼 서버 디스크가 통째로 사라져도 원격 백업으로 복구 가능. (3-2-1 원칙의 축소판)
- **백업 전용 계정과 키**: 비밀번호 없는 backup 계정에 공개키로만 접속하게 해서, 자동 실행이 가능하면서도 비밀번호 유출 위험을 없앰.
- **AD 복구는 휴지통이 먼저**: 시스템 상태 복원은 DSRM 재부팅이 필요해 도메인 전체가 멈춤. AD 휴지통은 서비스 중단 없이 그룹 소속까지 복구.
- **복구 후 SELinux 라벨 확인**: 복원한 파일은 라벨이 틀릴 수 있어 restorecon으로 다시 붙임.

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

막힌 점 5번 증거: 압축 실패(`tar: command not found`) 뒤에도 로그에 OK가 기록됨

![로그는 OK인데 tar 실패](images/05-backup-log-fail.png)
<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/fa74b434-3cf7-47ac-a130-7e6173863832" />
