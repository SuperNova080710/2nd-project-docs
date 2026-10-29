# 07. BoatingDuck 보안·보안관제 실행계획 — SecOps / WAF / IDS·IPS / SIEM

> 상태: **실행 전 작업계획** · 기준일: 2026-10-29 · 주 담당: **BoatingDuck** · 인프라·배포 협력: **Odyz**
>
> 프로젝트: **K3s 기반 웹 서비스 보안 인프라 구축 및 통합 관제** · 기간 가안: 2026-10-29 ~ 2026-11-11
>
> 본 문서는 할 일을 정의한 것이며, 보안 솔루션 설치·정책 적용·차단 효과가 검증되었다는 의미가 아닙니다. **기존 개인 서버·데이터·서비스에 영향을 주는 실험과 설정 변경은 서버 소유자의 승인을 먼저 받습니다.**

## 1. 맡은 역할과 최종 결과

**목표:** Ubuntu/K3s에서 운영되는 테스트용 블로그에 대해 접근 통제(Firewall·NetworkPolicy), 웹 요청 보안(WAF), 침입 탐지(IDS), 통합 보안 이벤트 관제(SIEM)를 구현하고, 탐지·차단 여부를 **실제 로그와 통신 검증 결과**로 입증한다. Suricata **인라인 IPS는 조건부 추가 목표**이며 IDS와 구분해 기록한다.

| 영역 | BoatingDuck이 책임지는 일 | Odyz와 협력하는 일 | 결과물 |
| --- | --- | --- | --- |
| 보안 설계 | 자산·신뢰 경계·공격 표면·위협 시나리오·접근 행렬 정의 | 실제 Service/포트/진입 경로 확인 | 보안 토폴로지, 포트·흐름 허용표 |
| 호스트 Firewall | SSH/웹/K3s API/DB 등 허용·거부 정책과 사전 검증·복구 절차 | 서버 변경 승인 및 원격 관리 경로 보존 | 정책 전후 접속 결과 |
| Kubernetes NetworkPolicy | Pod 간 최소 통신·기본 차단 정책·DNS 등 필수 예외 정의 | namespace/labels/Service/실제 연결 경로 | 매니페스트, 정책 검증 결과 |
| WAF | ModSecurity/OWASP CRS 모드, 룰·예외·경보/차단 정책 | WAF 컨테이너 배포, Traefik·API 연동 | 설정, 룰 ID, 정상/의심 요청 증적 |
| 관제 | Wazuh 구조·배치 결정, 수집·파싱·규칙·경보·대시보드 | 앱 로그 필드, CPU/RAM 예산·마운트/접근 | 관제 이벤트, 검색·대시보드 증적 |
| IDS/IPS | Suricata IDS 패킷 경로·룰·EVE JSON; 인라인 가능성 검토 | 네트워크 인터페이스/시험 경로/서비스 가용성 | IDS 탐지 증거, 조건부 IPS 차단 증거 |
| 검증 | 탐지율·차단율·오탐·탐지 지연 검증 | 블로그 정상성/회귀·재시작/운영 측정 | T04~T12 보안 시나리오 및 증적 |

**경계:** Odyz는 K3s·이미지·Deployment/Service·Ingress·TLS·DB·배포/롤백을 소유합니다. BoatingDuck은 **정책 내용과 탐지·관제 결과**를 소유합니다. WAF·Wazuh·Suricata의 실행 위치와 배포 방식은 함께 정하고, 운영 클러스터 적용은 Odyz의 변경 승인 절차를 따릅니다. Odyz가 WAF 컨테이너를 올렸다는 이유만으로 보안 규칙 검증 완료로 보지 않습니다.

## 2. 공통 선행조건 및 보안 작업 원칙 (D1)

- [ ] 작업 승인 범위: 테스트 도메인/서버/클라이언트, 시간, 실험 가능한 트래픽, 운영 중 개인 서비스 목록 확정
- [ ] Ubuntu CPU/RAM/디스크 여유, NIC/라우팅/SSH 경로, Docker/K3s/Traefik/기존 Firewall 현황 확인
- [ ] 프로젝트용 DB·사용자·데이터가 기존 개인 DB와 격리됨을 Odyz와 확인
- [ ] 웹 외부 진입점/내부 API·DB Service 이름/포트·Namespace·Pod label 목록 수령
- [ ] 현재 UFW/nftables/iptables, 공유기/NAT, Kubernetes 서비스 포트·NodePort·HostPort 등 우회 경로 확인
- [ ] 최소한의 백업(정책 파일, 현재 규칙 덤프, 복구 콘솔/접속 경로) 및 **변경 실패 시 원복 방법** 확보
- [ ] 정상 접속·로그인·글 작성 등 '보안 적용 전' 베이스라인 수집
- [ ] 시간 동기화 및 로그의 시간대·request_id 정책 합의
- [ ] 보안 변경 전 Odyz에게 변경 파일·영향·테스트·원복 방법을 공유

**운영 금지:** 개인 서버의 기존 컨테이너/DB/볼륨을 삭제하지 않습니다. 기존 SSH 연결이 끊길 수 있는 무계획 default-deny/방화벽 일괄 초기화, 개인 운영 트래픽 대상 무승인 IPS 인라인, 인터넷 전체에 Wazuh Dashboard/Indexer·K3s API·DB 개방을 하지 않습니다.

### D1 기준선 점검 (읽기 위주)

```bash
sudo ss -tulpen
sudo ufw status verbose
sudo nft list ruleset
sudo iptables-save
ip -brief address
ip route
docker ps -a
```

실행 결과에는 내부 주소/운영 정보가 포함될 수 있으므로 증적 공유 전 마스킹합니다. 방화벽이 실제 사용 중인 netfilter 체계(UFW/nftables/iptables, Docker, K3s)에 어떻게 적용되는지 먼저 확인합니다.

**D1 완료 조건:** 승인된 대상, 변경 전 상태, 실제 트래픽 경로, 접근 행렬 초안, 정책 백업·원복 방법을 문서에 확보.

## 3. D2~D4 — 보안 설계와 인프라 계약 확정

### 3.1 자산·신뢰 경계·위협 모델

| 자산·경계 | 방어 목표 | 확인할 경로 |
| --- | --- | --- |
| 외부 브라우저 → HTTPS Ingress | 웹만 허용; 관리 포트 불필요한 공개 금지 | TLS 종료 위치, 신뢰된 프록시, 원본 IP/헤더 |
| Ingress → WAF → 프론트/API | 모든 **외부** 웹 요청의 WAF 경유 보장 | 직접 공개 Service/NodePort/hostPort/기존 프록시 확인 |
| 프론트 → 백엔드 | 필요한 HTTP API만 통과 | SSR 내부 호출과 브라우저 호출 경로 구분 |
| 백엔드 → 프로젝트 DB | 전용 사용자·허용 출발지만 접근 | DB가 클러스터 외부일 때 NetworkPolicy만으로 통제 가능하다고 가정 금지 |
| Pod ↔ Pod / DNS | 불필요한 동서 트래픽 제한, DNS 보장 | label/namespace selector, CNI 정책 적용 지원 |
| 관리 단말 → SSH/K3s/Wazuh | 승인된 관리자만 접속 | 접속 IP/사설망, 인증·TLS, 운영 포트 |

- [ ] 정상 연결이 필요한 출발지/목적지/포트/프로토콜/이유를 **허용표**로 작성
- [ ] 외부 공개 경로, 관리자 접속 경로, 내부 서비스 간 경로를 구분
- [ ] 인증 실패·의심 웹 요청·직접 API 접근·Pod 간 횡적 연결 등 검증 시나리오 정의
- [ ] 서명키·비밀번호·JWT·쿠키 원문 등 수집 금지 정보와 보관 기한 결정
- [ ] 애플리케이션 자체 인증·인가·DTO 검증은 WAF가 대체하지 못함을 확인

### 3.2 Odyz와 사전 합의할 기술 인터페이스

| 요청할 정보 | 이 정보를 필요한 작업 | 합의가 끝났음을 입증할 것 |
| --- | --- | --- |
| 실제 namespace, Pod labels, Service 이름·포트 | NetworkPolicy | 배포 상태 캡처·허용표 |
| Traefik→WAF→Next.js/API 경로, /api 변환 지점 | WAF | 경로도·정상 요청 결과 |
| TLS 종료 지점, 프록시 전달 헤더·신뢰 범위 | WAF/로그 | 허위 X-Forwarded-For 신뢰 방지 |
| 앱 로그의 UTC 시각, request_id, 경로, 상태, 이벤트 종류 | Wazuh | 민감정보 없는 샘플 이벤트 |
| WAF/로그 수집용 디스크 경로·읽기 권한 | Wazuh | 수집 경로·권한 검증 |
| 서버 자원 여유/네트워크 인터페이스/관리 접속 방식 | Wazuh/Suricata/Firewall | 자원 예산 및 복구 허가 |

**D4 게이트:** 서비스가 K3s에서 로그인/글 조회/작성이 정상인지 Odyz의 실측을 확인하기 전, 차단 정책을 운영 서비스에 강제 적용하지 않는다.

## 4. D5~D6 — Ubuntu 호스트 Firewall

**BoatingDuck 책임:** 허용 포트 정책 설계·검증, 변화 전후 접근 결과, 롤백 문서. **Odyz 책임:** 서버 운영 영향 확인 및 변경 승인.

- [ ] 현재 정책/포트 리스닝 상태와 공유기 포트포워딩/외부 접속 경로 기록
- [ ] 웹 공개 포트와 관리자 접속 IP/포트, K3s API 접속 범위 정의
- [ ] 원격 SSH 접속과 기존 서비스 관리 경로를 먼저 허용; 대체 접근 방법 확보
- [ ] 불필요한 DB·백엔드·SIEM·K3s 포트 직접 접근 차단 계획
- [ ] Docker/K3s의 NAT·포워딩·ServiceLB가 UFW 동작에 미치는 영향 확인
- [ ] 현재 SSH 세션 유지 중 별도의 관리 연결 시험 및 복원 가능한 변경만 순차 적용
- [ ] 승인된 PC/승인되지 않은 경로에서 실측; 정책 전후 상태 비교
- [ ] 차단 후 기존 개인 서비스, 프로젝트 웹, SSH, DNS, DB 통신이 정상인지 회귀 검증

**검증 결과 예시:** 허용된 443 웹 접속 성공, 외부 5432/3000/K3s API는 의도된 정책대로 접근 불가, SSH 승인 경로 유지. 실제 리스닝 포트/네트워크 환경을 확인하기 전 특정 포트를 일괄 차단하지 않는다.

**실패 시:** 신규 규칙만 되돌리고 적용 전 규칙/서비스 상태로 복구한다. 전역 `iptables -F`·방화벽 중단으로 해결하지 않는다.

## 5. D5~D6 — Kubernetes NetworkPolicy

- [ ] K3s 설치 시 실제 CNI/NetworkPolicy 컨트롤러의 정책 지원과 적용 방식을 확인
- [ ] namespace/pod label, Service 포트, DNS 요청, API/DB 경로를 Odyz의 배포 설정에서 확인
- [ ] 순차적으로 **테스트 Namespace**에서 기본 ingress/egress 제한을 적용
- [ ] 허용 경로: Ingress/WAF→Frontend, WAF/API 프록시→Backend, Frontend SSR→Backend, Backend→프로젝트 DB, DNS·필수 제어 트래픽
- [ ] DB가 호스트/외부에 있으면 Pod의 egress 정책뿐 아니라 DB 자체 바인딩·호스트 Firewall/PG 접근 정책을 함께 확인
- [ ] HTTP 정상 요청·로그인·DB 접근·DNS 등 필수 연결이 유지되는지 확인
- [ ] 임의 시험 Pod에서 허용되지 않는 서비스 접근을 시도해 차단 증거 확인
- [ ] 기존 Pod, kube-system, 관리 기능에 미치는 영향 검토
- [ ] 정책 변경마다 YAML 차이, 적용 시각, 예상 결과, rollback 파일 남기기

검증 예시(실제 설치/정책 적용 후):

```bash
kubectl -n blog-security get networkpolicy -o wide
kubectl -n blog-security describe networkpolicy
kubectl -n blog-security get pods --show-labels
```

**주의:** NetworkPolicy는 서비스 인증/인가를 대체하지 않으며, 기본 차단은 정책에서 선택한 Pod와 방향에 대해서만 적용된다. 같은 Namespace라고 자동 허용하지 않으며, DNS/외부 DB/프록시 통신 허용 규칙을 누락하면 정상 서비스도 실패할 수 있다.

**완료 조건(T08):** 최소 1개 필요 통신 허용 사례와 1개 불필요 통신 차단 사례를 동일 조건에서 기록. 보안 정책을 적용했다는 사실만으로 차단 효과를 주장하지 않는다.

## 6. D5~D6 — WAF: ModSecurity + OWASP CRS

**Odyz:** 컨테이너/Service/Ingress/TLS/업스트림 연결과 배포 안정성. **BoatingDuck:** 규칙/모드/탐지·차단 증적/오탐/보안 로그.

### 6.1 도입·설정

- [ ] OWASP CRS의 공식 `modsecurity-crs` 컨테이너 계열을 검토하고 **실제 사용 이미지·태그·digest**를 기록
- [ ] 배포 경로에 WAF를 놓고 프론트·API로 정상 요청이 전달되는지 확인
- [ ] DetectionOnly(또는 해당 배포의 탐지 전용 동작)를 우선 사용, 정상·의심 테스트 로그 확인
- [ ] CRS 룰셋, Rule ID, anomaly score, paranoia level, 정책 예외의 의미·변경 이력 정리
- [ ] SQLi/XSS 등 **승인된 테스트 요청**의 탐지 로그와 정상 로그인/게시글 작성의 오탐 여부 비교
- [ ] 반복 재현과 오탐 분석 후에만 차단 모드로 전환, 적용 시 Odyz 승인
- [ ] 탐지/차단 시각, 요청 ID, 경로, 상태코드, 룰 ID/조치 상태 등 필드 확인
- [ ] WAF가 기존 API·DB·인증/인가를 대신하지 않음을 문서화
- [ ] Authorization/Cookie/비밀번호/요청 본문 등 민감정보가 audit log에 수집되는지 확인하고 마스킹·로그 축소
- [ ] Rule 파일·보안 설정을 별도 Git 이슈/브랜치로 관리. 운영 자격증명은 Git 제외

### 6.2 검증과 중단 기준

| 검증 | 사전 조건 | 합격 근거 |
| --- | --- | --- |
| 정상 요청 | WAF 경유, 테스트 로그인/조회/작성 | 정상 응답 및 잘못된 차단 없음 |
| 탐지 | 승인된 안전한 테스트 패턴 | WAF Rule ID와 탐지 기록 |
| 차단 | 탐지 검증 후 차단 모드 | 의도한 차단 응답과 룰 로그 일치 |
| 우회 방지 | 외부 테스트 PC에서 직접 API 접속 시도 | 프론트/WAF 외 노출 경로 없음 |
| 오탐 회귀 | 정상 API 샘플 반복 | 오류 패턴·원인·예외 검토 기록 |

**문제 발생 시:** 신규 룰/차단 모드를 이전 검증 상태로 되돌린다. **운영 웹 트래픽이 검사 없이 우회하도록 새 공개 API 포트를 열지 않는다.** 긴급 가용성 확보는 사전 합의한 접근 제한/안전한 롤백 경로에서 수행한다.

### 6.3 구현 시 기술적 주의

- TLS를 Traefik에서 종료하는 경우 WAF에는 복호화된 HTTP가 전달되는지 확인한다.
- 클라이언트 IP는 **신뢰된 프록시 체인**에서만 인식하도록 설정해 헤더 위조로 정책·로그가 오염되는 것을 방지한다.
- Next.js 서버 컴포넌트 → 내부 API 호출은 WAF 미경유일 수 있으므로 '외부 HTTP 요청 WAF 강제'와 구분해 신뢰 경계·NetworkPolicy를 기록한다.

**완료 조건(T05/T06/T07):** 정상 서비스 유지 + 의심 요청 탐지/차단 증적 + 외부 API 직접 접근 불가를 모두 확인.

## 7. D7~D8 — Wazuh SIEM 및 통합 보안 로그

### 7.1 설치 위치를 먼저 결정

- [ ] Wazuh Manager, Indexer, Dashboard, Agent의 역할·필수 포트와 CPU/RAM/디스크 요구 확인
- [ ] N100 4코어/16GB 서버에서 K3s/앱과 합산한 **실측 자원**을 바탕으로 설치 위치 결정
- [ ] 같은 서버에 모두 넣는 경우 운영 서비스 지연/OOM/디스크 부족 위험을 평가
- [ ] 여유가 부족하면 승인된 별도 장비 설치 또는 경량 중앙 로그 수집으로 목표 축소
- [ ] 단순 로그 파일/수집만 구현한 경우 **'Wazuh SIEM 완료'로 표기하지 않음**
- [ ] Wazuh Dashboard/Indexer API의 외부 직접 공개 금지; 관리자 접근 통제/TLS/인증 계획

### 7.2 로그 수집 체계

**Odyz 제공:** NestJS 애플리케이션의 timestamp, request_id, route, method, status_code, event_type 등 구조화 로그. **BoatingDuck 제공:** 수집/파싱/필터·경보 규칙과 사건 검색 화면.

| 출처 | 최소 확인 이벤트 | 검증 방법 |
| --- | --- | --- |
| WAF | 탐지/차단, Rule ID, 요청 경로·시각 | 테스트 Rule 이벤트와 동일 로그 |
| NestJS API | 인증 실패·HTTP 오류·상관 ID | 테스트 계정의 실패 시도 |
| Ubuntu 호스트 | 로그인·권한 변경 등 승인된 보안 이벤트 | 테스트 호스트 이벤트 |
| Suricata | IDS alert, flow/packet 메타데이터 | 허용된 시험 트래픽의 EVE 기록 |

- [ ] 시간 동기화(UTC 기준 권장), 노드와 수집기 시간차 측정
- [ ] 로그 경로·수집 권한·회전/보존 기간·디스크 사용량 계획
- [ ] 필요한 필드: `timestamp`, `source`, `request_id`, `route`, `status_code`, `rule_id`, `action`, `result`
- [ ] 변환·정규화 후 룰과 경보를 만들고 이벤트 검색 조건 저장
- [ ] 정상 로그인과 의심 요청의 로그를 각각 입력해 탐지·관제 여부 확인
- [ ] 민감한 Authorization/Cookie/JWT/Refresh Token/비밀번호 원문 기록 차단
- [ ] 요청 ID가 어느 계층에서 누락되면 이를 기록하고 단순히 같은 시간대라는 이유만으로 동일 사건이라고 단정하지 않음

**완료 조건(T09):** 한 요청에 대해 실제 WAF 이벤트, API 이벤트 및 관제 표시 시각을 확인하고 가능한 식별자/시각으로 연결. 이벤트 경로가 누락되면 원인과 제약을 기록.

## 8. D8 — Suricata IDS → 조건부 IPS

### 8.1 IDS 먼저

- [ ] 실제 서버 네트워크 인터페이스·패킷 경유 지점을 파악하고 **Suricata가 어떤 패킷을 볼 수 있는지** 확인
- [ ] Suricata 버전/룰·인터페이스·capture 방법 선택, 자원 비용 확인
- [ ] 승인된 시험 트래픽만 대상으로 IDS 탐지 모드를 구성
- [ ] `eve.json` 등 구조화 이벤트에 `timestamp`, `event_type`, `alert.signature_id`, `src_ip/dest_ip`, `flow_id` 등 실제 제공 필드 확인
- [ ] 테스트 이벤트와 IDS 탐지 기록을 연계하고 Wazuh/로그 수집 체계에 전달
- [ ] 네트워크 대역/호스트 선택 오류, 오탐, CPU/메모리 영향 분석

**중요:** TLS 종료 전에 암호화된 HTTPS 페이로드가 보이지 않는 경로에서는 HTTP 본문 SQLi/XSS 탐지를 입증했다고 주장하지 않는다. HTTP 웹 공격성 요청 탐지는 WAF와 분리해 평가한다.

### 8.2 IPS는 안전성 조건 충족 시에만

- [ ] 테스트할 패킷이 **실제로 IPS 인라인 경로를 경유**하도록 NFQUEUE/격리 테스트 네트워크 등 구성을 확인
- [ ] 운영 개인 트래픽과 격리, 원격 접속/기본 서비스 보존, fail-open/fail-close 영향 검토
- [ ] 관련 netfilter 규칙 변경 전 덤프·원복 명령 및 물리/원격 복구 경로 확보
- [ ] 제한된 허가 트래픽에 대해 패킷 경유·차단·정상 통신 유지 여부를 측정
- [ ] 실험 종료 후 정책을 원복하고 기존 서비스 정상 여부 재점검
- [ ] 조건이 충족되지 않으면 IDS 결과만 제출하고 IPS 미구현·사유 기록

**완료 조건:** T10은 탐지 로그, T11은 실제 경유/차단 결과가 별도 증적이어야 한다. IDS 이벤트만으로 IPS 성공을 주장하지 않는다.

## 9. D9 — 통합 보안 검증·정량 측정

모든 검증은 **팀 소유/승인된 테스트 블로그·테스트 PC·테스트 계정**에서만 수행한다. 실제 결과는 `05 검증 및 증적.md` 양식으로 기록하고, 미실행과 실패를 구분한다.

| TC | BoatingDuck 역할 | 공동/협력 및 판정 |
| --- | --- | --- |
| T01~T02 | WAF 정책 적용 후 로그인/접속 영향 확인 | Odyz 앱 정상 검증 |
| T03 | 인증·인가 거부 증적 및 보안 로그 협조 | Odyz API 상태와 권한 결과 |
| T04 | 외부 DB 포트 접근 거부 검증 | Odyz DB 격리 상태 |
| T05 | WAF 의심 요청 탐지/차단 규칙 검증 | 룰 ID·조치·응답 |
| T06 | 정상 로그인/글 작성의 WAF 오탐 검증 | 정상 응답률 |
| T07 | WAF 우회·API 직접 노출 검증 | 외부 접근 경로 확인 |
| T08 | NetworkPolicy 허용/차단 검증 | 정상/불필요 Pod 통신 비교 |
| T09 | Wazuh 경보·로그 상관 분석 | 앱 로그와 WAF 로그 연결 |
| T10 | Suricata IDS 탐지 이벤트 | 허가 트래픽·실제 alert |
| T11 | 조건부 IPS 인라인 차단 | 경유/차단/복구 여부 |
| T12 | 보안 솔루션 적용 전후 CPU/RAM·오류/응답 | Odyz 운영 측정 |
| T13 | 보안 정책·관제 재시작 후 동작 확인 | Odyz 서비스 복구 주도 |

### 지표와 산식

- 정상 요청 성공률 = 의도한 성공 응답 수 / 정상 요청 수
- 탐지율 = 사전 정의한 승인된 시험 공격성 요청 중 탐지된 수 / 시험 공격성 요청 수
- 차단율 = 차단 대상으로 정의한 시험 요청 중 실제 차단된 수 / 차단 대상 요청 수
- 오탐률 = 정상 요청 중 WAF가 잘못 차단한 수 / 정상 요청 수
- 탐지 지연 = 중앙 관제 표시 시각 − 이벤트 발생 시각(노드 시계 동기화와 수집 시간차 명시)
- 운영 영향 = 정상 및 시험 부하에서 CPU/RAM/디스크·재시작·응답 시간 실측 비교

지표에는 **샘플 수, 테스트 조건, 정책 버전, 시간대, 예외/누락, 측정 도구**를 함께 남긴다. 임의의 합격 수치를 선제적으로 만들어 넣지 않는다.

### 증적 기록 권장 포맷

```text
실행 일시(UTC/KST):
테스트 ID:
실행자 / 승인 대상:
정책/이미지/규칙 버전 및 모드:
요청·패킷 요약(민감정보 제거):
기대 결과:
실측 응답·로그·수집 시각:
Rule ID / alert ID / request_id:
CPU/RAM/서비스 영향:
증적 파일 경로:
판정: PASS / FAIL / BLOCKED / NOT RUN
이상 발생 시 원인·원복·재검증:
```

## 10. D10 — 보안 운영 인수인계

- [ ] 실제 트래픽 경로와 WAF/TLS/Firewall/NetworkPolicy 적용 위치가 반영된 도식
- [ ] 서버·클러스터/보안 솔루션 버전·룰셋·이미지 digest, 설정 파일 목록
- [ ] 방화벽 허용표·Kubernetes 허용표, 운영 변경 및 안전한 원복 절차
- [ ] WAF 탐지/차단 모드, 주요 룰·예외·오탐 기록과 변경 이유
- [ ] Wazuh 인프라 위치, 수집 경로, 이벤트 필드·경보 규칙, 로그 보관·삭제 정책
- [ ] Suricata 관찰 인터페이스·룰·IDS 결과와 IPS 조건부 검증 여부
- [ ] T04~T12 실제 증적과 탐지율·차단율·오탐률·지연 및 측정 조건
- [ ] 미완료 사항/위험·한계(N100/16GB, 단일 노드, IDS/IPS 경로 등)
- [ ] Odyz의 서비스 배포 성과와 BoatingDuck의 보안·관제 성과를 분리해 작성

## 11. 2주 업무 일정 및 중단 게이트

| 날짜(가안) | BoatingDuck 주 작업 | 종료 조건 |
| --- | --- | --- |
| D1 · 10/29 | 기존 정책·트래픽·자산·복구 경로 점검 | 보호/승인·기본 허용표 |
| D2 · 10/30 | 보안 요구사항·WAF/로그/정책 초안 | API 경로·로그 필드 합의 |
| D3 · 11/02 | NetworkPolicy 초안·WAF 이미지/설정 검토 | K3s/서비스 인터페이스 확인 |
| D4 · 11/03 | 배포 상태 점검·차단 전 정상 기준선 | 로그인·조회·작성 검증 수령 |
| D5 · 11/04 | WAF 배포 연동·탐지 모드 테스트 | 정상 요청·룰 탐지 로그 |
| D6 · 11/05 | 차단 모드, Firewall, NetworkPolicy | 허용·거부 행렬 실측 |
| D7 · 11/06 | WAF/API/호스트 수집, Wazuh 부하 평가 | 이벤트 상관 및 자원 기록 |
| D8 · 11/09 | Wazuh/대안 구성·Suricata IDS | IDS 이벤트·관제 상태 |
| D9 · 11/10 | T04~T12 통합 시험·조건부 IPS | 테스트 결과·복구·성능 |
| D10 · 11/11 | 정책·규칙·증적·한계 정리 | 제3자 재현 가능한 인수인계 |

**게이트:** D4 블로그 불안정 → 차단 정책 보류. D6 WAF 경유/우회 차단 불명확 → 관제 확장 보류. D8 Wazuh 자원 부족 → 별도 시스템/경량 수집으로 축소하고 미완료 기록. D9 안전한 인라인 경로 없음 → IPS 시도 중단, IDS만 제출.

## 12. GitHub 작업 분할 제안 (후속 이슈/브랜치는 별도로 생성)

| 상위 작업 | 권장 브랜치 예시 | 하위 작업/인수 조건 |
| --- | --- | --- |
| SEC-01 위협 모델·허용 행렬 | `docs/security-threat-model` | 자산·신뢰경계, 포트 허용표, 보안 시나리오 |
| SEC-02 호스트 Firewall | `feat/security-host-firewall` | 기준선/규칙/접근 테스트/원복 |
| SEC-03 NetworkPolicy | `feat/security-network-policy` | 정책 YAML·DNS/DB 예외·허용/차단 증적 |
| SEC-04 WAF | `feat/security-waf-crs` | 탐지→오탐→차단 모드·룰 로그 |
| SEC-05 Wazuh·로그 | `feat/security-wazuh-monitoring` | 자원 예산·수집·정규화·경보 |
| SEC-06 Suricata IDS | `feat/security-suricata-ids` | 인터페이스·EVE JSON·탐지 증적 |
| SEC-07 조건부 IPS | `feat/security-suricata-ips-lab` | 경유·원복·차단/정상 비교 |
| SEC-08 보안 검증·증적 | `docs/security-verification` | 테스트 ID/로그/표/한계 기록 |

각 상위 이슈는 **범위·선행조건·수정 대상·환경 정보·테스트 순서·완료 기준·원복 방법·Odyz 의존성**을 포함하고, 독립적으로 시작할 수 있는 작업을 GitHub 하위 이슈로 분리한다. 보안 변경 PR은 앱/인프라 담당 Odyz의 검토를 받아 병합하며, 실제 서버에 적용된 결과와 PR 병합을 별개로 표시한다.

## 13. 참고

- [01 프로젝트 개요](./01%20프로젝트%20개요.md)
- [03 아키텍처 구현설계](./03%20아키텍처%20구현설계.md)
- [04 2주 작업계획](./04%202주%20작업계획.md)
- [05 검증 및 증적](./05%20검증%20및%20증적.md)
- [Odyz DevOps 실행계획 PR #6](https://github.com/SuperNova080710/2nd-project-docs/pull/6)
- [Kubernetes NetworkPolicy 공식 문서](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [OWASP CRS 공식 컨테이너 안내](https://coreruleset.org/docs/1-getting-started/1-3-using-containers/)
- [Wazuh Quickstart](https://documentation.wazuh.com/current/quickstart.html)
- [Suricata IPS 개념](https://docs.suricata.io/en/suricata-8.0.5/ips/ips-concept.html)
- [Suricata EVE JSON](https://docs.suricata.io/en/latest/output/eve/eve-json-output.html)
- [Docker 방화벽 참고](https://docs.docker.com/engine/network/packet-filtering-firewalls/)
