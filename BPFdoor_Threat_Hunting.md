

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
* EDR 기반 이벤트 상관 분석
* 실제 환경 적용 가능한 탐지 로직 개발

---

### 🔹 1-4. Hunting 가설

25년 4월 SKT 사건을 웹쉘 RCE 시나리오를 가설적으로 구성해보겠습니다. 실제와는 다를 수 있고 웹공격이라는 가정하에 작성하겠습니다.

1. 초기 침투 및 시스템 장악
      - 취약점 악용 : 공격자는 Oracle Weblogic, Tomcat 혹은 다른 웹 앱 취약점이 존재(가정)를 사전에 스캔을 통해 확인
      - Web Shell 업로드 : 해당 취약점을 이용하여 curl -F file=@shell.php hxxp://target.com/upload.php 혹은 wget 등의 명령을 통해 시스템 내부에 Web Shell을 은밀하게 업로드하는데 성공했습니다.
      - 영향 : 최초 보안 경계 우회 성공, 공격자가 웹 서버를 통해 원격으로 명령을 실행할 수 있는 **지속적인 통로**를 확보했습니다. 
  
2. 권한 상승 및 백도어 설치 (Privilege Escalation & Backdoor Deployment)
    - Low-Privilege 셸 획득 : 웹서버 계정(tomcat, www-data) 권한으로 시스템 내부에 진입, uname -a , cat /etc/passwd 등으로 환경 탐색 &rightarrow; 추가 권한 상승 루트 파악
    - Root 권한 확보 (Escalation) : SUID 파일, 커널 버그, sudo misconfig 등 악용 &rightarrow; root 권한 상승 &rightarrow; BPFdoor 설치 가능 (커널 소켓 필터 등록에는 높은 권한 필요)

3. BPFdoor 업로드 & 실행
    - **디스크에 기록되지 않고 메모리상에서만 운영**되는 특징이 있는 **/dev/shm** 디렉터리를 이용하여 **/dev/shm/kdmtmpflush** 등 임시 디렉터리에 복사 &rightarrow; 원본 파일 삭제
    - 백도어 프로세스가 **부모 프로세스와 분리되며 메모리에 상주**
    - BPF 필터 setting &rightarrow; **매직 패킷**이 올때만 셸 연결을 열어둠
  
4. HSS DB 접근 및 자료 탈취
    - BPFdoor를 통해 공격자가 언제든지 원할때 재접속 가능함
    - HSS 내부 DB(USIM 인증키, 가입자 식별번호인 IMSI 등)에 대량 쿼리 &rightarrow; 결과 파일 유출
    - 결론적으로 SKT 고객 유심 정보 2300만 대규모 유출 사태 발생

---

## 2. 📊 데이터 소스 정의 (Data Source Mapping)

### 🔹 2-1. 사용한 데이터 소스

| 데이터 소스                     | 용도                | 탐지 포인트                  |
| -------------------------- | ----------------- | ----------------------- |
| **EDR**           | 프로세스, 파일, 네트워크 활동 | 백도어 생성, ps/ls 명령        |
| **Netflow / Firewall Log** | Magic Packet 실마리  | 비정상 포트 및 패킷 길이          |
| **pcap 패킷 캡처**             | Magic Packet 분석   | TCP/UDP/ICMP 기반 Magic 값 |
| **Linux Audit Log**        | 권한 상승, 파일 실행, 원본 파일 복제 및 삭제 등| 실행 파일 수정 여부, BPFdoor 파일 복제 및 삭제 |

---

### 🔹 2-2. BPFdoor 탐지 난이도를 높이는 요인

* 포트를 열지 않음
* 커널 레벨에서 스니핑
* Magic Packet이 일반 트래픽처럼 위장
* 네트워크 시그니처만으로 탐지 어려움

따라서, **프로세스 행동 + 패킷 패턴 + 시스템 로그**의 **다계층 분석**이 필요함.


---

## 3. 🔍 Hunting 절차 및 탐지 시나리오

### 🔹 3-1. 전체 절차 흐름도

1. **기초 정보 수집** (패킷캡처 + EDR 로그 확보)
2. **Magic Packet 성격 파악**
3. **프로세스 활동 기반 필터링**
4. **네트워크 이상 점수화(Abnormal Pattern Scoring)**
5. **탐지 로직 개발 (Splunk/ELK/Python)**
6. **오탐 분석 및 튜닝**

---
