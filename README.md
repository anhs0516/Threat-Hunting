# Threat-Hunting 🔎
SIEM과 EDR 데이터를 기반으로 위협 헌팅 시나리오를 정리한 저장소입니다.  
실제 이벤트 로그를 활용해 IoC 탐지, 상관분석, MITRE ATT&CK 매핑 과정을 기록합니다.  

## 📌 목적
- 금융권 맞춤형 헌팅 시나리오 설계
- IoC 기반 상관분석 및 위협 식별
- 헌팅 결과를 탐지 룰/정책으로 연결

## 🛠️ 기술/도구
- Splunk, ArcSight, EyeCloudSIM
- Genian EDR 등
- MITRE ATT&CK, Sigma

## 📂 주요 내용
- 의심스러운 프로세스 실행 추적 (EDR 기반)
- C2 통신 패턴 탐지 (DNS/HTTP 로그 분석)
- 공격자 활동 TTP → MITRE 매핑
- Splunk SPL 쿼리/헌팅 스크립트 모음

## 📖 방향성
- 단발성 탐지에서 벗어나 지속적인 헌팅 프로세스 정립
- 금융권 특화 위협 모델(피싱, 계정탈취, DDoS) 적용
- 헌팅 → 탐지 룰 → 대응 → 피드백의 순환 구조 구축
