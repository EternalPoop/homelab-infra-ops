# 4단계 모니터링 구축 (Zabbix 7.0 LTS)

## 목표
장애가 나면 관리자가 화면을 보기 전에 알림이 먼저 오는 구조를 만든다.

## 구성

| 호스트 | OS | 역할 | IP | 감시 방식 |
| --- | --- | --- | --- | --- |
| mon01 | Rocky Linux 9 | Zabbix 서버, 웹 화면, DB(MariaDB) | 10.10.10.30 | 에이전트 (자기 자신) |
| lin01 | Rocky Linux 9 | 웹서버 | 10.10.10.20 | Zabbix Agent 2 |
| dc01 | Windows Server 2022 | AD, DNS | 10.10.10.10 | Zabbix Agent 2 (MSI) |

- 웹 화면 접속: 내 PC → `127.0.0.1:8030` → (VirtualBox 포트 포워딩) → mon01 `10.10.10.30:8080`
- 알림: Gmail SMTP로 관리자 메일 발송

## 진행 순서

### 4-0. mon01 생성
- lin01의 `00-clean-install` 스냅샷을 **완전한 복제**로 mon01 생성 (MAC 주소 새로 생성)
- 호스트명, IP 변경

```bash
sudo hostnamectl set-hostname mon01.lab.local
sudo nmcli con mod enp0s3 ipv4.method manual ipv4.addresses 10.10.10.30/24 ipv4.gateway 10.10.10.1 ipv4.dns 10.10.10.10
sudo nmcli con up enp0s3
sudo timedatectl set-timezone Asia/Seoul
```

### 4-1. Zabbix 서버 설치 (mon01)

```bash
# 공식 저장소 등록 (기본 저장소에 Zabbix가 없음, 버전 7.0 고정)
sudo rpm -Uvh https://repo.zabbix.com/zabbix/7.0/rocky/9/x86_64/zabbix-release-latest-7.0.el9.noarch.rpm
sudo dnf clean all

# 서버, 웹 화면, Nginx 설정, DB 초기 스크립트, SELinux 정책, 에이전트 설치
sudo dnf -y install zabbix-server-mysql zabbix-web-mysql zabbix-nginx-conf zabbix-sql-scripts zabbix-selinux-policy zabbix-agent2

# DB 서버 설치 및 실행 (Zabbix 패키지에 포함되지 않음)
sudo dnf -y install mariadb-server
sudo systemctl enable --now mariadb
```

```sql
-- sudo mysql 로 접속 (MariaDB root는 비밀번호 없이 sudo로만 접속)
create database zabbix character set utf8mb4 collate utf8mb4_bin;
create user zabbix@localhost identified by '<DB 비밀번호>';
grant all privileges on zabbix.* to zabbix@localhost;
set global log_bin_trust_function_creators = 1;
quit;
```

```bash
# 초기 테이블·데이터 입력 (기본 관리자 계정 Admin 포함)
zcat /usr/share/zabbix-sql-scripts/mysql/server.sql.gz | mysql --default-character-set=utf8mb4 -uzabbix -p zabbix

# 테이블 수 확인 → 204
mysql -uzabbix -p zabbix -e "show tables;" | wc -l

# 임시 설정 원복
sudo mysql -e "set global log_bin_trust_function_creators = 0;"

# Zabbix 서버에 DB 비밀번호 설정
echo "DBPassword=<DB 비밀번호>" | sudo tee -a /etc/zabbix/zabbix_server.conf

# 웹 화면 포트: /etc/nginx/conf.d/zabbix.conf 에서 listen 8080; server_name mon01; 설정

# 서비스 시작 + 자동 실행
sudo systemctl restart zabbix-server zabbix-agent2 nginx php-fpm
sudo systemctl enable zabbix-server zabbix-agent2 nginx php-fpm

# 방화벽: 8080(웹 화면), 10051(액티브 에이전트 보고 수신)
sudo firewall-cmd --permanent --add-port={8080,10051}/tcp && sudo firewall-cmd --reload
```

- 웹 설치 마법사 완료 → 기본 계정(Admin) 로그인 → **기본 비밀번호 즉시 변경**
- Zabbix server name: `HomeLab-Monitoring`, 시간대: `Asia/Seoul`

### 4-2. 에이전트 설치

**lin01**

```bash
sudo rpm -Uvh https://repo.zabbix.com/zabbix/7.0/rocky/9/x86_64/zabbix-release-latest-7.0.el9.noarch.rpm
sudo dnf clean all
sudo dnf -y install zabbix-agent2

# /etc/zabbix/zabbix_agent2.conf 수정
#   Server=10.10.10.30        (패시브: 이 서버의 질문에만 응답)
#   ServerActive=10.10.10.30  (액티브: 이 서버로 보고)
#   Hostname=lin01            (Zabbix 화면의 Host name과 동일)

sudo firewall-cmd --permanent --add-port=10050/tcp && sudo firewall-cmd --reload
sudo systemctl enable --now zabbix-agent2
```

**dc01**
- Zabbix Agent 2 MSI (Windows, amd64, 7.0 LTS, OpenSSL) 설치
- Host name `dc01`, Server IP `10.10.10.30`, Active checks `10.10.10.30`, Port `10050`
- 방화벽 TCP 10050 허용 규칙 확인

**연결 테스트 (mon01)**

```bash
sudo dnf -y install zabbix-get
zabbix_get -s 10.10.10.20 -k agent.ping   # lin01 → 1
zabbix_get -s 10.10.10.10 -k agent.ping   # dc01  → 1
```

### 4-3. 호스트 등록

| Host name | Visible name | 템플릿 | 인터페이스 |
| --- | --- | --- | --- |
| Zabbix server | mon01 | Linux by Zabbix agent, Zabbix server health | 127.0.0.1:10050 |
| lin01 | lin01 | Linux by Zabbix agent | 10.10.10.20:10050 |
| dc01 | dc01 | Windows by Zabbix agent | 10.10.10.10:10050 |

- 세 호스트 모두 Availability **ZBX(초록)** 확인
- dc01 DNS 서비스(DNS Server, DNS Client) 상태 Running 확인

![호스트 등록 화면](images/04-hosts.png)

### 4-4. 메일 알림

1. Gmail 2단계 인증 → 앱 비밀번호 발급 (계정 비밀번호 대신 사용, 유출 시 해당 비밀번호만 폐기)
2. Alerts → Media types → Email: Email provider `Gmail`, 발신 주소, 앱 비밀번호, Enabled → Test 메일 수신 확인
3. Users → Admin → Media: Email, Warning 이상 등급만 수신
4. Alerts → Actions → Trigger actions: `Report problems to Zabbix administrators` 활성화
5. 테스트: lin01 에이전트 중지 → 장애 메일 / 재시작 → 복구 메일

- [ ] 장애 메일 수신 (걸린 시간: 분)
- [ ] 복구 메일 수신

<img width="849" height="368" alt="image" src="https://github.com/user-attachments/assets/d52b84ff-1643-435c-aa98-8f4a5d5101e6" />


## 배운 점
- **패시브(10050)와 액티브(10051)**: 패시브는 서버가 에이전트에 질문, 액티브는 에이전트가 서버에 보고. 에이전트의 `Server=` 값은 질문을 허용할 서버 목록이라, 허용되지 않은 서버(lin01 자신)에서 질문하자 연결이 거부됨.
- **Host name과 Visible name**: Host name은 에이전트 설정의 `Hostname=`과 일치해야 하는 실제 식별자, Visible name은 화면 표시용 별칭.
- **기본 계정**: 설치 시 공개된 기본 관리자 계정이 함께 생성되므로 설치 직후 비밀번호 변경이 필요.
- **알림은 장애 발생 시점에 판단**: 알림 설정 전에 이미 발생한 장애에는 메일이 가지 않음. 설정 후 장애를 다시 발생시켜 테스트.
