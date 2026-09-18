# NHN Cloud CAP 시험 직전 정리

시험 30분 전에 한 번 훑는 용도. 헷갈리기 쉬운 것과 숫자 위주로만 담았습니다.
설명이 필요한 개념은 뺐고, "바꿔치기로 나오면 틀리는 것"만 남겼습니다.

---

## 0. 문제 풀 때 먼저 확인할 것

- 조건에 **숫자**(RTO/RPO, 용량, 기간, 대역폭)가 있으면 그 숫자부터 서비스 한계와 대조한다.
- 조건에 **비용 제한**이 있으면 Active-Active·상시 인스턴스·전용 회선 보기는 대부분 오답이다.
- 조건에 **규제**(금융/공공)가 있으면 물리적 분리, 국내 저장, 최종 책임 주체를 먼저 본다.
- "가능한가"가 아니라 "**조건에 맞는가**"를 묻는다. 기술적으로 되지만 조건 하나를 어기는 보기가 대표 오답.
- 복수 선택형은 보기를 **하나씩 독립적으로** 판정한다. 요구사항과 정답이 1:1로 대응되지 않을 수 있다.

---

## 1. 네트워크 — 바꿔치기 단골

### VPC 연결 4종

| 방식 | CIDR 중복 | 리전 간 | 비용 |
|---|---|---|---|
| Transit Hub | **불가** | **불가**(리전 피어링 필요) | Attachment + 트래픽 |
| Peering Gateway | **불가** | **가능**(Region Peering) | 게이트웨이 무료, 트래픽만 과금 |
| Internet Gateway(FIP/NAT) | **가능** | 가능 | FIP·트래픽 과금 |
| 다중 NIC 인스턴스 | **가능** | — | 중계 서버에 NAT/Proxy 설정 필요 |
| 사용자 정의 엔드포인트 | ⭐ **가능** | — | 대상은 **Load Balancer만** |

- **CIDR이 겹친다 → Transit Hub·Peering·VPN 전부 탈락.** 사용자 정의 엔드포인트 / IGW / 다중 NIC.
- 전이적 피어링 미지원. A↔B, B↔C 있어도 A↔C는 별도 피어링.
- 피어링 연결 수 = **N(N−1)/2** (5개면 10개) → 규모 커지면 Transit Hub.
- Peering 3유형: VPC(동일 프로젝트) / **Project(동일 리전, 다른 프로젝트)** / **Region(다른 리전)**. 뒤 둘은 **테넌트 ID + VPC ID 등록** 필요.

### Transit Hub

- 공유받은 프로젝트의 Attachment는 **기본 라우팅 테이블 자동 연결·전파 안 됨** → 소유 프로젝트에서 **명시적 연결**.
- 공유받은 프로젝트에는 **라우팅 테이블을 만들 수 없다**(Attachment만).
- 리전 간 공유 불가, 허브 간 피어링 불가 → **리전 피어링 + 도착지 라우팅 리소스**로 전이적 통신.
- **멀티캐스트 옵션은 생성 시에만** 선택(사후 변경 불가, 삭제 후 재생성).
- Spoke to Hub / Hub to Spoke 라우팅 테이블 **분리 안 하면 허브 내부 루프**.
- NF_TRAFFIC_VIP는 Transit Hub에 **직접 연결 불가** → Forward(경유) 서브넷 필요.

### 라우팅

- **Longest Prefix Matching**: `/16 local`이 있으면 `/24` 경로를 추가해야 내부 트래픽이 방화벽 경유.
- 서브넷 정적 라우트는 **DHCP로 최초 부팅 시 적용** → 변경 시 재부팅/DHCP 재시작. 게이트웨이는 **IP로만** 지정. 멀티캐스트 등록 불가.
- 서브넷 : 라우팅 테이블 = **1 : 1** (반대는 1:N 가능).
- 라우팅 테이블 지원 게이트웨이 = **Peering Gateway, Transit Hub 둘뿐**. local 게이트웨이는 규칙 없음, 삭제 불가.

### 하이브리드 연결 3종

| | 대역폭 | 인터넷 경유 | 용도 |
|---|---|---|---|
| Direct Connect | **10Mbps ~ 10Gbps** | 미경유 | 전용(통신사, 물리 1:1) / **호스팅(파트너망, 논리, 멀티클라우드·빠른 구축)** |
| VPN Gateway | **20M / 50M / 100M / 1G** | 경유 | Site-to-Site |
| Network Firewall IPSec VPN | — | 경유 | VPN 종단 + 보안정책 + 로깅 동시 |
| Cloud Access | — | 경유 | ⭐ **Client-to-Site, WireGuard**, 재택·원격 사용자 |

- VPN Gateway: **VPC당 1개**, **연결 최대 10개**, **IPv6 미지원**, 대역폭은 VPC 내 모든 연결에 공통.
- VPN 연결된 VPC는 Peering/Transit Hub 동시 연결 불가. **단, Transit Hub 프로젝트 공유로 VPN 공유는 가능**.
- Network Firewall 서브넷: **NAT·로그 서브넷은 IGW 필수 / 트래픽 서브넷은 IGW 불필요**. **로그 서브넷에는 방화벽 정책 미적용**.
- Network Firewall은 **Stateful**(응답 자동 허용). 로그 전송 대상: LNCS, Object Storage, Syslog.

### DNS 4종

| | 관리 | 범위 |
|---|---|---|
| 내부 DNS | NHN Cloud | 인스턴스 기본·서비스 엔드포인트 |
| Private IP DNS | NHN Cloud | VPC 내부 자동 이름, 멀티 VPC 제한적 |
| **Private DNS** | **고객** | 내부 전용, Zone에 **여러 VPC 바인딩**(동일 프로젝트·리전) |
| **DNS Plus** | **고객** | **외부 인터넷**, Anycast, GSLB, ⚠️ **도메인 구입 불가** |

- 서비스 엔드포인트 도메인은 **외부 퍼블릭 DNS에서도 사설 IP가 조회된다**.

### 보안

- **Network ACL = Stateless / 서브넷** vs **Security Group = Stateful / 인터페이스**.
- 보안 그룹 적용 서비스: **Instance, NKS, NCS, Storage Gateway**. RDS·NAS·LB·Bastion은 **자체 ACL**.
- **가상 IP(VIP)**: 같은 서브넷 필수, **OS에서 직접 설정**, **보안 그룹 없음** → 상대 인스턴스 인바운드에 **VIP 주소를 직접 명시**. 권장 방식은 스푸핑 방지 유지 + 추가 허용 주소 등록.
- SSL: **LB는 Passthrough / Offloading만**. **Bridging은 Web Firewall**. (Offloading=성능, Passthrough=보안, Bridging=절충)
- 패킷 ACCEPT/REJECT 확인 = **Flow Log**(헤더만, payload 없음, Object Storage 저장).

---

## 2. IAM / 리소스 계층

- 계층: **조직 → 프로젝트 → 리소스**. **청구 = 조직**, **비용·사용량 추적 = 프로젝트**.
- 공통 보안 정책(MFA, 비밀번호, 태그 표준) = **조직 레벨**.
- ⭐ **조직에서는 계정에 프로젝트 역할을 부여할 수 없고 상속도 없다.** 바인딩은 프로젝트마다.
- 프로젝트 공통 역할 그룹은 **조직에서만 수정**, 프로젝트는 참조·사용만.
- ⭐ **최종 권한 = (허용 ∩ 조건) − 거부**. 평가 순서 **거부 → 허용 → 조건**. 설계 순서 **Guardrail(Deny) → Allow → Condition**.
- 조건은 **역할·역할 그룹 모두**에 부여 가능. 조건 여러 개면 **교집합**. 상위 역할 조건은 **하위가 상속**.
- CI/CD·자동화는 **IAM 시스템 계정**(루트 계정 금지). 결제 관리는 NHN Cloud 계정.
- **DR 프로젝트는 기본 VIEWER(읽기 전용)**, 장애 시에만 역할 변경.
- 환경 혼재는 안티패턴 → **환경별 프로젝트 분리 + 공통 인프라 프로젝트**.

---

## 3. Compute & Container

- **CA 증설 기준 = Pending Pod** (노드 CPU 사용률 아님). 설정 단위 = **노드 그룹**.
- 증설은 현재 < 최대, 감축은 현재 > 최소 + 임계 유지 시간.
- **HPA = Pod 개수**(재생성 아님, **DaemonSet 제외**) / **VPA = Pod 크기**(개수 안 바꿈).
- HPA+VPA 병행 가능하나 **같은 CPU·메모리 지표 동시 제어 금지**. VPA 모드: Off / **Initial(신규 Pod만)** / InPlaceOrRecreate.
- **IGW, 서비스 게이트웨이, 라우팅은 클러스터 생성 후 변경 불가.** IGW 없으면 NCR·OBS 서비스 게이트웨이 필수.
- 보안 그룹은 **노드 그룹 단위 → Pod 간 제어 불가**. Pod 간은 **NetworkPolicy**(기본값 전부 허용).
- 강한 격리(금융·공공)는 **클러스터 분리**(CP 2개 = 비용↑).
- **인그레스 컨트롤러는 기본 제공 안 함**(직접 설치). **replica 최소 3 + Anti-Affinity**.
- **Ingress = Feature Frozen / Gateway API = GA**. Gateway(인프라: LB IP·리스너·TLS) vs **HTTPRoute(개발: 경로·가중치·카나리)**.
- L7 컨트롤러 도입 기준 = 서비스 **3~5개 이상**. mTLS·분산 추적이면 서비스 메시.
- **NCS**: 노드 없음 → **사이드카만**(DaemonSet 불가). 내부 LB는 **동일 서브넷만, FIP 불가, 무료**. 한 작업에 여러 컨테이너 = 일부만 교체 불가·함께 스케일.
- NKS 부적합: K8s 경험 없음·CP 비용 절감 / NCS 부적합: 멀티 클라우드·K8s API 호환·복잡한 네트워크 정책.
- **Cloud Functions**: 메모리 128~1024MB, 실행 **최대 300초**, 동시 100. **Pool Manager=비용 / New Deployment=성능(Cold Start 없음)**.

---

## 4. Storage

| 수치 | 대상 |
|---|---|
| 1GB ~ 2,000GB | Block Storage (2TB 이상은 **LVM**) |
| **300GB ~ 10,000GB** | NAS (300GB 미만 불가, **축소도 가능**) |
| **30일** | Economy **최소 보관 기간** |
| 1 ~ 36,500일 | 수명 주기 / 객체 잠금 기간 |
| **5GB 초과** | 멀티파트 업로드 |
| SSD 50GB ~ 2,048GB / 1초 ~ 30일 | Storage Gateway 캐시 / 유효 시간 |

- ⭐ **수명 주기는 소급 적용 안 됨**(설정 이후 업로드분만).
- ⭐ **객체 잠금은 신규 컨테이너에만**. 아카이브·**복제 대상 컨테이너로 지정 불가**. 관리자도 삭제·덮어쓰기 불가.
- ⭐ **복제는 단방향**, 대상은 **빈 컨테이너**. 대상 삭제 후 같은 이름으로 재생성해도 **복제 재개 안 됨**.
- **IAM 권한 = 서비스 전체 / API 사용자 권한 = 컨테이너 단위.** 컨테이너에 READ만 줘도 IAM에 WRITE 있으면 쓰기 가능.
- `*:*` = 인증 토큰 있는 모든 사용자 / **`.r:*` = 인증 없는 공개** / `X-Container-View` = 목록 조회.
- **SLO = 매니페스트(JSON), 무결성 검증 O / DLO = 헤더 방식, 검증 X**(세그먼트 빠져도 다운로드됨).
- **Striped LVM = PV 1개 고장 → LV 전체 손상**(임시·스크래치용). 안정성 우선은 **Linear**.
- Block Storage는 **AZ 종속**, 루트 볼륨은 분리 후 타 인스턴스 연결 불가, **연결 중 증설은 가능**.
- **NKS = CSI + PVC / NCS = NFS 마운트 포인트**.
- **Storage Gateway = Object Storage를 NFS(v3/v4)로 마운트**. CIFS 미지원 → **Windows 파일 공유는 NAS(CIFS)**.
- 캐시: **Read Through=읽기 최적화 / Write Back=쓰기 최적화**.
- 지연 Block < NAS < Object, 처리량은 **Object가 최고**.
- 정적 웹 호스팅은 **웹서버·LB·오토스케일 불필요** + PUBLIC 설정.

---

## 5. Database

### 엔진별 (가장 자주 나옴)

| | MySQL | MariaDB | PostgreSQL | MS-SQL |
|---|---|---|---|---|
| 리전 | 판교·평촌·**도쿄** | **판교만** | 판교·평촌 | **판교만** |
| 읽기 복제본 | 5대 + ⭐**교차 리전(판교↔평촌)** | 5대, 교차 X | 5대, 교차 X | ⭐**미제공** |
| 교차 리전 백업 복제 | ⭐**O** | X | X | X |
| 스토리지 확장 | 자동·수동 | 자동·수동 | **수동만(재시작)** | **수동만(재시작)** |
| Failover 후 IP | **VIP, 안 바뀜** | **안 바뀜** | ⭐**바뀜(DNS/JVM TTL)** | ⭐**바뀜** |

→ **리전 단위 DR이면 무조건 MySQL.**

### 백업

| | MySQL/MariaDB | PostgreSQL | MS-SQL | EasyCache |
|---|---|---|---|---|
| 자동 백업 보관 | **730일** | 730일 | **30일** | **30일** |
| 방식 | 전체/증분 | **전체만** | 전체/**차등**+로그(5분) | **전체(RDB)만** |
| 시점 복원 | O | O(WAL, **7일**) | O | ⭐**X** |
| 복원 대상 | 신규 | 신규 | 신규 | **신규/기존 캐시** |

- 수동 백업은 전부 **영구**. DB 삭제 시 자동 백업 **선택 삭제 가능한 건 MySQL·MariaDB뿐**.
- ⭐ **PITR은 원본 인스턴스 필요**(로그가 원본 스토리지에 있음). 원본 지우면 백업 시점 복원만.
- **RDS 복원은 항상 새 인스턴스**(덮어쓰기 불가). **EasyCache만 기존 캐시 덮어쓰기 가능**.
- OBS 백업·복원은 **모든 엔진 동일 리전만**.
- 백업 수행 노드: RDS는 **예비 마스터**, **EasyCache는 HA여도 마스터**.
- 증분 = 직전 백업 이후 / **차등 = 마지막 전체 백업 이후 누적**(복구 파일 2개).

### EasyCache

- 복제본 **동일 리전 2대 + 교차 리전 1대**.
- **예비 마스터 없음** → Sentinel **과반수(3중 2)** 합의로 복제본 승격. **기존 마스터는 disable, 자동 복구 안 됨** → 삭제 후 복제본 재추가.
- ⭐ **교차 리전 복제본은 장애 감지 X, 읽기 전용 도메인 X, Failover 대상 X** (DR 용도).
- 모든 복제본 장애 시 **자동으로 마스터에 바인딩되지 않음** → 앱에서 전환.

### 캐싱 전략

- 읽기: **Cache-Aside**(미스 시 DB 부하 집중) / **Eager Loading**(사전 적재, 이벤트 폭주 대비) / Edge Caching(CDN+GSLB).
- 쓰기: Write-through(정합성, 잔액·포인트) / Write-around(로그성) / **Write-back**(EasyQueue 완충, **멱등성 필수**).
- ❌ 비권장 조합: **Cache-Aside × Write-back**, **Edge Caching × Write-back**.
- DB 모델: RAG·의미 검색 = **PostgreSQL + pgvector**, 사기 탐지 = 그래프, 세션·캐시 = 키-값, 대규모 집계 = 컬럼 기반.
- **Tibero·CUBRID·OS 에이전트·MHA/Pacemaker → RDS 아니라 DB Instance.**

---

## 6. DR / 고가용성

| 전략 | RTO | 보조 리전 상태 | 구분 |
|---|---|---|---|
| 백업 및 복원 | **수 시간** | 리소스 없음, 백업만 | Active-Passive |
| **파일럿 라이트** | **수십 분** | **DB 복제본 등 핵심만** | Active-Passive |
| **웜 스탠바이** | **수 분** | **앱 계층까지 최소 규모 상시 실행**(트래픽 X) | Active-Passive |
| 다중 사이트 | 실시간 | 양쪽 동시 트래픽 | **Active-Active** |

- ⭐ **장애 전환에서 GSLB 트래픽 전환은 항상 마지막.**
  - 파일럿 라이트: 이미지로 ASG **생성** → **Read Replica 승격 + HA 활성화** → 스토리지 경로 변경 → **GSLB**
  - 웜 스탠바이: 장애 감지 → DB 승격 → **이미 떠 있는 앱 계층 Scale-out** → 스토리지 경로 → **GSLB**
- **RPO = 데이터 유실 구간(복제·고빈도 백업으로 감소) / RTO = 중단 시간(자동 장애 조치·Standby로 감소).**
- 교차 리전 읽기 복제본은 **읽기 전용**. 쓰기는 **리전 피어링으로 주 리전 마스터에**.
- **관리형 DB는 Active/Active 미지원** → Galera, NDB, Cassandra, MongoDB 등.
- **L4 헬스 체크는 앱 오류를 감지 못 함** → L7 `/health` 200 OK.
- **NKS Probe = Readiness + Liveness** (Readiness 실패 → **트래픽 차단**, Liveness 실패 → **재시작**) / **NCS = Startup + Liveness**.
- 멀티 AZ 전제: 동시 장애 없음, LB 자동 전환, 무상태, **DB·스토리지·인증까지 이중화**. ⚠️ **서브넷은 AZ별로 나눌 필요 없음**.
- AZ 단위 서비스 = **Instance, NKS, RDS, Block Storage**. DNS Plus·CDN·CloudTrail·Cloud Monitoring은 AZ 지정 없음.
- 가용성 = MTBF/(MTBF+MTTR). **99.9% = 연 약 8.7시간**.
- **CAP**: 금융·재고 = **CP**, SNS·로그·스트리밍 = **AP**.
- 저사양 다수 = 단가는 싸도 **라이선스 누적·초기화 오버헤드·확장 지연** → **TCO로 판단**.
- **이미지 vs 스크립트 균형형**: OS~런타임은 **이미지**, 라이브러리·코드 배포·모니터링은 **스크립트**. 시작 시간은 **RTO와 직결**.
- Velero: **StorageClass는 백업 안 됨** → 복구 전 동일 이름 생성. 리전 간 DR에서는 **파일 단위 백업** + Object Storage 리전 간 복제 필수.
- **Instance Backup 리전 간 복제(소산)는 공공 클라우드 전용.**

---

## 7. 로깅 / 모니터링 / 데이터 플랫폼

| 요구사항 | 정답 |
|---|---|
| 누가 어떤 API를 호출했나 | **CloudTrail** (조직 자동 활성화, 콘솔 조회 **90일**, 장기 보관은 OBS) |
| 리소스 생성·변경 즉시 알림 | **Resource Watcher** |
| 위험한 보안 설정 점검 | **Security Advisor** (프로젝트 단위, **한 리전에서 켜면 전 리전 적용**) |
| 패킷 ACCEPT/REJECT | **Flow Log** |
| 인프라 지표 임계치 | **Cloud Monitoring** (보관 **최대 1년**) |
| 사용자 체감 가용성, Webhook | **Service Monitoring** (가상 브라우저) |
| 여러 로그 상관 분석 | **SIEM** (수집→탐지→**분석(상관)**→대응) |
| 느린 쿼리·데드락 | **RDS 분석** (⚠️ **MySQL·MariaDB만**) |
| 실시간 이벤트 분산 전달 | **EasyQueue** (Producer=Push, Consumer=Pull) |
| 코드 없이 ETL·암호화·Parquet | **DataFlow** (Cipher 필터, Parquet 코덱) |
| 데이터 이동 없이 조인 | **DataQuery** (Trino, Federated Query) |
| 앱·서버 로그, 모바일 크래시 | **Log & Crash Search** |

- ⭐ **예산 알림은 실시간 아님**: 전일 사용량 기준 **다음 날 오전 10시**. 월 중간 추세는 **임계치 기준일**.
- **EasyQueue 병목 지표 = Consumer Group Lag**(CPU 아님). 처리량은 **파티션 수**가 상한. 같은 그룹은 나눠 처리, **다른 그룹은 각자 전체 소비**.
- 로그 수집: **NKS = DaemonSet(또는 사이드카) / NCS = 사이드카만** / VM·온프레미스 = Logstash·Fluent Bit / 모바일 = LNCS SDK.
- AI EasyMaker: 분산 학습 **최대 10노드**, 체크포인트 공유 = **NAS(동일 프로젝트만)**, **승인된 모델만 배포**, 실시간=Endpoint / 대량=**Batch 추론**.

---

## 8. 규제 (공공 / 금융)

### 공공

| 등급 | 기준 | 영역 분리 |
|---|---|---|
| 상 | 국가 중대 이익, 수사·재판, 주민번호 대규모 | **물리적** |
| 중 | 비공개 업무자료, 개인정보 포함 | **물리적** |
| 하 | 개인정보 없는 공개 공공데이터 | **물리적 또는 논리적** |

- 업무망은 **인터넷 접점 없음**, 망연계는 **일방향 또는 전용 망연계 솔루션(스트리밍)**.
- **CSAP는 아직 운영 중.** 2027 하반기 국정원 단일 검증, 기존 인증은 **유효기간 5년까지** 인정.
- **N2SF ≠ 망분리 폐지**. 순서 = **업무정보 식별 → 등급 분류 → 시스템 등급 확정**. 혼재 시 **최고 등급**(O+S → S).
- **PPP Cloud 이용 ≠ N2SF 자동 충족.**
- 국가 보안관제: LB **TERMINATED_HTTPS** 복호화 → IDS/IPS, 상위 관제센터와는 **IPSec VPN**.

### 금융

- 절차: 중요도 평가 → **CSP 평가** → BCP·안전성 방안 → **정보보호위원회 심의·의결** → **계약** → 이용 → **사후 보고(3개월 이내)** → 출구 전략.
- **최종 책임은 금융회사**(금보원 대표평가를 써도 동일).
- **종합평가 최소 3년 / 정기평가 매년.** (숫자 뒤바꿈 주의)
- **망분리 기본 = 물리적**, 예외는 **R&D 등 비업무 + 사전 위험분석 + VDI·DRM**.
- **고유식별정보·신용정보는 국내 전산실.** **Exit Plan은 문서화 의무.**
- 랜딩존: **온프레미스 경로(Security VPC 1) / 인터넷 경로(Security VPC 2) 분리**, Untrust·Trust 분리, VPC 간은 **Transit Hub**, **PRD/STG/DEV 완전 분리**.
- 산업 특성: 금융은 **IaaS 중심**(SaaS·PaaS에 보수적), 게임은 멀티리전+GSLB, 미디어는 Edge Cache+Origin Shield, 제조는 Edge+Private.

### 프라이빗 클라우드 3종

| | Deck | Station | Region |
|---|---|---|---|
| 규모(VM) | 240~960 | 600~3,000 | **1,000~10,000+** |
| **AZ** | 1 | 1 | **2** |
| 네트워크 | **VLAN** | VxLAN | VxLAN |
| 관리 | **고객 직접** | NHN Cloud | NHN Cloud |
| 구축 | **1개월 이내** | 2개월 | **5개월** |

→ **무중단(멀티 AZ) 필수 = Private Region.**

---

## 9. 마이그레이션

### 6단계와 KEY

| 단계 | KEY |
|---|---|
| Discovery | 리스크 리스트업 |
| **Planning** | ⭐ **PoC 계획 수립 + Cutover 전략(전환 시점·롤백 기준)** |
| Pilot/PoC | **비용 모델·TCO 검토** |
| Execution | **롤백 플랜 실행 준비**, 웨이브 단위 이전 |
| **Cutover** | 최종 동기화 → ⭐**구 시스템 차단** → 트래픽 전환 → 사용자 검증 |
| Optimization | **임시 포트·계정 정리**, 문서 정비 |

- **PoC "계획"은 Planning**, PoC 단계는 TCO 검토. 헷갈리기 쉬움.
- 구 시스템 차단 이유 = **데이터 분기(split) 방지**.

### 도구 선택

- 서버: **사용자 이미지**(중지 후 생성, **qcow2** 변환, **메타데이터 생성 → 업로드**) / **ZConverter**(운영 중 증분, 에이전트 설치, virtio 자동).
- 컨테이너: **Velero** (저장소 = Object Storage S3 호환, NKS 쪽 **ReadOnly 권장**, 실패 원인 = StorageClass·LB 타입·CRD).
- DB: 중단 가능 → **외부 백업 복원**(XtraBackup, **5GB 이상 멀티파트**, 버전·문자셋·콜레이션 영향) / 중단 최소화 → **X-Log CDC**.
- CDC 사전 설정: MySQL **binlog_format=ROW**, PostgreSQL **wal_level=logical**, MS-SQL **CDC Enable**. Agentless여도 **CDC 전용 인스턴스는 필요**.
- 파일: **rsync=NFS/Linux, robocopy=CIFS/Windows**. Object Storage는 NFS/CIFS 직접 지원 X → Storage Gateway(NFS) / Windows는 NAS(CIFS).
- **Data Transporter 방향**: 이동식 장치 = **온프레미스 → NHN Cloud**(1대 **100TB**, 여러 대 가능) / 네트워크 = **NHN Cloud → 온프레미스**(고객이 직접 전송).
- 전략: 단기 효과 → **Migration(리호스트·리플랫폼)** / 장기 유연성·확장성 → **Modernization(리팩터링·리아키텍팅)**. 혼합이 일반적.

---

## 10. 아키텍처 설계 프로세스 (9단계)

요구사항·계획(01 요구사항 → 02 클라우드 모델 → 03 설계 원칙) / 인프라(04 데이터 → 05 네트워킹 → **06 보안**) / 운영·검증(07 자동화 → 08 모니터링 → 09 테스트)

- ⚠️ **01단계에서는 Auto Scale·CI/CD·다중 AZ 같은 구현 방식을 정하지 않는다.** 측정 가능한 요구사항 + KPI까지만.
- **데이터 보호·네트워크 보안 정책은 06 보안 설계에서 일괄 정의**(04·05 아님).
- 테스트 유형: **리전 failover = 복원력** / Auto Scale 확장·축소 = **확장성** / 한계 초과 = **스트레스** / OWASP = **보안** / REST API 연동 = **통합**.
- 도구: **JMeter=성능, Selenium=기능, SonarQube=코드 품질**.
- 성능 효율 원칙 예시 = 읽기 복제본 + CDN / 비용 효율 = Auto Scale + 이용 현황·예산 관리.

---

## 11. 교재 안에서 서로 어긋나는 것 (나오면 넘기기)

정답이 갈릴 수 있어 확신하기 어려운 항목들입니다. 다른 보기에서 확실한 근거를 먼저 찾으세요.

- PostgreSQL PITR 지원 여부 (본문 "미제공" vs 비교표 "WAL 기반 지원")
- EasyCache 자동 백업 보관 (730일 vs 30일 → **비교표의 30일** 기준)
- Velero PV 백업 방식 (CSI 스냅샷 vs 파일 단위 → **리전 간 DR은 파일 단위**)
- 멀티 AZ 가용성 표기 (Four 9's vs Six 9's → 계산값은 99.9999%)
- VPN 연결 VPC의 Transit Hub 동시 연결 (불가 vs 공유 구조로 가능)

---

## 마지막 한 줄

**"기술적으로 가능한가"가 아니라 "모든 조건을 동시에 만족하는가".**
비용 제한, 규제 등급, 권한 분리, RTO/RPO 중 **하나라도 어기면 오답**입니다.
