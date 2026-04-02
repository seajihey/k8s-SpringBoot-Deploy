# N-tier 구조의 Kubernetes 클러스터 구축 및 SpringBoot와 MySQL 배포
<br>

### 📍VirtualBox 기반 멀티 노드 K8s 클러스터 구축부터 애플리케이션 배포 실습입니다.

<br>

Spring Boot 애플리케이션과 MySQL을 Kubernetes 환경에서 운영하기 위해, **3개의 VM** 으로 멀티 노드 클러스터를 구축하고,  
**Secret, ConfigMap**을 활용한 보안 구조를 적용하여 배포까지 완성하는 실습입니다.

k8s 클러스터 구축 실습에서 더 나아가, **브리지 네트워크, CNI, Pod 스케줄링, Service 타입** 등 k8s의 핵심 개념을 실제로 트러블슈팅하며 체득하는 데 목적이 있습니다.



<br>

---

## 📌 실습 목적

<br>

- **VirtualBox + Ubuntu 24.04** 기반으로 k8s 멀티 노드 클러스터 구축
- **kubeadm, kubelet, kubectl, containerd** 를 활용한 수동 설치 경험
- **Secret, ConfigMap** 을 활용한 보안 설정 분리 구조 이해
- **Pod, Service, Deployment** 의 동작 원리와 상호관계 파악
- 트러블슈팅을 통한 K8s 운영 감각 체득

<br>

---

## 📌 전체 아키텍처

```
[호스트 PC]
    │
    ├── VirtualBox
    │     ├── server01 (마스터 노드) - 10.0.2.15 (-p 2015)
    │     ├── server02 (워커 노드)  - 10.0.2.20  (-p 2020)
    │     └── server03 (워커 노드)  - 10.0.2.25  (-p 2025)
    │
    └── K8s 클러스터
          ├── Secret          ← DB 비밀번호 암호화
          ├── ConfigMap       ← DB URL, 포트 설정
          ├── MySQL Pod       ← PersistentVolume 연결
          ├── MySQL Service   ← ClusterIP:3306
          ├── SpringBoot Pod  ← Secret, ConfigMap 주입
          └── SpringBoot Service ← NodePort:30083
```

<br>

### 핵심 개념 정리

| 개념 | 설명 |
|------|------|
| **Pod** | 컨테이너가 실행되는 최소 단위. K8s가 워커 노드에 자동 배치 |
| **Deployment** | Pod를 관리하는 단위. 재시작, 복제, 업데이트 담당 |
| **Service** | Pod에 접근하는 창구. Pod IP가 바뀌어도 안정적으로 연결 |
| **ClusterIP** | 클러스터 내부에서만 접근 가능한 Service 타입 (MySQL 등) |
| **NodePort** | 외부에서 접근 가능한 Service 타입 (SpringBoot 등) |
| **ConfigMap** | DB URL 등 설정값을 코드와 분리하여 관리 |
| **PersistentVolume** | Pod가 재시작되어도 데이터가 유지되도록 하는 스토리지 |
| **CNI** | 노드 간 Pod 통신을 가능하게 하는 네트워크 플러그인 (Calico 사용) |

<br>

---

## 📌 환경 구성

### 노드 스펙

| 노드 | 역할 | IP | 메모리 | CPU |
|------|------|-----|--------|-----|
| server01 | 마스터 (control-plane) | 10.0.2.15 | 6GB | 4코어 |
| server02 | 워커 노드 | 10.0.2.20 | 4GB | 4코어 |
| server03 | 워커 노드 | 10.0.2.25 | 4GB | 4코어 |

### 설치 패키지

| 패키지 | 역할 |
|--------|------|
| containerd | 컨테이너 런타임 |
| kubeadm | 클러스터 초기화 및 노드 join 도구 |
| kubelet | 각 노드에서 Pod를 실행하는 에이전트 |
| kubectl | 클러스터 제어 CLI 도구 |
| Calico | CNI 플러그인 (Pod 간 네트워크 통신) |

<br>

---

## 📌 k8s설치 과정

### 1️⃣ 1단계 — 모든 노드 공통 설정 (네트워크 커널 설정)

```bash
sudo -i

# 커널 모듈 등록 (부팅 시 자동 로드)
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

# 즉시 적용
modprobe overlay
modprobe br_netfilter

# 네트워크 커널 파라미터 설정
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF

sysctl --system
```

> **왜 ?**  
> - `overlay`: 서로 다른 노드의 Pod가 같은 네트워크처럼 통신하게 해줌  
> - `br_netfilter`: 컨테이너 트래픽에 iptables 규칙 적용  
> - `ip_forward`: Pod와 외부 네트워크 간 패킷 전달 허용

<br>

### 2️⃣ 2단계 — containerd 설치 및 설정

```bash
sudo apt update && sudo apt install -y containerd
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml > /dev/null

# SystemdCgroup = true 로 변경 (K8s와 자원 관리 방식 통일)
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
sudo systemctl restart containerd
```

<br>

### 3️⃣ 3단계 — kubeadm, kubelet, kubectl 설치

```bash
sudo apt-get install -y apt-transport-https ca-certificates
sudo mkdir -p /etc/apt/keyrings

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
```

<br>

### 4️⃣ 4단계 — 마스터 노드 초기화 (server01만)

```bash
# 이미지 사전 pull
sudo kubeadm config images pull --cri-socket unix:///run/containerd/containerd.sock

# 클러스터 초기화
sudo kubeadm init \
  --apiserver-advertise-address=10.0.2.15 \
  --pod-network-cidr=192.168.0.0/16 \
  --cri-socket /run/containerd/containerd.sock

# kubectl 설정
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

<br>

### 5️⃣ 5단계 — Calico CNI 설치 (마스터 노드에서)

```bash
curl -O https://raw.githubusercontent.com/projectcalico/calico/v3.27.3/manifests/calico.yaml
kubectl apply -f calico.yaml --validate=false
kubectl get pods -n kube-system
```

<br>

### 6️⃣ 6단계 — 워커 노드 join (server02, server03)

```bash
sudo kubeadm join 10.0.2.15:6443 --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

> 토큰 분실 시 마스터에서 재발급:
> ```bash
> sudo kubeadm token create --print-join-command
> ```

<br>

### 7️⃣ 7단계 — 클러스터 확인

```bash
kubectl get nodes
```

```
NAME       STATUS   ROLES           AGE   VERSION
server01   Ready    control-plane   17h   v1.30.14
server02   Ready    <none>          59m   v1.30.14
server03   Ready    <none>          59m   v1.30.14
```

<br>

---

## 📌 SpringBoot + MySQL 배포

### 배포 구조

```
Secret (DB 비밀번호)  ConfigMap (DB URL)
        ↓                    ↓
   SpringBoot Pod ──────────────→ MySQL Service (ClusterIP)
        ↓                                  ↓
SpringBoot Service                     MySQL Pod
  (NodePort:30083)                  (PersistentVolume)
        ↑
      사용자
```

<br>

### application.properties 설정 (환경변수 분리)

```properties
spring.datasource.url=${SPRING_DATASOURCE_URL}
spring.datasource.username=${DB_USERNAME}
spring.datasource.password=${DB_PASSWORD}
```

-  비밀번호를 코드에 직접 넣지 않고 K8s Secret에서 환경변수로 주입받는 구조

<br>

### Secret (비밀번호 암호화)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
type: Opaque
stringData:
  db-username: user01
  db-password: user01
```

<br>

### ConfigMap (설정 분리)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-config
data:
  db-url: jdbc:mysql://mysqldb:3306/fisa?serverTimezone=Asia/Seoul&useSSL=false&allowPublicKeyRetrieval=true
  db-name: fisa
```

<br>

### MySQL Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: db-password
        - name: MYSQL_DATABASE
          valueFrom:
            configMapKeyRef:
              name: mysql-config
              key: db-name
        - name: MYSQL_USER
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: db-username
        - name: MYSQL_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: db-password
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: mysql-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-storage
        persistentVolumeClaim:
          claimName: mysql-pvc
```

<br>

### 이미지 빌드 및 전송 (Docker → containerd)

호스트에서 Docker로 이미지를 빌드하고, containerd가 설치된 VM으로 전송하는 방식을 사용했습니다.  
VM에는 Docker가 없어도 containerd가 동일한 이미지 형식을 지원하므로 가능합니다.

```bash
# 호스트에서
docker build -t springboot-emp:1.0 .
docker save springboot-emp:1.0 -o springboot-emp.tar
scp springboot-emp.tar ubuntu@server01:~/

# VM(server01, server02, server03) 각각에서
sudo ctr -n k8s.io images import springboot-emp.tar
```

> **왜 k8s.io 네임스페이스?**  
> K8s는 containerd의 k8s.io 네임스페이스에서 이미지를 찾기 때문에, -n k8s.io 옵션으로 import 해야 합니다.

<br>

### 배포 결과 확인

```bash
kubectl get pods -o wide
```

<img width="727" height="65" alt="image" src="https://github.com/user-attachments/assets/9c15acff-fa99-40d8-98a5-d9c03f3e7a01" />


```bash
curl http://10.0.2.20:30083/emp/alldept
```

```json
[
  {"deptno":10,"dname":"ACCOUNTING","loc":"NEW YORK"},
  {"deptno":20,"dname":"RESEARCH","loc":"DALLAS"},
  {"deptno":30,"dname":"SALES","loc":"CHICAGO"},
  {"deptno":40,"dname":"OPERATIONS","loc":"BOSTON"}
]
```
<img width="732" height="40" alt="image" src="https://github.com/user-attachments/assets/236c8a94-7ded-4e38-9a9c-bd7ca91304cf" />

<br>

---

## 🔧 트러블슈팅

### 1. API 서버 연결 거부 (`connection refused`)

**원인**: 메모리 부족(2GB)으로 kubelet이 멈춤  
**해결**: VM 메모리를 6GB로 증설 후 kubelet 재시작

```bash
sudo systemctl restart kubelet
```

### 2. `ErrImageNeverPull`

**원인**: `imagePullPolicy: Never` 설정 시 이미지가 해당 노드에 없으면 발생  
**해결**: 모든 워커 노드에 이미지 import

```bash
sudo ctr -n k8s.io images import springboot-emp.tar
```

### 3. Calico Pod `Pending` 상태

**원인**: 마스터 노드에 NoSchedule taint가 걸려있어 CNI Pod가 못 뜸  
**해결**: taint 제거

```bash
kubectl taint nodes server01 node-role.kubernetes.io/control-plane:NoSchedule-
```

### 4. 워커 노드 `NotReady` (CNI not initialized)

**원인**: containerd 재시작 후 CNI 플러그인 초기화 안 됨  
**해결**: containerd, kubelet 순서대로 재시작

```bash
sudo systemctl restart containerd
sudo systemctl restart kubelet
```

### 5. `kubeadm join` 재시도 시 충돌

**원인**: 이전 join 시도 흔적이 남아있음  
**해결**: reset 후 재시도

```bash
sudo kubeadm reset -f
sudo rm -rf /etc/cni/net.d /var/lib/kubelet/*
sudo rm -f /etc/kubernetes/kubelet.conf /etc/kubernetes/pki/ca.crt
sudo systemctl restart kubelet
sudo kubeadm join ...
```

<br>

---

## 📊 최종 결과

| 항목 | 결과 |
|------|------|
| 클러스터 구성 | server01(마스터) + server02, server03(워커) |
| K8s 버전 | v1.30.14 |
| 컨테이너 런타임 | containerd 1.7.28 |
| CNI | Calico v3.27.3 |
| 배포 앱 | SpringBoot (포트 30083) + MySQL 8.0 |
| 보안 설정 | Secret(비밀번호), ConfigMap(설정) 분리 |
| 데이터 확인 | `/emp/alldept` API 정상 응답 |

<br>

---

## 📊 Insight

1. **메모리가 가장 중요한 변수**였습니다. 2GB 환경에서 API 서버, etcd, Calico가 동시에 뜨면서 OOM으로 인해 컴포넌트가 계속 재시작되는 문제가 발생했습니다. K8s 마스터 노드는 최소 4GB, 권장 6GB 이상이 필요합니다.

2. **containerd 네임스페이스** 개념을 이해하는 것이 핵심이었습니다. `ctr images import`와 `ctr -n k8s.io images import`는 전혀 다른 결과를 만들어내며, K8s는 반드시 `k8s.io` 네임스페이스의 이미지를 참조합니다.

3. **Secret과 ConfigMap 분리**는 단순한 보안 설정이 아니라, 환경(개발/운영)이 바뀔 때 코드 수정 없이 설정만 교체할 수 있는 실무 구조의 핵심입니다.

4. **Pod는 어느 노드에 뜰지 모릅니다.** 마스터가 스케줄러를 통해 자동 결정하므로, 이미지는 모든 워커 노드에 미리 준비되어 있어야 합니다.



<br>


---

## 🔜 추후 계획
 
### Ingress 적용으로 외부 접근 구조 개선
 
현재는 NodePort(30083)를 통해 외부에서 접근하는 구조입니다.  이 방식은 IP:포트 직접 접근이 필요하고, 접근하기 어렵습니다.
 
**목표**: Ingress Controller를 도입하여 도메인 기반으로 외부 접근이 가능하도록 개선
 
```
현재 구조
사용자 → http://10.0.2.20:30083/emp
 
목표 구조
사용자 → http://emp.local/emp → Ingress(nginx) → Service → Pod
```
 
**적용 예정 내용**
 
| 단계 | 내용 |
|------|------|
| Ingress Controller 설치 | nginx 기반 Ingress Controller 배포 |
| ClusterIP로 Service 변경 | 외부 직접 노출 대신 Ingress를 통해서만 접근 |
| 도메인 기반 라우팅 적용 | `emp.local` 도메인으로 SpringBoot 접근 |
| 경로 기반 라우팅 | `/emp`, `/api` 등 경로별 서비스 분기 |
 
<br><br>
