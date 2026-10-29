# 06. Odyz 담당 실행계획 — DevOps / K3s / 웹 서비스 배포·운영

> 문서 상태: **실행 계획(미검증)** · 작성 기준: 2026-10-29  
> 담당: **Odyz** (서비스·인프라 운영) / 협력: **BoatingDuck** (보안·관제)  
> 적용 범위: `2nd-project-docs`, `Blog-backend`, `Blog-frontend` 및 승인된 Ubuntu 실습 서버  
> 원칙: **현황·백업 확인 → 서비스 기준선 → K3s → 앱 배포 → 진입점/WAF 연동 → 운영 자동화 → 보안 통합 검증**  
> 이 문서는 작업 지침이며 서버 접속, 설정 변경, 배포 또는 테스트 실행을 완료했다는 의미가 아니다.

## 1. 나의 책임과 최종 결과

**나의 임무:** 기존 Next.js/NestJS 블로그를 프로젝트용 데이터와 함께 Ubuntu의 **K3s 단일 노드**에서 실행하고, 재현 가능한 배포·업데이트·롤백·복구 및 보안 구성 요소의 연결 경로를 제공한다. 기존 개인 서비스와 DB는 보존한다.

| 구분 | 내가 직접 책임지는 결과물 | BoatingDuck과 합의할 사항 |
| --- | --- | --- |
| 서버 기반 | 현황/포트/자원/디스크 점검, 백업·복구 경로, K3s 설치 결정 | 보안 솔루션 자원 예산, SSH·포트 정책 |
| 앱 이미지 | Next.js/NestJS 이미지, 버전 고정, 비밀정보 제외 | 로그 보안 요구사항 |
| 데이터 | 프로젝트 전용 PostgreSQL DB·계정, 마이그레이션·백업·복원 | DB 접근 제한 |
| K3s 배포 | Namespace, Deployment, ClusterIP Service, ConfigMap/Secret 주입, Probe, 리소스 제한 | 최소 통신 행렬·NetworkPolicy |
| 외부 경로 | Traefik Ingress, DNS·TLS, WAF 앞뒤 라우팅, 우회 노출 점검 | WAF 정책·차단 모드 |
| 운영 | 버전 배포·롤백, 장애 대응, CI/CD 또는 수동 배포 절차, 상태·용량 기록 | 보안 변경의 릴리스 창 및 검증 |
| 앱 로그 | 요청 ID, 시각, 상태코드, 인증 실패 등 안전한 이벤트 출력 | Wazuh 수집, 룰·경보·대시보드 |
| 통합 검증 | 로그인·글 조회/작성·복구·성능 및 배포 증적 | 탐지·차단·오탐·IDS/IPS 증적 |

**책임 경계:** 내가 WAF 컨테이너/Service/Ingress 연결을 준비하더라도 **ModSecurity·OWASP CRS 탐지/차단 규칙, Firewall 정책, Kubernetes NetworkPolicy 정책, Wazuh 및 Suricata 분석·운영은 BoatingDuck 주 담당**이다. 보안 설정의 운영 반영은 서로 검토하고, 서비스 가용성에 영향을 주는 변경은 합의 후 적용한다. NetworkPolicy YAML의 위치와 배포 파이프라인은 공동 관리할 수 있으나 보안 정책의 의미·검증 책임은 BoatingDuck이 진다.

## 2. 착수 전 확인해야 하는 미확정 사실 (D1 게이트)

**아래 정보가 확인되기 전에는 Ubuntu 서버에 K3s를 설치하거나 방화벽/포트·볼륨·DB를 변경하지 않는다.**

- [ ] 서버 소유자, 관리자 권한, 원격 접속 방식과 작업 승인 범위
- [ ] Ubuntu 버전/커널/아키텍처, CPU/메모리/디스크 여유량, 네트워크 인터페이스와 실제 IP 구성
- [ ] 기존 개인 서비스·Docker 컨테이너·데이터 볼륨·PostgreSQL 인스턴스·리버스 프록시 목록
- [ ] 실제 사용 중인 포트(특히 22, 80, 443, 3000/3001, 5432, 6443) 및 NAT·공유기 설정
- [ ] 기존 데이터 백업 경로, 복원 가능한 백업인지, 서비스 장애 시 접근할 수 있는 콘솔/복구 경로
- [ ] 프로젝트용 DB 방식(A: 기존 엔진의 별도 DB/계정, B: 별도 PostgreSQL 인스턴스), DB 저장 위치
- [ ] 내부 접속 우선 여부, 도메인·TLS 발급 방식, 공개 범위와 허용된 테스트 PC
- [ ] 외부 이미지 레지스트리 및 서버의 이미지 pull 권한, GitHub Actions의 배포 권한 범위
- [ ] N100/16GB에서 K3s+앱+보안 솔루션의 CPU/RAM/디스크 예산과 Wazuh 분리 여부
- [ ] K3s 기본 Traefik/ServiceLB와 기존 서비스의 80/443 충돌 여부
- [ ] 적용·복구를 수행할 수 있는 승인된 점검 시간과 실패 시 원복 결정권

점검 명령(읽기 위주; 실제 서버에서 승인 후 실행):

```bash
hostnamectl
uname -a
lscpu
free -h
df -h
lsblk -f
sudo ss -tulpen
docker ps -a
docker volume ls
systemctl status k3s --no-pager
sudo ufw status verbose
```

설정값, 개인 IP/도메인, SSH 키, DB 비밀번호, 토큰은 **Git 이슈/문서/스크린샷에 원문 게시하지 않는다**. 위 출력도 공유 전에 민감정보를 검토한다. 포트 목록은 실제 프로세스와 매핑해 기록한다. 기존 Docker 방화벽 규칙을 일괄 삭제하거나 Docker/K3s 제거·초기화 명령을 실행하지 않는다.

**D1 산출물:** `evidence/D1/` (민감정보 제거한 환경 점검표), 기존 서비스 기준선, 위험·복구 계획, 서버 소유자 승인 기록. 실제 증적이 없다면 완료 표시하지 않는다.

## 3. D2 — 기존 Blog 실행 기준선과 데이터 격리

### 3.1 코드와 빌드 점검

- [ ] `Blog-backend`의 현재 `main` 커밋 기록 및 `npm ci`, `npm run build` 결과 확인
- [ ] `Blog-frontend`의 현재 `main` 커밋 기록 및 `npm ci`, `npm run lint`, `npm run build` 결과 확인
- [ ] 테스트 계정으로 회원가입/로그인/갱신/로그아웃 확인
- [ ] 게시글 목록/상세/생성과 작성자 권한 검증. 기대 응답은 API 코드와 비교
- [ ] 실패 항목은 재현 조건/로그/원인을 이슈로 분리. 기존 서비스 정상 여부와 분리 표기

**확인된 코드 사실 (2026-10-29 `main` 검토):** Backend에는 Dockerfile, TypeORM 마이그레이션, `/health-check`가 있다. Frontend에는 Dockerfile이 없다. Frontend `src/api/client.ts`는 `NEXT_PUBLIC_API_BASE_URL`을 요구하며, `src/app/posts/[id]/page.tsx`가 서버 컴포넌트에서 `getPost`를 호출한다. Backend의 `npm test`는 실제 테스트 실행 스크립트가 아니다. 검증 미실행 상태를 테스트 통과로 간주하지 않는다.

### 3.2 프로젝트 DB 분리

- [ ] 기존 개인 운영 DB와 동일 데이터셋을 사용하지 않도록 프로젝트 전용 DB/계정 준비
- [ ] DB 접근 권한은 프로젝트 DB에 한정; 기존 계정·권한·데이터 유지
- [ ] 프로젝트 전용 접속값을 Secret으로 제공, 평문 값을 커밋하지 않음
- [ ] 기존 마이그레이션을 **테스트 DB에만** 실행, 테이블 목록 및 적용 이력 검증
- [ ] 테스트 데이터로 로그인/글 작성 검증, 백업 및 별도 복구 시험
- [ ] 기존 개인 DB 변경 여부와 복구 시험 결과 기록

`synchronize: false`는 유지한다. 앱 자동 시작 시 운영 DB에 임의 마이그레이션을 실행하지 않는다. 마이그레이션은 배포 전 별도 실행 절차(승인된 Job 또는 명시적 작업)로 분리하고 실패 시 앱 배포를 중단한다.

**D2 완료 기준:** 빌드 결과 및 테스트 DB의 로그인·게시글 흐름 증적, 원본 DB 보존 확인, DB 백업/복구 계획.

## 4. D2~D3 — Docker 이미지·배포 산출물 준비

### Backend

- [ ] 기존 `Blog-backend/Dockerfile` 빌드/실행을 검증하고 실행 사용자·기본 포트·종료 신호·헬스 엔드포인트 검토
- [ ] `.dockerignore`에 `.env`, 개인 자료, 비밀키/로컬 DB 파일이 이미지에 포함되지 않도록 확인
- [ ] 빌드/런타임 의존성 분리, 이미지 태그를 Git SHA 등 재현 가능한 값으로 관리
- [ ] DB 접속/서명키/런타임 포트는 런타임 Secret·환경 변수로 주입

### Frontend

- [ ] `Blog-frontend/Dockerfile` 및 `.dockerignore` 작성(멀티 스테이지 빌드)
- [ ] 가능하면 Next.js standalone 출력 구성과 실행 검증
- [ ] 브라우저 API는 동일 출처의 `/api`를 사용하도록 설계; 빌드 시 인라인되는 `NEXT_PUBLIC_*`의 특성에 주의
- [ ] `/api/*` → 실제 NestJS `/auth`, `/posts` 경로 변환 계층은 **한 곳**으로 결정하고 문서화
- [ ] Next.js 서버 컴포넌트의 내부 API 호출은 브라우저 URL과 분리: 내부 Service DNS 등 서버에서 접근 가능한 경로 사용
- [ ] SSR 내부 호출의 WAF 경유 여부는 보안팀과 별도 합의. WAF 미경유 내부 통신은 NetworkPolicy·인증으로 통제하며 "모든 요청 WAF 검사"라고 주장하지 않음
- [ ] 임시 이미지에 운영 DB 주소·비밀키·토큰이 포함되지 않았는지 확인

**산출물:** Dockerfile, .dockerignore, 로컬/격리 환경 이미지 실행 증거, 이미지 태그·digest, 변경한 API 경로와 환경 변수 매트릭스.

## 5. D3 — K3s 단일 노드 설치 및 클러스터 운영 기반

### 설치 전 결정

- [ ] 호스트 포트/라우팅, 기존 Docker/iptables/nftables/UFW 영향 검토
- [ ] K3s 버전 **실제 설치 버전 고정**, 설치 경로와 서비스 설정 백업
- [ ] Traefik/ServiceLB 활성화 여부를 기존 포트와 정책에 맞춰 결정
- [ ] 단일 노드 장애·서비스 가동 중단을 수용하는지 팀 합의
- [ ] K3s와 보안 솔루션의 디스크, CPU, 메모리 사용 예산 정의

### 설치 후 확인

- [ ] 클러스터 노드 Ready, CoreDNS/네트워킹/Ingress 등 필수 구성요소 정상
- [ ] `blog-security` Namespace 및 RBAC/서비스 계정/리소스 분리 초안
- [ ] Pod 간 DNS 조회, ClusterIP Service 연결 시험
- [ ] kubeconfig 접근 통제 및 백업: 개인 원격 관리자 자격증명을 Git에 저장하지 않음
- [ ] K3s 설정·클러스터 백업 및 복구 문서 마련(단일 노드의 한계 명시)

검증 예시(클러스터가 설치된 후):

```bash
sudo k3s kubectl get nodes -o wide
sudo k3s kubectl get pods -A
sudo k3s kubectl get svc -A
sudo k3s kubectl get ingress -A
```

**D3 완료 기준:** 실제 노드/Pod 상태 증적, 충돌 없는 기존 서비스 상태, 설정·백업 위치 기록. 개인 서비스 중단이 발생하면 배포를 확대하지 않고 롤백한다.

## 6. D4 — Kubernetes로 블로그 배포

매니페스트 제안 구조(현재 미구현; 저장소 위치는 협의 후 결정):

```text
k8s/
  base/
    namespace.yaml
    frontend-deployment.yaml
    frontend-service.yaml
    backend-deployment.yaml
    backend-service.yaml
    configmap.yaml
    kustomization.yaml
  overlays/
    test/
      kustomization.yaml
    production/
      kustomization.yaml
  policies/                  # BoatingDuck과 공동 검토
  ingress/                   # WAF 경유 경로 합의 후 배포
```

- [ ] Namespace, frontend/backend Deployment, **ClusterIP** Service
- [ ] 이미지 태그/digest 고정, requests·limits, readiness/liveness/startup Probe(실제 동작 검증)
- [ ] ConfigMap은 비밀이 아닌 값만, Secret은 Git 외부 안전한 경로에서 생성·주입; 권한/RBAC 점검
- [ ] API 및 DB는 NodePort/LoadBalancer/호스트 공개 포트 없이 내부 통신
- [ ] 컨테이너 재시작 후 DB 연결, 로그, 헬스 확인
- [ ] 인증/글 목록/상세/작성 실사용 경로 확인
- [ ] 리소스 부족·시작 지연 때 실패 증거를 확보하고 Probe/예산을 조정

예시 검증 명령(실제 리소스 생성 후):

```bash
kubectl -n blog-security get deploy,pod,svc
kubectl -n blog-security rollout status deployment/blog-backend
kubectl -n blog-security rollout status deployment/blog-frontend
kubectl -n blog-security get events --sort-by=.metadata.creationTimestamp
```

**주의:** 프로젝트 DB를 반드시 K3s 내부에 배포할 필요는 없다. 별도 PostgreSQL 서비스/인스턴스로 격리할 경우 접근 경로와 NetworkPolicy·방화벽 연계를 기록한다. 상태 저장 리소스는 백업/PV/PVC/스토리지 수명주기 확인이 우선이다.

**D4 완료 기준:** K3s에서 로그인·조회·작성 성공, API/DB의 외부 직접 노출 없음, 재시작 후 데이터 유지.

## 7. D5~D6 — Ingress·TLS·WAF 배포/연동 (공동 작업)

내 담당(배포·연결):
- [ ] DNS/도메인·인증서 조건 확인(처음에는 승인된 사설망부터 검증)
- [ ] Traefik Ingress로 **WAF Service**를 유일한 공개 웹 진입 대상에 배치
- [ ] WAF의 업스트림을 Next.js 및 API 프록시 경로에 올바르게 연결
- [ ] 정규 요청(로그인, 조회, 작성)이 경유 후도 정상 작동하는지 확인
- [ ] 백엔드/DB/WAF 관리·SIEM/K3s API의 공개 노출 여부 점검
- [ ] WAF 컨테이너 롤아웃/상태·로그 출력·장애 시 복구 담당
- [ ] 인증서 갱신 및 라우팅 장애 시 대응 절차 기록

BoatingDuck 주 담당(보안 정책):
- [ ] ModSecurity·CRS 규칙 버전과 탐지/차단 모드, 예외 정책
- [ ] WAF 탐지 이벤트·차단 증거, 정상 요청 오탐 검토
- [ ] 호스트 Firewall과 NetworkPolicy 허용 통신 명세
- [ ] 탐지·차단 정책 변경 후 보안 회귀 검증

**공동 수락 조건:** 웹 서비스 정상 요청 성공, 승인된 테스트 공격성 요청 탐지/차단 로그 확보, 백엔드 직접 우회 불가, 통신 정책 변경 전/후 비교. WAF가 우회 가능한 공개 API 포트가 남으면 **미완료** 처리한다.

**중요한 경계:** Traefik이 TLS를 종료한 뒤 WAF가 HTTP를 검사하는 구조라면 TLS 종단 위치, 신뢰 프록시 헤더 처리, 내부 구간 신뢰 모델을 명확히 기록한다.

## 8. D7~D8 — 운영 로그·관제 연결 및 자원 확인

내 담당:
- [ ] Backend 접근·오류·인증 실패 로그를 시각, 서비스명, 요청 ID, 메서드/경로, 상태코드 중심으로 구조화
- [ ] HTTP 요청마다 요청 상관 ID를 일관되게 전달하고 WAF 로그와 연결 가능하도록 연동 검증
- [ ] 로그에 비밀번호·JWT·Refresh Token·쿠키·민감 요청 본문을 남기지 않음
- [ ] stdout/stderr 수집 경로, 로그 보존·회전/용량 제한, 시계 동기화 확인
- [ ] CPU/RAM/디스크 및 Pod 재시작/OOM, 응답 지연 측정
- [ ] Wazuh 등 설치 전후 가용량 확인 및 경량 대안 제시

BoatingDuck 담당:
- [ ] WAF·Ubuntu·애플리케이션 로그 수집/파싱
- [ ] Wazuh Manager/Indexer/Dashboard 또는 제약을 명시한 대체 수집 체계
- [ ] Suricata IDS 이벤트 수집, IPS는 안전한 인라인 경로가 확인될 때만 수행

**완료 기준:** 승인된 한 테스트 요청에 대해 API/WAF/관제 타임스탬프·상관 ID로 사건을 연관 지을 수 있다. SIEM이 설치되지 않은 경우 "SIEM 완료"라고 표기하지 않는다.

## 9. D9~D10 — 운영 자동화·통합 검증·인수인계

### 배포 자동화

- [ ] 앱 소스 PR 시 lint/build 등의 CI 검사를 수행할 수 있도록 준비
- [ ] `main` 병합 → 검증된 이미지 빌드/레지스트리 push → 승인된 배포 경로를 문서화
- [ ] Production 배포는 수동 승인/권한 분리 우선, Kubeconfig/Registry Secret은 Git에 커밋하지 않음
- [ ] Git SHA ↔ 이미지 digest ↔ 배포 시각 ↔ K8s 배포 버전을 추적
- [ ] `kubectl rollout status/history/undo` 등 앱 롤백 검증; **DB 스키마 롤백은 별도 절차**로 관리
- [ ] 서버가 꺼지거나 디스크가 손상된 경우 단일 노드 K3s 자체로 고가용성·자동 복구되지 않음을 명시

### 최종 테스트(기획서 `05 검증 및 증적.md`와 연결)

| 원본 TC | 내가 주도할 검증 | 판정 근거 |
| --- | --- | --- |
| T01 | 허용된 테스트 PC에서 블로그 접속 | 실제 HTTP 응답/화면 |
| T02 | 로그인·로그아웃·세션 | 테스트 계정·상태 변화 |
| T03 | 타 사용자 글 수정·삭제 거부 | 실제 응답코드·DB 불변 |
| T04 | 외부 DB 포트 거부 | 허가된 PC의 연결 결과 |
| T06 | WAF 적용 후 정상 서비스 흐름 | 오탐 여부·응답 기록 |
| T07 | API 우회/직접 접속 불가 | 외부 접속 매트릭스 |
| T08 | 필요/불필요 Pod 통신 차이 | BoatingDuck 정책 협업 |
| T09 | 앱 이벤트·WAF 이벤트 연관 | 요청 ID·시각·관제 검색 |
| T12 | 부하 시 CPU/RAM·재시작/응답 | 측정치와 측정 조건 |
| T13 | 앱 재시작·DB 데이터 복원 | 복원 절차·결과 |

T05(WAF 공격성 요청), T10(IDS), T11(조건부 IPS)는 BoatingDuck 주도 테스트에 **서비스 가용성 및 트래픽 관찰 협력**으로 참여한다. 승인된 프로젝트 환경만 대상으로 한다.

### 최종 문서/인수인계

- [ ] 실제 배포 버전·서비스 맵·트래픽 경로·포트/통신 허용 행렬
- [ ] 매니페스트/빌드 지침·변수 이름 목록(비밀 값 제외)
- [ ] 이미지 배포, 버전 업데이트, 스케일·재시작·롤백 명령
- [ ] 서버/DB·K3s 상태 복구, 백업·복구 증적
- [ ] 테스트 케이스별 날짜·실행자·조건·로그·화면·판정
- [ ] 문제/원인/조치/재검증과 해결되지 않은 위험
- [ ] 개인 기존 블로그 개발 성과와 이번 DevOps·보안 프로젝트 기여 구분

## 10. 일정과 단계별 통과 게이트

| 날짜(2026) | 내 주 작업 | 완료 게이트 | 막힐 경우 |
| --- | --- | --- | --- |
| 10/29 D1 | 서버 점검, 기존 서비스 보호, DB 격리 결정 | 자원·포트·백업·승인 기록 | K3s 설치 보류 |
| 10/30 D2 | Blog 빌드, 테스트 DB, Docker 준비 | 로그인/글 흐름과 테스트 DB 증거 | 기반 앱 정상화 우선 |
| 11/02 D3 | K3s 기반·Namespace·Service | Node Ready/기존 서비스 영향 없음 | 설정/포트 충돌 복구 |
| 11/03 D4 | Frontend/Backend 배포 | K3s 실제 로그인·글 작성 | WAF 연동 착수 보류 |
| 11/04 D5 | Ingress/WAF 경유 라우팅 | 경유 정상 요청·우회 차단 | 노출 경로 수정 |
| 11/05 D6 | Firewall/Policy 적용 협력 | 허용 통신 통과·차단 실측 | 정책 롤백 및 원인 분석 |
| 11/06 D7 | API 구조화 로그·관제 연동 | 보안 이벤트 상관 관계 | 경량 로그 수집 우선 |
| 11/09 D8 | IDS 연동 협력·자원 시험 | 서비스 가용성·사용량 측정 | Wazuh 분리/범위 축소 |
| 11/10 D9 | 회귀·장애·복구·부하 시험 | TC 증적, 재현성 | 문제를 미완료로 기록 |
| 11/11 D10 | 운영 매뉴얼·최종 인수인계 | 설정, 증적, 한계와 역할 구분 | 미검증을 성공으로 표시하지 않음 |

날짜는 원본 계획의 **가안**이다. 안전성과 프로젝트 게이트를 위해 날짜보다 선행 조건을 우선한다.

## 11. 공동 협업 계약

1. **내가 제공할 인터페이스:** 공개 진입 URL/경로, 내부 서비스 이름·포트, 네트워크 허용 요구, WAF 업스트림, 앱 로그 스키마, K3s 자원 예산.
2. **BoatingDuck이 제공할 인터페이스:** WAF 정책 파일·변경 이력, Firewall/NetworkPolicy 허용표, 이벤트 필드·수집 구성, 탐지/차단 검증 결과.
3. **변경 책임:** 각자 담당 이슈·브랜치·PR로 작업. 공유 Ingress·NetworkPolicy·비밀 값·운영 서버 접근 변경은 상호 리뷰 및 사전 복구 계획 필수.
4. **커밋 권장 형식:** 한 줄 제목, 빈 줄, 변경사항 3~4개 불릿. 검증 근거는 PR 본문에 기재.
5. **병합/운영 배포:** PR 검토·병합과 실제 운영 적용은 구분한다. 문서·코드 병합만으로 서버에 반영되었다고 표시하지 않는다.

## 12. 예시 포트·통신 행렬 (계획; 실제 값 확인 필요)

| 출발지 | 목적지 | 프로토콜/포트(예시) | 허용 원칙 | 책임 |
| --- | --- | --- | --- | --- |
| 승인된 PC | Traefik HTTPS | TCP 443 | 접속 범위 합의 | 공동 |
| Traefik | WAF Service | HTTP 내부 포트 TBD | WAF만 공개 경유 | 나/BoatingDuck |
| WAF | Frontend / API 프록시 | 내부 HTTP 포트 TBD | 최소 경로 허용 | 나/BoatingDuck |
| Frontend 서버 | Backend ClusterIP | TCP 3000 (현재 기본값) | 필요한 API만 | 나 |
| Backend | 프로젝트 PostgreSQL | TCP 5432 (현재 기본값) | 전용 DB/계정 | 나 |
| Pod | 클러스터 DNS | 실제 DNS 포트 확인 | 이름 조회 보장 | 공동 |
| 승인된 관리자 | K3s API/SSH | 6443/SSH 포트(확인 필요) | 외부 전체 공개 금지 | 나/BoatingDuck |
| 미승인 외부 | Backend/DB/SIEM | 해당 포트 | 직접 접근 차단 | 공동 |

표의 주소·포트는 **배포된 값이 아닌 예시**다. 허용표는 실제 Service, 네트워크 인터페이스 및 원격 관리 경로를 확인한 뒤 확정한다.

## 13. GitHub 후속 작업 단위 제안 (아직 구현·완료된 이슈 아님)

- **INFRA-01 / D1:** Ubuntu 안전성·백업·자원/포트 점검 (완료: 점검표와 복구 승인)
- **APP-01 / D2:** Frontend/Backend 빌드 기준선·테스트 DB 격리 (완료: 실사용 API 흐름)
- **IMAGE-01 / D2:** 프론트 컨테이너화·백엔드 이미지 점검 (완료: 버전 고정 이미지 실행)
- **K3S-01 / D3:** 클러스터 설치·Namespace·RBAC (완료: Node Ready)
- **DEPLOY-01 / D4:** 매니페스트·내부 연결·Health Probe (완료: 로그인/글 작성)
- **EDGE-01 / D5:** TLS/Traefik/WAF 연결 (완료: 정상 요청과 직접 API 차단)
- **OBS-01 / D7:** 애플리케이션 구조화 로그·관제 연동 (완료: 상관 ID 추적)
- **OPS-01 / D9:** CI/CD·배포/롤백·DB 복구 (완료: 재현 가능한 실행 기록)
- **VERIFY-01 / D9~10:** 통합 TC·증적·최종 인수인계 (완료: 원본 T01~T13의 상태별 판정)

각 기능 이슈에는 목적·선행조건·수정 파일·수락 기준·담당자·검증 명령·롤백을 포함하고, 실제 구현은 **별도 이슈와 기능 브랜치**에서 진행한다.

## 14. 참고 자료

- [프로젝트 개요](./01%20프로젝트%20개요.md)
- [아키텍처 구현설계](./03%20아키텍처%20구현설계.md)
- [2주 작업계획](./04%202주%20작업계획.md)
- [검증 및 증적](./05%20검증%20및%20증적.md)
- [Blog-backend](https://github.com/SuperNova080710/Blog-backend)
- [Blog-frontend](https://github.com/SuperNova080710/Blog-frontend)
- [K3s 공식 설치 요구사항](https://docs.k3s.io/installation/requirements)
- [K3s 네트워킹 서비스](https://docs.k3s.io/networking/networking-services)
- [Kubernetes NetworkPolicy](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Kubernetes Secrets 가이드](https://kubernetes.io/docs/concepts/security/secrets-good-practices/)
- [OWASP CRS 컨테이너](https://coreruleset.org/docs/1-getting-started/1-3-using-containers/)
- [Wazuh Quickstart](https://documentation.wazuh.com/current/quickstart.html)
