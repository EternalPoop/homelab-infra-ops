# 3단계 Windows Server (AD, DNS, GPO, 파일서버, 원격 접속)

작업일: 2026-10-18
대상: dc01 (10.10.10.10, Windows Server 2022), cli01 (10.10.10.100, Windows 11 Enterprise)
도메인: lab.local

공공기관·기업 상주 업무에서 가장 많이 하는 계정 관리, 그룹 정책, 공유 폴더 권한, 원격 접속, 이벤트 로그 확인을 직접 수행했다.

---

## 3-1. dc01 설치와 기본 설정

| 항목 | 선택 | 이유 |
| --- | --- | --- |
| ISO | Windows Server 2022 평가판 (영어) | 180일 무료. 에러 메시지 검색이 쉬운 영어판 |
| 에디션 | Standard Evaluation (Desktop Experience) | GUI 포함 버전 |
| 사양 | vCPU 2, 메모리 4GB, 디스크 50GB | |
| 네트워크 | NAT 네트워크 LabNet | |

```powershell
# 어댑터 이름 확인
Get-NetAdapter

# 고정 IP와 게이트웨이
New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 10.10.10.10 -PrefixLength 24 -DefaultGateway 10.10.10.1

# AD 설치 전 임시 DNS
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 8.8.8.8

# 시간대를 한국으로 (AD는 시간이 틀리면 인증 실패)
Set-TimeZone -Id "Korea Standard Time"

# 서버 이름 변경 (AD 설치 전에 해야 함)
Rename-Computer -NewName dc01 -Restart
```

---

## 3-2. AD DS와 DNS 구축

```powershell
# AD DS 기능 설치
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools

# 새 도메인 lab.local 생성, dc01을 도메인 컨트롤러로 승격, DNS 함께 설치
Install-ADDSForest -DomainName "lab.local" -DomainNetbiosName "LAB" -InstallDns
```

재부팅 후 `LAB\Administrator`로 로그인해서 DNS 정리:

```powershell
# dc01 자신을 DNS로
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 10.10.10.10

# 사내에 없는 이름(google.com 등)은 구글 DNS로 넘김 (전달자)
Add-DnsServerForwarder -IPAddress 8.8.8.8

# 사내 서버 이름 등록 (A 레코드)
Add-DnsServerResourceRecordA -ZoneName lab.local -Name lin01 -IPv4Address 10.10.10.20
Add-DnsServerResourceRecordA -ZoneName lab.local -Name mon01 -IPv4Address 10.10.10.30

# 확인
Resolve-DnsName lin01.lab.local
```
```
Name               Type   TTL    Section    IPAddress
lin01.lab.local    A      3600   Answer     10.10.10.20
```

**lin01도 사내 DNS로 전환**

```bash
sudo nmcli con mod enp0s3 ipv4.dns 10.10.10.10
sudo nmcli con up enp0s3
getent hosts dc01.lab.local     # 사내 이름: dc01이 직접 응답
getent hosts google.com         # 외부 이름: dc01이 전달자로 대신 조회
```

---

## 3-3. OU, 그룹, 사용자

```powershell
New-ADOrganizationalUnit -Name "LabUsers" -Path "DC=lab,DC=local"
New-ADGroup -Name "GG_Ops" -GroupScope Global -Path "OU=LabUsers,DC=lab,DC=local"
New-ADUser -Name "kim.it" -SamAccountName kim.it -Path "OU=LabUsers,DC=lab,DC=local" `
  -AccountPassword (Read-Host -AsSecureString "PW") -Enabled $true -ChangePasswordAtLogon $true
Add-ADGroupMember -Identity GG_Ops -Members kim.it

# 확인
Get-ADUser -Filter * -SearchBase "OU=LabUsers,DC=lab,DC=local" | Select Name, Enabled
Get-ADGroupMember GG_Ops | Select Name
```

| 개념 | 의미 |
| --- | --- |
| OU | 관리·정책 적용 단위 (폴더) |
| 보안 그룹 | 권한 부여 단위 (명단) |

---

## 3-4. 그룹 정책 (GPO)

| GPO | 적용 범위 | 설정 |
| --- | --- | --- |
| Default Domain Policy | 도메인 전체 (Computer Configuration) | 계정 잠금 5회 / 15분, 비밀번호 최소 9자·복잡성·90일 |
| GPO_ScreenLock | LabUsers OU (User Configuration) | 화면 보호기 600초, 암호 보호 |

- 계정·비밀번호 정책은 컴퓨터 단위라 도메인 전체에, 화면 잠금은 사용자 단위라 직원 OU에만 적용
- 화면 잠금을 도메인 전체에 걸면 서버 관리자 계정도 10분마다 잠기기 때문

(gpmc.msc 화면 캡처를 끌어다 놓으세요)

---

## 3-5. 공유 폴더와 권한

```powershell
New-Item -Path C:\Share\Ops -ItemType Directory
New-SmbShare -Name Ops -Path C:\Share\Ops -FullAccess "LAB\GG_Ops"
Get-SmbShare
```

- NTFS 권한: 상속 해제 후 Users 제거, GG_Ops에 수정 권한
- 공유 권한(건물 출입증)과 NTFS 권한(방 열쇠)이 다르면 더 엄격한 쪽이 적용됨

---

## 3-6. 클라이언트 도메인 가입 (cli01)

```powershell
New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 10.10.10.100 -PrefixLength 24 -DefaultGateway 10.10.10.1
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 10.10.10.10
Test-NetConnection 10.10.10.10 -Port 53
Resolve-DnsName dc01.lab.local
Add-Computer -DomainName lab.local -NewName cli01 -Restart
```

- `LAB\kim.it` 로그인 → 첫 로그인 시 비밀번호 변경 요구 확인
- `\\dc01\Ops` 접근, `gpresult /r`에서 GPO_ScreenLock 적용 확인

(도메인 로그인 화면, gpresult 결과 캡처를 끌어다 놓으세요)

---

## 3-7. 원격 접속과 이벤트 로그

- dc01 원격 데스크톱 허용 → 내 PC에서 `mstsc` → `127.0.0.1:33890` (포트 포워딩) → `LAB\Administrator`
- 이벤트 뷰어 Security 로그에서 방금 접속한 기록 확인: **4624, Logon Type 10(원격 데스크톱), Administrator**

| 이벤트 ID | 의미 |
| --- | --- |
| 4624 / 4625 | 로그온 성공 / 실패 |
| 4740 | 계정 잠김 (DC) |
| 6008 | 예기치 않은 종료 |

(4624 상세 화면 캡처를 끌어다 놓으세요)

---

## 막힌 점과 해결

| 문제 | 원인 | 해결 |
| --- | --- | --- |
| VirtualBox 클립보드 공유가 동작하지 않음 | 게스트 확장 설치 후에도 미동작 | 필수 명령은 직접 입력하고, RDP를 먼저 활성화해 원격 접속 환경에서 작업 |
| `A positional parameter cannot be found ... 'Rename-Computer'` | 두 명령을 한 줄에 이어서 입력 | 명령 하나에 Enter 한 번. 에러에 찍힌 엉뚱한 단어가 끊어야 할 위치 |
| `Add-DnsServerForwarder` is not recognized | AD DS·DNS 설치 단계를 건너뛰고 설치 후 명령을 먼저 실행 | 설치 → 승격 → 재부팅 → DNS 설정 순서로 다시 진행 |
| `Install-ADDSForest` 승격 검사 실패 | 복구 모드(DSRM) 비밀번호가 복잡성 기준 미달 | 대·소문자·숫자·특수문자 조합으로 재입력 |
| 서버 시계가 미국 시간 | 설치 기본 시간대 | `Set-TimeZone`. AD는 시간 차이 5분 초과 시 Kerberos 인증 실패 |
| `New-ADGroup` server is unwilling to process | 경로 오타 `DC=loacl` | 에러 메시지에 나온 올바른 경로 목록을 보고 수정 |
| 사용자 생성 시 already exists | 두 줄짜리 명령(줄 끝 백틱 `` ` ``)의 첫 줄만 실행 → 비밀번호 없는 비활성 계정이 먼저 생성됨 | 지우지 않고 `Set-ADAccountPassword`, `Enable-ADAccount`로 보완 |
| lin01 `nmcli` Error checking authorization | sudo 누락 (Minimal 설치라 권한 확인 프로그램 없음) | sudo 붙여 재실행 |
| VM이 느려지고 키 입력 실패 (Keyboard failure) | PC 자원 부족 (VM 동시 실행) | 필요 없는 VM 종료 후 작업 |
| cli01 설치 직후 Windows License is expired | 평가판 ISO 자체의 기한 경과 | 1시간마다 꺼지는 제약을 감안해 짧은 작업 단위로 진행 |
| cli01 `Access is denied` | 일반 권한 PowerShell | 관리자 권한(Terminal Admin)으로 실행 |
| cli01 → dc01 DNS 시간 초과 / Destination Host Unreachable | (실제 원인으로 수정) | `ipconfig`, `Test-NetConnection -Port 53`으로 경로 확인 |
| cli01 시계가 16시간 앞섬 | VM 시계 설정 | `Set-Date`로 보정. 도메인 가입 전 시간 차이 해소 필요 |
| RDP 접속 실패 | 포트 포워딩 호스트 포트를 33890이 아닌 3389로 입력 | dc01 쪽 정상 확인 → 내 PC `Test-NetConnection`, `netstat`으로 33890 미개방 확인 → 예약 포트 확인 → 규칙 재확인 후 오타 수정 |
| RDP 로그인 실패 | 내 PC의 Microsoft 계정이 자동 입력됨 | 다른 계정 사용 → `LAB\Administrator` |

## 배운 점

- AD는 DNS로 도메인 컨트롤러를 찾기 때문에 DNS가 죽으면 로그인·가입·GPO가 모두 실패한다
- AD는 서버와 PC의 시간 차이가 5분을 넘으면 인증이 실패한다
- 네트워크 에러는 종류로 원인을 좁힌다: Unreachable(같은 망에서 상대를 못 찾음), TimedOut(방화벽·무응답), Refused(받는 프로그램 없음)
- 원격 접속 장애는 서버 쪽(서비스·방화벽)과 내 쪽(포트 포워딩·열린 포트)을 나눠서 확인한다
- OU는 정책 적용 단위, 그룹은 권한 부여 단위
