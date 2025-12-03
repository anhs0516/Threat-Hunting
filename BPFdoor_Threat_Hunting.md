

# 🛡️ BPFdoor Threat Hunting 프로젝트

---


## 1. 📌 Threat Overview & Hunting 가설

### 🔹 1-1. BPFdoor 개요

BPFdoor는 중국계 위협그룹인 Red Menshen(레드멘션)이 주로 사용하는 장기 잠입형 악성코드로 Linux 서버에 설치되어 포트를 열지 않고도 특정 매직 패킷을 통해 명령을 수신하여 원격제어 하기 위해 설계된 고도화된 백도어 악성코드이다.
  
BPF(Berkeley Packet Filter)는 본래 네트워크에서 필요한 데이터만 걸러내도록 돕는 필터링 기술인데 이를 악용하여 보안 탐지를 우회하는 것이 특징이며, 최근에는 한국 통신사 외에도 홍콩, 미얀마, 말레이시아, 이집트에서 통신, 금융, 소매 산업과 기업 등 대규모 서버를 겨냥한 APT(Advanced Persistent Threat)공격에 활용됩니다.
