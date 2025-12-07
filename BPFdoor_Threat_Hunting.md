

# 🛡️ BPFdoor Threat Hunting 프로젝트

---


## 1. 📌 Threat Overview & Hunting 가설

### 🔹 1-1. BPFdoor 개요

BPFdoor는 중국계 위협그룹인 **Red Menshen(레드멘션)** 이 주로 사용하는 장기 잠입형 악성코드로 Linux 서버에 설치되어 **포트를 열지 않고**도 특정 **매직 패킷**을 통해 명령을 수신하여 원격제어 하기 위해 설계된 **고도화된 백도어** 악성코드이다.
  
**BPF(Berkeley Packet Filter)** 는 본래 네트워크에서 **필요한 데이터만 걸러내도록 돕는 필터링** 기술인데 이를 악용하여 보안 탐지를 우회하는 것이 특징이며, 최근에는 한국 통신사 외에도 홍콩, 미얀마, 말레이시아, 이집트에서 통신, 금융, 소매 산업과 기업 등 대규모 서버를 겨냥한 APT(Advanced Persistent Threat)공격에 활용됩니다.

### 🔹 1-2. 공격기술 MITRE ATT&CK 매핑

| Tactic (전술) | Technique (기법)       | ID    | 설명                 |
| ----------- | -------------------------- | ----- | ------------------ |
| Execution (실행) | Command and Scripting Interpreter: Bash| T1059.004 | Bash 셸을 통한 명령 실행 |
|  | Command and Scripting Interpreter: PowerShell | T1059.001 | PowerShell로 메모리 내 악성 코드 실행 (Windows 시) |
| Persistence (지속성) | Modify System Binary       | T1543.003 | 시스템 바이너리 변조 또는 서비스 등록  |
| Privilege Escalation (권한 상승)| Sudo and Sudo Caching | T1548.003 | 합법적 권한 상승 도구 악용    |
| Defense Evasion (탐지 회피) | Masquerading |T1036 | 정상 프로세스 이름으로 위장 (예: sshd, syslogd 등)|
|| Obfuscated Files or Information |  T1027 | 난독화/암호화된 스크립트, 실행 파일 사용|


#### 왜 MITRE ATT&CK을 매핑했는지?

MITRE ATT&CK은 전세계 보안 전문가들이 **공격자의 전술(Tactics)과 기법 (Techniques)** 을 체계적으로 정리해놓은 지식 베이스입니다.

BPFdoor가 활동했다는 것은 그전에 **이미 여러 단계(웹쉘 업로드, 권한 상승, 내부 정찰 등)** 의 APT공격이 발생했을 것입니다.

**BPFdoor 같은 백도어**는 공격 후반단계에 진행이 되고, MITRE ATT&CK 관점에서 보면 초기 침투(Initial Access) &rightarrow; 실행(Execution) &rightarrow; 권한 상승(Privilege Escalation) &rightarrow; ... 등 여러 전술을 거치므로, **사전에 여러 이벤트를 탐지할 기회**가 존재했음을 뜻합니다.

따라서 APT 공격 전체가 어떤 전술과 기법을 거쳤는지 표준화된 방식으로 살펴봄으로써 1. **BPFdoor를 설치하기 전에** 이미 발생했을 위협 징후(웹쉘 업로드, RCE 공격, 커널 취약점 악용 등)을 놓치지 않았는지 확인 2. **체계적인 대응 방안 (각 단계별 탐지, 차단)** 을 마련하는 것이 필수적이기에 **SKT 해킹**과 **BPFdoor 시나리오**를 MITRE ATT&CK에 매핑하여 공격자의 구체적 행위를 전술, 기법에 따라 분류하여 **사전 탐지** 기회를 극대화하고 **유사 공격에 대해 방어**해내는 방어 전략을 세울 수 있습니다.


---

### 🔹 1-3. Hunting 목표

* Magic Packet 기반 통신 여부 식별
* BPFdoor 프로세스 및 파일 흔적 탐지
* Network-level 이상 징후 분석
* Sysmon/EDR 기반 이벤트 상관 분석
* 실제 환경 적용 가능한 탐지 로직 개발

---

### 🔹 1-4. Hunting 가설

25년 4월 SKT 사건을 웹쉘 RCE 시나리오를 가설적으로 구성해보겠습니다. 실제와는 다를 수 있고 웹공격이라는 가정하에 작성하겠습니다.

1. 초기 침투 및 시스템 장악
      - 취약점 악용 : 공격자는 Oracle Weblogic, Tomcat 혹은 다른 웹 앱 취약점이 존재(가정)를 사전에 스캔을 통해 확인
      - Web Shell 업로드 : 해당 취약점을 이용하여 curl -F file=@shell.php http://target.com/upload.php 혹은 wget 등의 명령을 통해 시스템 내부에 Web Shell을 은밀하게 업로드하는데 성공했습니다.
      - 영향 : 최초 보안 경계 우회 성공, 공격자가 웹 서버를 통해 원격으로 명령을 실행할 수 있는 **지속적인 통로**를 확보했습니다. 
  
3. 권한 상승 및 백도어 설치 (Privilege Escalation & Backdoor Deployment)
    - Low-Privilege 셸 획득 : 웹서버 계정(tomcat, www-data) 권한으로 시스템 내부에 진입, uname -a , cat /etc/passwd 등으로 환경 탐색 &rightarrow; 추가 권한 상승 루트 파악
    - Root 권한 확보 (Escalation) : SUID 파일, 커널 버그, sudo misconfig 등 악용 &rightarrow; root 권한 상승 &rightarrow; BPFdoor 설치 가능 (커널 소켓 필터 등록에는 높은 권한 필요)

5. BPFdoor 업로드 & 실행
    - 
---

