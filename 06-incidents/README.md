# 06. 장애 시나리오 훈련

## 목적

장애 8건을 일부러 만들고 실무 순서(**탐지 → 영향 확인 → 진단 → 조치 → 재발방지**)대로 처리해 기록했습니다.
공고의 "장애 분석 및 조치", "1차 장애조치", "장애 진단" 업무를 직접 해 본 기록입니다.

- 실습일: 2026-10-23
- 대상: lin01(Rocky 9 웹서버), dc01(Windows Server 2022 AD/DNS), cli01(도메인 PC)
- 감시: mon01(Zabbix 7.0) + Gmail 알림
- 시나리오마다 VirtualBox 스냅샷(`06-before-번호`)을 먼저 찍고 진행

## 추가한 감시 항목

기존 Zabbix 템플릿은 디스크·CPU는 보지만 **웹페이지가 실제로 열리는지**는 확인하지 않았습니다. 그래서 mon01이 밖에서 lin01의 80번 포트를 직접 두드려 보는 항목을 추가했습니다.

| 구분 | 설정 |
|---|---|
| Item | Name `HTTP 80 외부 확인` / Type **Simple check** / Key `net.tcp.service[http,,80]` / 1m |
| Trigger | Name `웹 서비스 응답 없음` / Severity **High** / `last(...)=0` |

Type을 Zabbix agent가 아니라 **Simple check**로 한 이유: agent 방식은 lin01이 자기 자신을 검사해서, 서비스는 살아 있는데 방화벽이 막은 장애(시나리오 3)를 못 잡습니다.

## 결과 요약

| # | 장애 | 유발 | 탐지 | 원인 찾은 방법 | 조치 | 복구 (소요) |
|---|---|---|---|---|---|---|
| 1 | lin01 디스크 99% | 11:14:34 | Zabbix 11:19:30 + Gmail | `df -h` → `du -sh` | 원인 파일 삭제 | 11:20:30 (알림 후 1분) |
| 2 | nginx 설정 오타 | 11:36:38 | Zabbix 11:36:43 | `systemctl status` → `journalctl` → `nginx -t` | 15번째 줄 근처 `;` 복구 | 11:39:17 (2분 39초) |
| 3 | 방화벽 http 제거 | 11:43:57 | Zabbix 11:44:43 | 서버 안 정상 / `firewall-cmd --list-all` | http 다시 허용 | 11:45:46 (1분 49초) |
| 4 | sshd 설정 오타 | 11:51:54 | 새 SSH 접속 실패 | `sshd -t` (131번째 줄) | 틀린 줄 삭제 | 11:53:05 (1분 11초) |
| 5 | fstab 오류로 부팅 불가 | 재부팅 | 콘솔에서 부팅 멈춤 확인 | 부팅 메시지 `lv_none` 대기 | 응급 모드에서 fstab 수정 | 12:28:28 |
| 6 | root 비밀번호 분실 | 12:42:51 | `su -` 실패 | 콘솔 접근만 가능 | GRUB `rd.break`로 재설정 | 12:51:15 (8분 24초) |
| 7 | AD 계정 잠김 | 12:03:53 | 사용자 로그인 실패(문의) | 이벤트 4740 → Caller **CLI01** | `Unlock-ADAccount` | 12:08:35 (4분 42초) |
| 8 | DNS 서비스 중지 | 12:09:48 | Zabbix 12:12:13 | cli01 `nslookup` timed out, `Get-Service DNS` | `Start-Service DNS` | 12:14:51 (5분 3초) |

![Zabbix 알림 기록 - 시나리오 1, 2, 3, 8 알림과 복구]
<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/5b48cf07-e313-4f1e-a92a-d25c7d909379" />

---

## 시나리오 1. 디스크 가득 참 (lin01)

```bash
# [lin01] 유발: 남은 용량을 큰 파일로 채움 (6.9G 중 6.8G)
sudo fallocate -l 6.8G /srv/web/dummy.img
df -h /srv/web          # Use% 99%

# [lin01] 진단: 어느 파티션 → 그 안에서 큰 것
df -h
sudo du -sh /srv/web/* | sort -h

# [lin01] 조치
sudo rm /srv/web/dummy.img
df -h /srv/web          # Use% 2%
```

- Zabbix `FS [/srv/web]: Space is critically low (used > 90%)` 알림과 Gmail 메일 수신(Space used 98.6%)
- **알림까지 약 5분**: 트리거가 "5분 동안 계속 90% 초과"일 때만 울리도록 되어 있음. 첫 시도는 1분 24초 만에 지워서 알림이 안 떴음

![디스크 99% → du로 원인 확인 → 삭제 후 2%]
<img width="1294" height="823" alt="image" src="https://github.com/user-attachments/assets/495f0bac-362f-462d-957a-ed3ac04dac4d" />

![Zabbix 디스크 알림 메일]
<img width="794" height="369" alt="image" src="https://github.com/user-attachments/assets/786a1825-640d-45c1-9472-409522b4dd7e" />

**심화: 지웠는데 용량이 안 줄어드는 경우**

```bash
# [lin01] 프로세스가 열고 있는 파일을 지우면 용량이 그대로
sudo fallocate -l 500M /srv/web/dummy.img
sudo bash -c 'sleep 3600 < /srv/web/dummy.img &'
sudo rm /srv/web/dummy.img
df -h /srv/web                 # 9% 그대로

# [lin01] 지워졌지만 아직 열린 파일 찾기 → sleep 이 (deleted) 파일을 잡고 있음
sudo dnf install -y lsof
sudo lsof +L1

# [lin01] 붙잡은 프로세스 종료 → 2%로 돌아옴
sudo pkill -f "sleep 3600"
df -h /srv/web
```

![lsof +L1 로 deleted 파일을 잡은 프로세스 확인]
<img width="1300" height="816" alt="image" src="https://github.com/user-attachments/assets/0d1ceca9-1183-4404-86b3-e6b0e08ede2d" />

## 시나리오 2. 웹 서비스 다운 (lin01)

```bash
# [lin01] 유발: worker_connections 1024; 의 세미콜론 삭제 후 재시작 → 실패
sudo cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.bak
sudo sed -i 's/worker_connections 1024;/worker_connections 1024/' /etc/nginx/nginx.conf
sudo systemctl restart nginx

# [lin01] 진단
systemctl status nginx
sudo journalctl -u nginx -n 20
sudo nginx -t      # unexpected "}" in /etc/nginx/nginx.conf:15

# [lin01] 조치: 줄 수정 → 검사 → 재시작
sudo vim /etc/nginx/nginx.conf
sudo nginx -t      # syntax is ok / test is successful
sudo systemctl restart nginx
curl -s localhost | head -1
```

- 실제로 지운 건 14번째 줄의 `;`인데 에러는 15번째 줄 `}`를 가리킴 → **에러가 가리킨 줄의 바로 위 줄도 확인**
- 진단 화면을 지운 뒤에도 `journalctl -u nginx --since "11:00"`, `/var/log/nginx/error.log`, `history`로 **사후 분석**이 가능했음

![journalctl, error.log, history로 사후 확인]
<img width="869" height="826" alt="image" src="https://github.com/user-attachments/assets/b97054eb-8d63-42fc-bc80-c7b96e24835e" />

## 시나리오 3. 방화벽으로 웹 차단 (lin01)

```bash
# [lin01] 유발
sudo firewall-cmd --permanent --remove-service=http
sudo firewall-cmd --reload

# [lin01] 서버 안에서는 전부 정상
systemctl is-active nginx       # active
sudo ss -tlnp | grep :80        # LISTEN
curl -s localhost | head -1     # <h1>lin01 lab web</h1>
sudo firewall-cmd --list-all    # services 에 http 없음

# [lin01] 조치
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --reload
```

- 서버 안에서는 서비스·포트·페이지 모두 정상인데, **mon01이 밖에서 검사한 `웹 서비스 응답 없음`이 11:44:43에 발생**
- lin01에서 자기 IP(10.10.10.20)로 curl 하면 방화벽을 거치지 않아 성공함 → **감시와 확인은 밖에서 해야 한다**

![서버 안에서는 정상, firewall-cmd 로 http 빠진 것 확인 후 복구]
<img width="1129" height="830" alt="image" src="https://github.com/user-attachments/assets/24c2459e-0bd9-4ebe-b05d-1faaef83a0a4" />

## 시나리오 4. SSH 접속 불가 (lin01)

```bash
# [lin01] 유발: 철자가 틀린 옵션 추가 (Authentica-ton)
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
echo 'PasswordAuthenticaton no' | sudo tee -a /etc/ssh/sshd_config
sudo systemctl restart sshd     # Job for sshd.service failed

# [내 PC] 새 접속 → kex_exchange_identification: Connection aborted

# [lin01] 진단 (이미 열려 있던 SSH 창은 끊기지 않음)
systemctl status sshd --no-pager   # activating (auto-restart)
sudo sshd -t     # line 131: Bad configuration option: PasswordAuthenticaton

# [lin01] 조치
sudo sed -i '/PasswordAuthenticaton/d' /etc/ssh/sshd_config
sudo sshd -t
sudo systemctl restart sshd
```

- 이미 열린 SSH 연결은 sshd가 죽어도 유지됨 → **sshd 설정을 바꿀 땐 기존 세션을 열어 둔 채로 새 창에서 접속 확인**
- 세션이 없었다면 콘솔(실무: iLO/iDRAC 원격 콘솔)로 들어가야 함

![sshd 재시작 실패]
<img width="1121" height="651" alt="image" src="https://github.com/user-attachments/assets/d7a29e7f-7d83-43ae-b3cb-77b253b1a6d0" />

![sshd -t 로 131번째 줄 확인 후 복구]
<img width="929" height="510" alt="image" src="https://github.com/user-attachments/assets/f347bb6c-e0c2-44fa-80db-0acbd942ef64" />

![접속 실패 → 복구 후 접속 성공]
<img width="665" height="190" alt="image" src="https://github.com/user-attachments/assets/738f0dd6-6809-4a51-8520-ba011abdd43f" />

## 시나리오 5. 부팅 불가 - fstab 오류 (lin01)

```bash
# [lin01] 유발: 없는 볼륨을 fstab에 추가하고 재부팅
echo '/dev/vg_data/lv_none /mnt/none xfs defaults 0 0' | sudo tee -a /etc/fstab
sudo reboot
# → 부팅이 "A start job is running for /dev/vg_data/lv_none" 에서 멈춤

# [lin01 응급 모드] 조치
mount -o remount,rw /
vim /etc/fstab            # lv_none 줄 삭제
systemctl daemon-reload
mount -a                  # glusterfs 만 실패 (응급 모드엔 네트워크가 없어서 정상)
reboot
```

- fstab = 부팅할 때 자동으로 연결할 디스크 목록. 한 줄이라도 실패하면 부팅이 응급 모드에서 멈춤
- **2차 장애 발견**: 정상 부팅 후 `df -h`에서 `/mnt/backupvol`이 안 붙어 있었음 → bak01~03이 꺼져 있었음 → 켜고 `sudo mount /mnt/backupvol`로 복구 (12:32)

![부팅이 lv_none 을 기다리며 멈춤]
<img width="736" height="612" alt="image" src="https://github.com/user-attachments/assets/a2717843-40be-4caa-a858-7c611e5b43b1" />

![응급 모드에서 fstab 수정]
<img width="882" height="825" alt="image" src="https://github.com/user-attachments/assets/4ce72588-de4f-4f9c-bad9-9a2316f14e8c" />

![재부팅 후 백업 클러스터 미연결 → mount 로 복구]
<img width="498" height="458" alt="image" src="https://github.com/user-attachments/assets/c26e0a0d-c048-4235-a644-23cb8cfd4cd0" />

## 시나리오 6. root 비밀번호 분실 (lin01)

```bash
# [lin01] 유발: root 비밀번호를 모르는 값으로 변경
sudo passwd root
su -          # su: Authentication failure
```

```text
[lin01 콘솔] 복구
1. sudo reboot → GRUB 메뉴에서 맨 위 커널 선택 → e
2. linux 줄 맨 끝에 rd.break 추가 → Ctrl+X
3. switch_root:/# 에서
   mount -o remount,rw /sysroot
   chroot /sysroot
   passwd root
   touch /.autorelabel     # 빠뜨리면 SELinux 때문에 로그인 안 됨
   exit
   exit
4. su - → whoami → root
```

- 콘솔에 접근할 수 있으면 누구나 root 비밀번호를 바꿀 수 있음 → **IDC 출입 통제, GRUB 비밀번호 같은 물리 보안이 필요한 이유**
- 다음 로그인 때 `Last failed login ... There was 1 failed login attempt` 로 실패 시도가 표시됨

![su 실패]
<img width="482" height="296" alt="image" src="https://github.com/user-attachments/assets/6000664c-57e6-4983-bacc-ba6460f49d54" />

![재설정 후 root 전환 성공]
<img width="582" height="393" alt="image" src="https://github.com/user-attachments/assets/a43de737-7ffd-49f4-b923-9c5f7138bc45" />

## 시나리오 7. AD 계정 잠김 (dc01, cli01)

```powershell
# [dc01] 잠금 정책 확인: 이미 5회 실패 / 15분 잠금으로 설정돼 있었음
Get-ADDefaultDomainPasswordPolicy | Select-Object LockoutThreshold, LockoutDuration

# [cli01] LAB\kim.it 로 틀린 비밀번호 5번 → 계정 잠김

# [dc01] 진단: 잠김 이벤트 4740 → Caller Computer Name: CLI01
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4740} -MaxEvents 3 | Format-List TimeCreated, Message

# [dc01] 조치
Unlock-ADAccount -Identity kim.it
Get-ADUser kim.it -Properties LockedOut | Select-Object Name, LockedOut   # False
```

- 잠김이 반복되면 4740의 Caller Computer에 남아 있는 옛 비밀번호(휴대폰 메일, 네트워크 드라이브, 예약 작업)가 원인. 해제는 본인 확인 후
- dc01 시각이 미국 서부 시간으로 찍혀 있었음(시간대 설정 문제) → 아래 "막힌 점" 참고

![이벤트 4740 - 잠긴 계정과 호출 PC]
<img width="983" height="522" alt="image" src="https://github.com/user-attachments/assets/8599d8c3-5fe6-4660-bbb5-7817100137ef" />

![잠금 해제]
<img width="800" height="279" alt="image" src="https://github.com/user-attachments/assets/0cd80dca-8219-4e0d-a1cc-3e2978cb898e" />

## 시나리오 8. DNS 장애 (dc01)

```powershell
# [dc01] 유발
Stop-Service DNS

# [cli01] 영향 확인
nslookup dc01.lab.local     # DNS request timed out
explorer \\dc01\Ops         # 열림 (DNS 캐시 때문)

# [dc01] 조치
Start-Service DNS
Get-Service DNS             # Running

# [cli01] 확인
ipconfig /flushdns
nslookup dc01.lab.local     # Address: 10.10.10.10
```

- DNS가 죽었는데도 공유 폴더가 열린 이유: `nslookup`은 DNS 서버에 직접 묻지만, 탐색기는 PC에 남은 이름 기록(캐시)을 씀 → **DNS 장애는 캐시가 만료되는 순서대로 서서히 드러남**
- Zabbix가 `Windows: "DNS" (DNS Server) is not running` 알림을 12:12:13에 띄움
- DC가 1대뿐이라 DNS가 단일 장애점 → 재발방지: DC 2대 + 클라이언트에 DNS 2개 등록

![DNS 중지 후 cli01 에서 조회 실패, 공유 폴더는 캐시로 열림]
<img width="1032" height="780" alt="image" src="https://github.com/user-attachments/assets/5a89310d-5c50-4616-824a-99ef4717f2fd" />

![DNS 재시작 후 조회 성공]
<img width="1037" height="778" alt="image" src="https://github.com/user-attachments/assets/dcd4af08-84a1-4e9f-8d15-7f6474be72d7" />

---

## 1차 조치와 에스컬레이션

| 구분 | 내용 |
|---|---|
| 내가 직접 한 1차 조치 | 파일 정리, 설정 원복, 서비스 재시작, 방화벽 규칙 복구, 계정 잠금 해제, 콘솔 복구 |
| 협력사로 넘길 일 | 하드웨어 부품 교체, 원인 불명 반복 장애, 데이터 손상 |
| 넘길 때 전달할 정보 | 발생 시각, 증상, 영향 범위, 이미 해 본 조치, 관련 로그와 캡처 |

## 배운 점

- **바꾸기 전에 검사**: `nginx -t`, `sshd -t`, `mount -a`. 재시작·재부팅 전에 한 번만 쳤으면 2·4·5번은 안 났을 장애
- **감시는 밖에서**: 서버 안에서는 정상이어도 손님은 못 들어오는 장애가 있음(3번)
- **탐지 시간은 트리거 조건에 따라 다름**: 웹은 1분 이내, 디스크는 5분 지속 조건 때문에 약 5분
- **화면을 놓쳐도 로그는 남음**: `journalctl`, `/var/log/nginx/error.log`, `history`, Windows 이벤트 로그
- **재부팅 후에는 마운트 상태까지 확인**: 백업 클러스터가 빠진 걸 `df -h`로 발견
- **서버마다 시간대가 다르면 장애 순서가 꼬임**: dc01만 미국 시간이었음. Zabbix는 자기 서버 시각으로 기록해서 순서를 맞출 수 있었음

## 막힌 점과 해결

| # | 막힌 점 | 원인 | 해결 |
|---|---|---|---|
| 1 | `fallocate: invalid option -- '1'` | 옵션 `-l`(소문자 엘)을 숫자 1로 입력 | `-l`로 다시 입력 |
| 2 | 디스크를 채웠는데 Zabbix 알림이 안 뜸 | 트리거가 "5분 동안 계속 90% 초과" 조건인데 1분 24초 만에 지움 | 다시 채우고 알림이 뜰 때까지(약 5분) 기다림 |
| 3 | `sudo: lsof: command not found` | Rocky minimal 설치에는 lsof가 없음 | `sudo dnf install -y lsof` |
| 4 | 웹 감시 Item을 Zabbix agent 타입으로 만듦 | agent는 서버 안에서 검사해서 방화벽 장애를 못 잡음 | Type을 **Simple check**로 변경 |
| 5 | Latest data에 새 항목이 안 보임 | Name 필터에 예전에 쓴 `DNS`가 남아 있었음 | Name을 `HTTP`로 바꾸고 Apply |
| 6 | nginx 고장 화면을 clear로 지워 버림 | 진단 결과를 캡처하기 전에 화면 정리 | `journalctl -u nginx --since`, `error.log`, `history`로 사후 확인 |
| 7 | 방화벽을 막았는데 curl이 성공함 | lin01에서 자기 IP로 curl → 방화벽을 거치지 않음 | 밖(mon01)의 Zabbix Simple check 기록(11:44:43)으로 차단 확인 |
| 8 | sshd 장애가 안 남 | 오타 대신 올바른 철자 `PasswordAuthentication no`를 입력 → 비밀번호 로그인만 꺼짐 | `.bak`로 원복 후 `PasswordAuthenticaton`(i 빠진 오타)으로 다시 |
| 9 | `systemctl status` 후 키가 안 먹힘 | 결과가 페이지 보기(pager) 화면으로 열림 | `q`로 빠져나오고 이후 `--no-pager` 사용 |
| 10 | `tail /etc/ssh/sshd_config` Permission denied | root만 읽을 수 있는 파일 | `sudo tail` |
| 11 | `Set-ADDefaultDomainPasswordPolicy` 파라미터 에러 | 붙여넣기 중 파라미터 이름이 깨짐 | 정책이 이미 5회/15분으로 설정돼 있어서 Set 불필요 |
| 12 | dc01 이벤트·`Get-Date`가 전날 오후 8시로 나옴 | dc01 시간대가 미국 서부(Pacific) | `Set-TimeZone -Id "Korea Standard Time"` (예정). Zabbix는 서버 기준 한국 시각으로 기록 |
| 13 | DNS 장애 확인 중 `\\dc01\Ops` 권한 오류 | dc01에서 Administrator로 열어서 생긴 권한 문제(DNS와 무관) | 직원 PC인 cli01에서 다시 확인 |
| 14 | DNS를 껐는데 cli01에서 공유 폴더가 열림 | Windows DNS 캐시에 남은 기록 사용 | `ipconfig /flushdns`로 캐시를 비우고 확인 |
| 15 | 이벤트 7036 최근 5개에 DNS가 없음 | 다른 서비스 이벤트에 밀림 | `Where-Object Message -like '*DNS Server*'`로 필터 |
| 16 | Zabbix History에 알림이 안 나옴 | History 버튼만 누르고 Apply 안 함(주소가 `show=1`) | Apply → `show=2`로 바뀌며 전체 기록 표시 |
| 17 | 부팅이 `(no limit)`으로 계속 기다림 | 없는 장치를 시간 제한 없이 기다림 | 응급 모드로 들어가 fstab 수정 |
| 18 | 응급 모드에서 `mount -a` 시 glusterfs 실패 | 응급 모드엔 네트워크가 없음(`_netdev` 마운트) | 정상. 로컬 디스크만 확인하고 재부팅 |
| 19 | 재부팅 후 `/mnt/backupvol`이 안 붙음 | bak01~03 VM이 꺼져 있었음 | bak 3대를 켜고 `sudo mount /mnt/backupvol` |
| 20 | GRUB에서 rescue 커널이 선택돼 있음 | 기본 선택이 세 번째 줄 | ↑로 맨 위 커널 선택 후 `e` |
| 21 | VirtualBox 메뉴에서 재설정을 못 찾음 | 버전마다 메뉴 이름이 다름 | `sudo reboot` 후 GRUB 화면에서 방향키로 멈춤 |
