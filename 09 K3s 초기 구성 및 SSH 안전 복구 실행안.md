# 09. K3s 초기 구성·SSH 안전 게이트 및 단계별 실행안

> 작성 기준: 2026-10-30 · 담당: Odyz · 보안 협력: BoatingDuck  
> 상태: **설계 초안 / 서버 미적용** · 이 문서의 구성 예시와 명령은 승인 전 실행 지시가 아니다.  
> 연계 이슈: [#9](../../issues/9) · [#10](../../issues/10) · [#13](../../issues/13) · [#19](../../issues/19) · [#24](../../issues/24) · [#25](../../issues/25) · [#34](../../issues/34) · [#35](../../issues/35)

## 1. 변경 승인 원칙과 현재 판정

**현재 결론: K3s 초기 설정안은 제안 가능, 실제 설치·서비스 활성화는 BLOCKED.** 서버는 SSH로만 관리하며 독립적인 물리/원격 콘솔이 확인되지 않았다. 동시 SSH 세션 2개는 네트워크 또는 방화벽 오구성 시의 대체 복구 경로가 아니다. `systemd-run`의 예약 실행과 `/run` 임시 파일 원복 dry-run은 통과했으나, **SSH 장애 시 호스트 네트워크를 원복하는 기능은 검증하지 않았다**. 기존 10개 Docker 컨테이너는 사용자 승인 후 중지됐지만 볼륨·설정은 삭제하지 않았다.

서버 소유자 및 BoatingDuck의 검토, 원복 가능성 및 외부 접속 경계가 확정되기 전에는 `k3s` 서버 시작, `ufw disable`, `iptables-restore`, iptables flush, NetworkManager 재시작, NIC unmanaged 전환, 재부팅, `k3s-killall.sh`, `k3s-uninstall.sh`를 시행하지 않는다.

### 실측 기반선 (2026-10-30)

| 범주 | 사용자 제공 실제 결과 | 한계 |
| --- | --- | --- |
| 호스트 | Ubuntu 22.04.4, N100 4C/4T, 약 15 GiB RAM, 2 GiB swap, 디스크 여유 약 765 GB | 설치 시점 실시간 부하는 재확인 |
| K3s 후보 | stable 채널 리다이렉트 `v1.36.5+k3s1`; AMD64 바이너리 76M; SHA-256 manifest 대조 `k3s: 성공` | 실제 설치/부팅·독립적 배포 서명 확인은 미실행 |
| 네트워크 | 주 NIC `enp1s0` NM-MANAGED=예·연결됨, `docker0` 외부 구성, NetworkManager 활성 | Flannel/CNI 생성 이후 간섭은 미검증 |
| 커널 | IPv4 forwarding=1, overlay/br_netfilter 존재, VXLAN 커널 모듈 파일 존재, cgroup v2, bridge netfilter 설정 확인 | Pod 간 통신은 미검증 |
| UFW | 활성; 기본 incoming deny/routed deny; iptables-nft 1.8.7; Docker 규칙과 다수 기존 브리지 잔존 | Docker/UFW/K3s FORWARD·NAT 동작, NAT/공인망 포워딩 미검증 |
| 포트 | 컨테이너 중지 후 80/443/6443 등 당시 비점유; SSH 22 리스닝 | 실제 적용 직전 재확인 필수 |
| 복구 | SSH 두 연결 확인, UFW/iptables/nftables 로컬 백업 존재, 예약 파일 자동 복원 dry-run PASS | 실제 방화벽·라우트·SSH 장애 복구 및 외부 콘솔 미확보 |

정확한 관리 주소, 계정/토큰, DB 자격증명, kubeconfig, 전체 UFW 덤프를 공개 문서나 이슈에 올리지 않는다. 이전 Docker 환경변수 출력에 자격증명이 포함된 이력이 있으므로 관련 서비스 재가동 전 비밀번호·토큰 교체 필요성을 평가한다.

## 2. 최초 클러스터 후보 구성 (아직 미적용)

| 항목 | 제안 | 이유 / 승인 조건 |
| --- | --- | --- |
| 형태 | 단일 서버 노드, 기본 내장 containerd | 기존 Docker와 이미지·네트워크를 논리적으로 분리; Docker 볼륨 무삭제 |
| K3s 버전 | `v1.36.5+k3s1` 고정 후보 | 설치 직전에도 바이너리/manifest 해시 재확인 |
| 데이터스토어 | 기본 내장 SQLite | 단일 서버에 적합. 프로젝트 PostgreSQL과 **별개**이며 클러스터 데이터 백업 정책 필요 |
| Pod CNI | Flannel VXLAN(기본) | 지원 모듈 존재; 실제 NetworkManager/UFW 호환성은 미검증 |
| Pod/Service CIDR | K3s 기본 `10.42.0.0/16`, `10.43.0.0/16` **후보** | 기존 Docker/VPN/LAN·상위 라우팅 중첩 확인 뒤 결정. 기본 게이트웨이로 route get 된 사실만으로 충돌 부재 판정 금지 |
| 초기 공개 진입 | Traefik 및 ServiceLB **비활성화** | 초기 80/443 공개·포트 점유 방지; 이후 WAF/Ingress 설계 승인 후 별도 활성화 |
| CoreDNS/local-storage/metrics-server | 기본값 유지 후보 | 실기동 후 DNS·스토리지·자원 영향 검증 |
| K3s API | TCP 6443 공개 허용 **추가하지 않음** | 초기에는 현장 호스트 kubectl로만 확인; 추후 관리망/SSH 터널 경로 검토 및 BoatingDuck 승인 |
| Flannel VXLAN | UDP 8472 공인망 허용 **금지** | 단일 노드 외부 개방 불필요 후보. 다중 노드 전환 시 노드 간 최소 범위만 협의 |
| kubeconfig | root 소유/0600 유지 | 사용자 복사·0644 설정 금지; 서비스 계정/RBAC 후속 구성 |
| 호스트 Firewall | 기존 UFW 유지 | 공식 문서의 전체 허용 규칙을 무심코 복사하지 않고 인터페이스·CIDR·방향 기반 최소 정책 합의 |

검토용 `/etc/rancher/k3s/config.yaml` **내용 예시** (Git 상의 제안, 실제 서버에 아직 생성하지 않음):

```yaml
# 제안안: 실제 적용 전에 네트워크 CIDR·관리 경로·보안 승인 필요
flannel-backend: vxlan
disable:
  - traefik
  - servicelb
```

기본 SQLite/containerd를 사용하는 안으로 별도 datastore endpoint, Docker runtime flag, 불필요한 `node-external-ip` 또는 광범위한 `tls-san`을 추가하지 않는다. `cluster-cidr`/`service-cidr`도 확인 전에는 YAML에 고정하지 않는다. 핵심 설정은 장기적으로 systemd 실행 인자보다 이 YAML로 관리한다.

## 3. 실제 변경 전 승인 게이트 (모두 통과해야 함)

- [ ] **G0 소유자/작업 창:** 변경 담당, 승인 시각, 예상 영향, 중지·원복 최종 결정권 및 변경 범위 기록
- [ ] **G1 대체 접근:** SSH가 완전히 끊겨도 서버 네트워크/방화벽을 고칠 수 있는 독립 콘솔 또는 동등하게 검증된 관리 경로 확보. 현재 **없음(BLOCKED)**
- [ ] **G2 백업·실복구:** 원본 상태/파일 백업 확인에 더해 실제 대상별 안전한 복원 절차와 실패 시 확인 수단 마련. `/run` 파일 dry-run만으로 통과 처리 불가
- [ ] **G3 보안 합의:** BoatingDuck이 UFW routed deny·Docker 규칙·Flannel/CNI·6443·8472·80/443 범위를 검토해 최소 허용표 승인. 공인망 6443/8472 허용 금지
- [ ] **G4 네트워크 중첩:** 서버의 실제 모든 interface/route 및 Docker subnet, VPN, 공유기/상위 라우트에서 Pod/Service CIDR 중첩 여부 확인
- [ ] **G5 버전·자원:** 설치 대상 바이너리 해시 재검사, CPU/RAM/디스크·80/443/6443·기존 restart policy/서비스 상태 점검
- [ ] **G6 실행 분리:** 설치(서비스 start/enable 생략)와 네트워크 활성화를 별도 변경 창/증적/승인으로 분리
- [ ] **G7 테스트 및 종료:** Node Ready/Pod DNS/ClusterIP/host SSH/기존 서비스 상태·실제 접근 통제 확인 후 결과 기록

> **핵심:** G1이 미충족 상태에서 타이머 원복만 믿고 호스트 네트워크/방화벽을 변경하는 것을 승인하지 않는다. 동작이 확인된 타이머는 보조 수단이지 원격 콘솔의 대체재가 아니다.

## 4. 단계별 실행 절차 (현재 수행 상태 표기)

| 단계 | 작업 | 조건 | 현재 상태 |
| --- | --- | --- | --- |
| A | 기존 환경·백업·SSH·버전/체크섬 정적 확인 | 읽기 전용·안전성 확인 | **부분 PASS**: 체크섬/백업/예약 파일 복원 PASS, NAT/복구 미완료 |
| B | 설정 YAML·Firewall/접근 매트릭스·원복 절차 PR 검토 | 호스트 변경 없음 | **이 문서 PR에서 제안** |
| C | K3s 바이너리/서비스 파일 설치, 자동 enable/start 생략 | G0~G6 승인 후 별도 진행 | **NOT RUN** |
| D | UFW/Flannel 제약 하의 K3s 서비스 첫 기동 | G1~G6 승인 및 보안 담당 참여 | **NOT RUN** |
| E | Node Ready, CoreDNS·Service/Pod·SSH 회귀, 외부 노출 확인 | D 성공 시 | **NOT RUN** |
| F | Traefik/ServiceLB 또는 별도 Ingress/WAF의 노출 정책 적용 | #34/#35 보안 설계 확정 | **NOT RUN** |

**C 단계 후 보장할 수 없는 것:** `INSTALL_K3S_SKIP_START=true`와 `INSTALL_K3S_SKIP_ENABLE=true`는 K3s 서비스의 즉시 실행과 자동 활성화를 건너뛰지만, 설치 도중 바이너리/유닛/파일 생성 등 호스트 변경 자체를 막지는 못한다. `INSTALL_K3S_VERSION`은 명시적으로 고정하고, 공식 스크립트의 현재 해시·동작을 재검토한 뒤 진행한다. 이미 검증된 바이너리를 재사용할 경우 다운로드 생략/설치 디렉터리 관련 스크립트 옵션을 별도로 확인한다. **지금은 실제 설치 명령을 제공하거나 실행하지 않는다.**

## 5. SSH·네트워크 장애 대응 정책 초안

**유지해야 하는 것:** `sshd` 설정·22/TCP 관리 경로·주 NIC `enp1s0`·NetworkManager 현재 연결·기존 Docker 브리지/볼륨·기존 UFW 기본 정책·SSH 세션. `nm-cloud-setup` service/timer는 현재 systemd에서 not-found로 확인돼 비활성화 작업이 필요 없다.

**방어적인 실행 방식**
1. 변경 전후 SSH 접속을 *새 터미널에서* 검증하고 기존 터미널 유지. 두 세션이 존재해도 경로 장애가 공통으로 발생할 수 있음을 명시.
2. 명령은 포트/규칙 단위로 최소화하고, 담당자가 변경 전후 diff를 승인. 공개 허용표 작성 전 `ufw allow 6443` 같은 전체 허용 금지.
3. 예약 systemd 파일 복구 시험은 PASS이지만, 실제 UFW/iptables 네트워크 복원에는 별도 위험이 있다. 예약 동작 로그·출력 및 **네트워크 접근과 독립적인 확인 수단**까지 증명하기 전 자동 복구로 인정하지 않는다.
4. 문제가 생기면 사전 정한 독립 접근 경로에서 SSH/주 NIC 상태를 확인하고, 이번 변경으로 추가된 부분만 되돌린다. 과거의 `iptables-save`나 `nft list ruleset`을 **무조건 통째로 복원하지 않는다**(Docker 등 현재 규칙과 충돌 가능).
5. `systemctl stop k3s`만으로 Pod/네트워크 규칙이 모두 정리되는 것은 아니다. K3s의 `k3s-killall.sh`는 iptables/네트워크를 지우며 다른 Docker 서비스에 영향을 줄 수 있으므로 자동 원복 경로에 넣지 않는다. `k3s-uninstall.sh`는 클러스터 데이터/PV 데이터 삭제 위험으로 금지한다.
6. SSH와 별도 콘솔이 모두 없어서 시스템 연결이 끊기면 복구 실행 여부를 실측할 수 없다. 이를 '복구 보장'으로 표시하지 않으며, 무리한 변경 대신 설계·검증 가능한 오프라인 작업을 지속한다.

## 6. BoatingDuck과 최종 합의할 통신 행렬

| 통신 | 후보 방향 | 현재 결정 |
| --- | --- | --- |
| 관리자 → Ubuntu SSH 22/TCP | 기존 승인된 관리 경로 유지 | **유지**, 범위 축소는 대체 접근 확보 뒤 |
| 관리자 → K3s API 6443/TCP | 초기에는 호스트 내부 사용 우선; 필요한 경우 관리망/터널 한정 | **공개 허용 없음** |
| K3s 노드 간 VXLAN 8472/UDP | 단일 노드 초기엔 외부 허용 불필요 후보 | **공인망 차단**, 실제 CNI 통신 확인 필요 |
| K3s Pod/Service 내부 통신 | 승인된 CIDR/인터페이스·DNS/Service 경로 | UFW routed deny와 실제 네트워크 분석 후 결정 |
| 인터넷 → 80/443 | Traefik·ServiceLB 초기 비활성; WAF 경유 설계 후만 공개 | 초기 **신규 공개 없음** |
| Kubernetes API·DB·보안 관리 포트 | 직접 외부 공개하지 않음 | 네트워크 접근 제한/우회 검증 설계 대기 |

BoatingDuck이 [#34](../../issues/34)에서 UFW와 Flannel 호환성을, [#35](../../issues/35)에서 실제 통신 경로를 담당한다. K3s 실행 후에만 알 수 있는 상세 CNI 경로는 **사전 설계와 사후 실측을 분리**하고, 사전 승인 없이 방화벽 전체를 비활성화해서 통신을 맞추지 않는다.

## 7. 안전한 재점검 명령 (읽기 전용; **실행해도 설정 변경 없음**)

민감정보는 로컬에서 확인하고, GitHub 공유 전 IP/계정/토큰을 가린다.

```bash
# 현재 SSH 및 관리 인터페이스
sudo sshd -t && systemctl is-active ssh
nmcli -f GENERAL.DEVICE,GENERAL.STATE,GENERAL.NM-MANAGED device show enp1s0

# 현재 핵심 포트 및 실행 중인 기존 컨테이너
sudo ss -lntup
docker ps --format '{{.Names}} {{.Status}} {{.Ports}}'

# 네트워크 중복 검토를 위한 현재 라우팅·Docker 네트워크 목록
ip -4 route show table main
docker network ls

# 방화벽 및 K3s 활성화 전 상태
sudo ufw status verbose
systemctl is-enabled k3s 2>&1 || true
systemctl is-active k3s 2>&1 || true
```

`docker network inspect`의 자세한 출력에는 개인 설정/주소 정보가 들어갈 수 있으므로 전체 결과를 공용 이슈에 복사하지 않는다. 포트 스캔/외부 접속 시험은 승인된 접속 지점에서만 수행한다.

## 8. 완료·보류 판정

- [x] 설치 후보 바이너리 SHA-256 manifest 대조 PASS (76M AMD64 `k3s`)
- [x] `systemd-run` 예약 생성·실행 및 `/run` 파일 자동 복원 dry-run PASS
- [x] 호스트·UFW·SSH·Docker 기준선 조사 및 로컬 백업 기록
- [ ] 변경 승인과 **SSH와 독립적인 실효 복구 경로**
- [ ] Pod/Service CIDR·관리 포트·Flannel/UFW/NetworkManager 호환성 설계 승인
- [ ] 실제 K3s 설치·시작·기능 검증 및 복구 시험

**최종 판정: PLANNING / PARTIAL PASS / ACTIVATION BLOCKED.**

## 9. 공식 근거

- K3s 설치 환경변수 및 start/enable 생략: https://docs.k3s.io/reference/env-variables
- K3s 설정 파일 로딩: https://docs.k3s.io/installation/configuration
- K3s 네트워크 요구사항/UFW/VXLAN 보안: https://docs.k3s.io/installation/requirements
- Flannel 기본 VXLAN: https://docs.k3s.io/networking/basic-network-options
- Traefik·ServiceLB 노출/비활성화: https://docs.k3s.io/networking/networking-services
- 기본 SQLite 데이터스토어: https://docs.k3s.io/datastore
- K3s stop과 killall 차이: https://docs.k3s.io/upgrades/killall
- K3s 제거 시 데이터 삭제 주의: https://docs.k3s.io/installation/uninstall
