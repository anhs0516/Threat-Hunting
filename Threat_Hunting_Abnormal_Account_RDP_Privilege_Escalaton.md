
# Threat Hunting Case: 비정상적인 계정 행위 및 권한 상승 탐색 (RDP 중심)

## 1. Hunting 가설 (Hypothesis)

공격자는 초기 침투 후, 탈취한 계정 정보를 활용하여 **RDP(원격 데스크톱)** 를 통해 네트워크 내 다른 호스트로 **횡적 이동(Lateral Movement)** 을 시도합니다. 이는 정상적인 관리 작업으로 위장하기 쉬우나, **접속하는 호스트의 수**, **시간 패턴**, 그리고 이어진 **권한 상승 행위**를 분석하여 악의적인 행위를 식별할 수 있습니다.

본 케이스는 **RDP**를 이용한 **비정상적인 다중 호스트 접속**과 이어지는 **자격 증명 덤프(Credential Dumping) 시도**를 탐지하여, 공격자의 Lateral Movement 및 권한 상승 시도를 식별하는 것을 목표로 합니다.

**가설:**
**`일반 사용자 계정`** 또는 **`서비스 계정`** 이 **`평소와 다른 시간대`** 에 **`짧은 시간 내 다수의 호스트에 RDP 접속`** 한 후, 접속한 호스트에서 **`시스템 계정 정보를 덤프`** 하려는 시도가 발생한다면, **Lateral Movement 및 권한 상승 공격**으로 간주할 수 있다.

---

## 2. 데이터 소스

| 데이터 소스 | 역할 |
| :--- | :--- |
| **Active Directory/인증 로그** | **RDP 로그온 성공 (Event ID 4624, Logon Type 10)**, 로그온 실패 (Event ID 4625), RDP 세션 연결/끊김 등 **RDP 관련 계정 활동 내역** 확인. |
| **EDR 로그** | **프로세스 실행 정보**, 특히 **LSASS 프로세스**에 대한 **메모리 접근** (Credential Dumping 시도) 추적. |
| **Windows Security Event Log** | **4624 (로그온 성공)** 및 기타 권한 변경 관련 이벤트 수집. |
| **SIEM 연관 분석 엔진** | RDP를 이용한 **다중 호스트 접속** 이벤트와 **메모리 접근** 이벤트를 시간대별로 연관 분석. |



### RDP 로그온 성공 실패 로그 확인


* 이벤트 뷰어 켜기
  
<img width="784" height="639" alt="image" src="https://github.com/user-attachments/assets/e7c672e6-92d8-4231-bddf-40b21e1e0e1d" />

* 로그온 성공 실패 로그
  
  Winodws 로그 - 보안
  
<img width="2083" height="395" alt="image" src="https://github.com/user-attachments/assets/52bceb39-2c69-478b-a47d-44b9f1bd8f37" />


* RDP 로그온 성공 로그

현재 로그 필터링 4624

<img width="1538" height="926" alt="image" src="https://github.com/user-attachments/assets/34021cca-296c-4298-9ab5-67aac9d74057" />

* RDP 로그온 실패 로그

현재 로그 필터링 4625

<img width="1863" height="213" alt="image" src="https://github.com/user-attachments/assets/64f776b8-f544-4550-ba7a-c8376b0f224e" />

* 로그인 시도한 IP 확인 가능

<img width="1378" height="301" alt="image" src="https://github.com/user-attachments/assets/d5ca955e-f620-4b02-a09e-a75f645987d0" />



---

## 3. Hunting 절차 및 탐지 시나리오

#### 1단계: RDP를 이용한 비정상적인 다중 호스트 횡적 이동(Lateral Movement) 탐색

**목적:** 탈취된 계정이 짧은 시간 내에 RDP를 사용하여 다수의 서버에 접속하는 패턴을 탐지합니다.

```Splunk

index=auth_logs event_id="4624" 
| stats dc(dstn_host) as unique_rdp_hosts_count, earliest(_time) as first_access, latest(_time) as last_access by user, src_ip
| eval time_diff_sec = last_access - first_access
| where unique_rdp_hosts_count >= 5 AND time_diff_sec <= 900
| sort -unique_rdp_hosts_count

```

설명 : 900초(15분)이내 동일 계정이 5개 이상의 고유 호스트에 RDP 로그온 성공 이력을 식별
(정상 사용자는 짧은 시간 내 다수 호스트에 RDP 시도하지 않는다는 가정)


#### 2단계 : 자격 증명 덤프 (Credential Dumping) 시도 탐색

**목적:** 1단계에서 식별된 호스트에서 권한 상승을 위한 LSASS 메모리 접근 시도를 탐지합니다.

```

index=sec_edr process_name IN ("procdump.exe", "powershell.exe", "mimikatz.exe", "rclone.exe") OR (target_process_name="lsass.exe" AND access_type="read_memory")
| stats count by user, host, process_name, command_line
| where count > 0

```

설명 : lsass.exe 메모리 접근을 시도하거나 , Credential Dumping에 사용되는 도구(procdump.exe, mimikatz.exe 등)의 실행 이력을 탐지합니다.

