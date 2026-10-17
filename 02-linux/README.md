# 2단계 Linux 서버 (Rocky Linux 9)

대상: lin01 (10.10.10.20)
작업일: 2026-10-17

신입 인프라 담당자가 서버를 받으면 처음 하는 작업(설치, 네트워크, 계정, 접속 보안, 방화벽, 디스크, 웹 서비스, 점검 자동화)을 직접 수행했다.

---

## 2-0. 설치

| 항목 | 선택 | 이유 |
| --- | --- | --- |
| ISO | Rocky-9.8-x86_64-minimal | 실무 서버처럼 GUI 없이 최소 구성 |
| 언어 | English (US) | 에러 메시지를 그대로 검색하기 쉬움 |
| 소프트웨어 | Minimal Install | 필요한 패키지만 직접 설치 |
| 디스크 | 20GB, 자동 파티션 | |
| 호스트명 | lin01 | |
| 네트워크 | 설치 중 비활성 | LabNet은 DHCP가 없어 설치 후 nmcli로 고정 IP 설정 |
| root 계정 | 잠그지 않음, SSH 비밀번호 로그인 불허 | 부팅 장애 시 응급 모드 진입에 root 비밀번호 필요 |
| 관리자 계정 | admin1 (sudo 권한) | root 대신 개인 계정 + sudo로 작업 기록을 남기기 위해 |

설치 직후 스냅샷 `00-clean-install` 생성

---

## 2-1. 기본 설정과 고정 IP

```bash
sudo timedatectl set-timezone Asia/Seoul
nmcli con show
sudo nmcli con mod enp0s3 ipv4.method manual ipv4.addresses 10.10.10.20/24
sudo nmcli con mod enp0s3 ipv4.gateway 10.10.10.1
sudo nmcli con mod enp0s3 ipv4.dns 8.8.8.8
sudo nmcli con up enp0s3
ip addr
ping -c 3 8.8.8.8
sudo dnf -y update
sudo dnf -y install vim bash-completion policycoreutils-python-utils
```

<img width="833" height="575" alt="image" src="https://github.com/user-attachments/assets/3cd9432e-af0b-477f-af7c-b6a4bb8c5998" />

| 명령 | 의미 |
| --- | --- |
| `nmcli con mod` | 네트워크 연결 설정 수정 (manual = 고정 IP) |
| `nmcli con up` | 수정한 설정을 실제로 적용 |
| `dnf update` | 설치 직후 보안 패치 적용 |

---

## 2-2. 계정과 권한

```bash
sudo groupadd ops
sudo useradd -G ops user1
sudo useradd -G ops user2
sudo mkdir /srv/share
sudo chown root:ops /srv/share
sudo chmod 2770 /srv/share
sudo chage -M 90 user1
ls -ld /srv/share
```

- `/srv/share`는 ops 그룹만 읽기·쓰기 가능 (2770)
- 앞자리 2(setgid): 이 폴더에 새로 만든 파일도 자동으로 ops 그룹 소유가 됨
- 비밀번호 만료 90일: 공공기관 보안 점검 항목

---

## 2-3. SSH 보안

목적: 무작위 비밀번호 대입 공격을 차단하고, 누가 작업했는지 기록이 남게 한다.

**1) 내 PC(Windows)에서 키 생성 후 공개키를 서버에 등록**

```powershell
ssh-keygen -t ed25519
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh -p 2220 admin1@127.0.0.1 "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
ssh -p 2220 admin1@127.0.0.1
```

- 개인키(열쇠)는 접속하는 쪽인 내 PC에, 공개키(자물쇠)는 서버에 둔다
- 실습이라 키 암호(passphrase)는 생략했지만 실무에서는 설정한다

**2) root 원격 로그인과 비밀번호 로그인 차단**

```bash
sudo vim /etc/ssh/sshd_config.d/10-hardening.conf
```
```
PermitRootLogin no
PasswordAuthentication no
```
```bash
sudo sshd -t
sudo systemctl reload sshd
```

- 설정 변경 전 `sshd -t`로 문법 검사
- 기존 SSH 창을 닫지 않고 새 창으로 접속을 확인 (설정이 틀리면 원격으로 다시 못 들어오기 때문)

**3) 확인**

```powershell
ssh -p 2220 -o PubkeyAuthentication=no admin1@127.0.0.1
```
→ `Permission denied (publickey...)` : 비밀번호 로그인 차단 확인

(Permission denied 화면 캡처를 끌어다 놓으세요)

---

## 2-4. 방화벽과 SELinux

```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --reload
sudo firewall-cmd --list-all
ss -tulnp
getenforce
```

- 방화벽: 필요한 서비스(ssh, http)만 열고 나머지는 닫음
- SELinux: Enforcing 유지. 끄지 않고 문제가 생기면 라벨을 맞춰 해결 (2-6 참고)

---

## 2-5. 디스크 추가와 LVM

VirtualBox에서 10GB 디스크(SATA 포트 1)를 추가하고 LVM으로 구성했다.

```bash
lsblk
sudo pvcreate /dev/sdb
sudo vgcreate vg_data /dev/sdb
sudo lvcreate -n lv_web -L 5G vg_data
sudo mkfs.xfs /dev/vg_data/lv_web
sudo mkdir /srv/web
echo '/dev/vg_data/lv_web /srv/web xfs defaults 0 0' | sudo tee -a /etc/fstab
sudo mount -a
sudo systemctl daemon-reload
df -h /srv/web
```
```
/dev/mapper/vg_data-lv_web  5.0G   68M  4.9G   2% /srv/web
```

**무중단 용량 확장**

```bash
sudo lvextend -r -L +2G /dev/vg_data/lv_web
df -h /srv/web
```
(확장 후 7.0G 결과 붙여 넣기)

| 용어 | 의미 |
| --- | --- |
| PV | LVM에 쓸 물리 디스크 (/dev/sdb) |
| VG | PV를 모은 저장 공간 풀 (vg_data, 10GB) |
| LV | VG에서 필요한 만큼 떼어 쓰는 볼륨 (lv_web, 5GB → 7GB) |
| fstab | 부팅할 때 자동으로 마운트할 목록 |

---

## 2-6. Nginx 웹서버와 SELinux 문제 해결

```bash
sudo dnf -y install nginx
sudo systemctl enable --now nginx
echo '<h1>lin01 lab web</h1>' | sudo tee /srv/web/index.html
sudo vim /etc/nginx/nginx.conf      # root 를 /srv/web 으로 변경
sudo nginx -t && sudo systemctl reload nginx
curl http://localhost
```

**장애: 403 Forbidden**

권한(chmod)은 문제가 없는데 403이 발생했다.

```
# /var/log/nginx/error.log
"/srv/web/index.html" is forbidden (13: Permission denied)

# sudo ausearch -m avc -ts recent
avc: denied { getattr } for comm="nginx" path="/srv/web/index.html"
  scontext=system_u:system_r:httpd_t:s0 tcontext=...:unlabeled_t:s0

# ls -Zd /srv/web
system_u:object_r:unlabeled_t:s0 /srv/web
```

- 원인: 새로 만든 /srv/web에 웹서버용 SELinux 라벨이 없음 (unlabeled_t)
- 해결: SELinux를 끄지 않고 라벨을 지정

```bash
sudo semanage fcontext -a -t httpd_sys_content_t "/srv/web(/.*)?"
sudo restorecon -Rv /srv/web
curl http://localhost
```
```
<h1>lin01 lab web</h1>
```

내 PC 브라우저에서 `http://127.0.0.1:8020` (포트 포워딩)으로 접속 확인

(브라우저 화면 캡처를 끌어다 놓으세요)

---

## 2-7 ~ 2-8. 일일 점검 스크립트 자동화

매일 08:50에 가동시간, 메모리, 디스크, 주요 서비스 상태, SSH 로그인 실패 수를 로그 파일에 기록한다.
스크립트: [daily_check.sh](daily_check.sh)

```bash
sudo crontab -l
50 8 * * * /usr/local/bin/daily_check.sh
```

(sudo cat /var/log/daily_check.log 결과 붙여 넣기)

---

## 막힌 점과 해결

| 문제 | 원인 | 해결 |
| --- | --- | --- |
| nmcli `invalid <setting>.<property> 'ipv4'` | `ipv4.gateway`를 `ipv4 gateway`로 띄어 씀 | nmcli 속성은 `설정.항목` 형식임을 확인 |
| dnf `Could not resolve host` | IP 통신은 되지만 DNS 응답 실패 | `ping 8.8.8.8`로 범위를 좁힌 뒤 DNS를 8.8.8.8로 임시 변경 (dc01 구축 후 내부 DNS로 전환 예정) |
| 서버 시각이 한국 시간과 13시간 차이 | 설치 시 시간대가 다르게 설정됨 | `timedatectl set-timezone Asia/Seoul`. 로그와 cron 시각이 맞아야 장애 추적 가능 |
| SSH 키를 서버에서 생성함 | 키 생성 위치를 잘못 이해 | 키는 접속하는 쪽(내 PC)에서 만들고 공개키만 서버에 등록. 서버에 만든 키는 삭제 |
| 내 PC에서 SSH, 웹 접속 거부 (connection refused) | NAT 네트워크 포트 포워딩 규칙 없음 | LabNet에 2220→22, 8020→80 규칙 추가 |
| `pvcreate /dev/sdb` No device found, 이후 명령 연쇄 실패 | VM에 디스크를 추가하지 않음 | `lsblk`로 디스크 확인, VM 종료 후 디스크 추가. 첫 에러부터 확인하는 습관 |
| mount 후 `systemctl daemon-reload` 안내 | fstab 변경을 systemd가 다시 읽지 않음 | daemon-reload 실행 |
| 웹 403 Forbidden | SELinux 라벨 없음 | semanage, restorecon으로 라벨 지정 |

## 배운 점

- 웹이 안 열릴 때는 요청이 지나가는 길(포트 포워딩 → 방화벽 → 서비스 → SELinux → 파일)을 따라가며 끊긴 곳을 찾는다
- connection refused는 받는 곳이 없을 때, timed out은 중간에서 막힐 때가 많다
- LVM을 쓰면 서비스를 멈추지 않고 디스크 용량을 늘릴 수 있다
- 설정 변경 전에는 문법 검사(`sshd -t`, `nginx -t`)와 스냅샷을 먼저 한다
