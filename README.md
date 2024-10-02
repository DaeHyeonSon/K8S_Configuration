Docker 이미지를 Kubernetes를 통한 배포를 통해 구성하는 실습을 진행한다.

### 개요 🚩
<div align ="center">
  <img src ="https://github.com/user-attachments/assets/430474a7-d117-4f04-9a04-7226835337b5" width = 60%>
</div>

Spring Boot로 작성된 SpringApp을 그림과 같이 Kubernetes 클러스터에 배포하는 과정으로써 minikube를 사용하여 로컬에서 Kubernetes 클러스터를 실행하고 SpringApp을 배포하는 방법을 다뤄본다.


### Kubernetes 클러스터 및 Minikube 설정 😮

```bash
# Minikube 클러스터를 시작하려면 아래 명령어를 실행
minikube start

# Minikube가 Docker 데몬을 사용하도록 설정
eval $(minikube -p minikube docker-env)
```

<br>

<details>
  <summary>파일 구조 확인</summary>
  <div align ="center">
  <img src ="https://github.com/user-attachments/assets/1b5c832d-c6a9-49c4-93ec-8b5be63d3956" width = 40%>
</div>
</details>


### Docker 이미지 빌드 및 Minikube 설정 ⚙

<br>

**Minikube 환경 설정**
Minikube 내부에서 Docker 데몬을 사용하기 위해 다음 명령어를 실행한다.

```bash
eval $(minikube docker-env)
```

**Docker 이미지 빌드**
프로젝트의 디렉토리에서 Dockerfile을 사용하여 이미지를 빌드한다.

```bash
docker build -t springapp:latest .
```

빌드가 성공하면 Docker 이미지를 확인할 수 있습니다.

```bash
docker images
```

<br>

### Kubernetes 설정 파일 🛠
Kubernetes 배포와 서비스 설정을 위해 두 개의 YAML 파일을 작성한다. <br>
`이때 주의`할 점은 port 번호를 `SpringApp-0.0.1-SNAPSHOT.jar`의 포트번호와 일치되게 구성한다.

<div align ="center">
  <img src ="https://github.com/user-attachments/assets/6d8a9b6d-acae-46a0-abcf-4597ac194899" width = 40%>
</div>

<br>

`deployment.yml` 작성

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: springapp-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: springapp
  template:
    metadata:
      labels:
        app: springapp
    spec:
      containers:
      - name: springapp
        image: springapp:latest
        resources:
          limits:
            memory: "128Mi"
            cpu: "500m" 
        ports:
        - containerPort: 8899 # 컨테이너에서 사용하는 ContainerPort와 일치되게끔 설정
```

`service.yml` 작성

```yaml
apiVersion: v1
kind: Service
metadata:
  name: springapp-service
spec:
  type: NodePort # 
  selector:
    app: springapp
  ports:
  - protocol: TCP
    port: 8899
    targetPort: 8899
```

후에 yaml 파일을 Minikube 클러스터에 배포한다.

```bash
kubectl apply -f deployment.yml
kubectl apply -f service.yml
```
<br> 

배포 확인 결과 다음과 같다.

<div align ="center">
  <img src ="https://github.com/user-attachments/assets/7beab6ba-5a1e-4083-90f8-c350bce3869c" width = 50%>
</div>

<br>

테스트를 위해 `kubectl get service`로 확인한 포트번호를 포트포워딩 해준다. 

<div align ="center">
  <img src ="https://github.com/user-attachments/assets/0194a9f5-5a55-4a7e-ab6c-b82291e8ce15" width = 50%>
</div>

<br>

올바르게 실행되었는지 `curl` 명령어 및 `ip`에 접속해 확인한다. 

<div align ="center">
  <img src ="https://github.com/user-attachments/assets/c0c1c302-2901-4f17-9e1f-e6cf37cc8bbd" width = 50%>
</div>

정상적으로 작동하는 것을 확인할 수 있다. 

<br>

`minikube dashboard`를 통해서도 확인 가능하다.

<div align ="center">
  <img src ="https://github.com/user-attachments/assets/19ebd458-d489-44b0-825f-5626cd4cb997" width = 50%>
</div>

<br>

### Trouble Shooting 🔥
위 과정을 진행하면서 겪은 사소한 이슈들에 대해 언급하고자 한다.

먼저 `springapp-service`와 연결된 `pod`가 정상적으로 실행되지 않아서 문제 발생하는것을 확인하였다.

<div align ="center">
  <img src ="https://github.com/user-attachments/assets/567e061d-97ba-4774-a01f-d05887a2a1f9" width = 50%>
</div>

`pod`의 상태 확인을 위해 확인 결과 

<div align ="center">
  <img src ="https://github.com/user-attachments/assets/f772b0ef-634e-4b06-8de0-2236babab49f" width = 50%>
</div>

<br>

각각의 상태가 `ImageBackPullOff`, `ErrorImagePull`, `Pending` 상태인 것을 확인할 수 있다.

이는 먼저 `ImageBackPullOff`, `ErrorImagePull` 문제의 경우 deployment.yml 파일에 `imagePullPolicy`를 `Never`로 설정해주었다. 로컬 Minikube 환경에서 이미지를 직접 사용할 때는 Kubernetes가 외부 레지스트리에서 이미지를 가져오려고 시도하지 않도록 `imagePullPolicy`를 `Never`로 설정해줌으로써 이슈가 해결되었다.

<br>

수정된 deployment.yml은 다음과 같다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: springapp-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: springapp
  template:
    metadata:
      labels:
        app: springapp
    spec:
      containers:
      - name: springapp
        image: springapp:latest
        imagePullPolicy: Never # 외부 레지스트리에서 이미지를 가져오지 않도록 설정
        ports:
        - containerPort: 8080
```

위 두 문제는 해결하였지만 아직 그림과 같은 `Pending` 문제가 여전히 존재하였다. 

<div align ="center">
  <img src ="https://github.com/user-attachments/assets/4569dffb-9184-436a-b27f-aa09e9cd203c" width = 50%>
</div>

<br>

이를 해결하기 위해 먼저 `describe` 명령어를 통해 문제 확인 하였다. 

```bash
username@servername:~/springapp-k8s-deployment$ kubectl describe pod/springapp-deployment-57f5cc75fb-5pdtj
==================================================================
Name:             springapp-deployment-57f5cc75fb-5pdtj
Namespace:        default
Priority:         0
Service Account:  default
Node:             <none>
Labels:           app=springapp
                  pod-template-hash=57f5cc75fb
Annotations:      <none>
Status:           Pending
IP:               
IPs:              <none>
Controlled By:    ReplicaSet/springapp-deployment-57f5cc75fb
Containers:
  springapp:
    Image:      springapp:latest
    Port:       8080/TCP
    Host Port:  0/TCP
    Limits:
      cpu:     500m
      memory:  128Mi
    Requests:
      cpu:        500m
      memory:     128Mi
    Environment:  <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-pnlfr (ro)
Conditions:
  Type           Status
  PodScheduled   False 
Volumes:
  kube-api-access-pnlfr:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   Guaranteed
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  FailedScheduling  36s   default-scheduler  0/1 nodes are available: 1 Insufficient cpu. preemption: 0/1 nodes are available: 1 No preemption victims found for incoming pod.
```

확인 결과 CPU 리소스가 부족으로 인해 로그 메시지에서 `Insufficient cpu`라는 경고가 발생하는 것을 확인할 수 있었다.

<br>

더 적은 리소스를 요청하도록 설정하면 Pod가 스케줄링될 가능성이 높아지는 것을 확인하였다.

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: springapp-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: springapp
  template:
    metadata:
      labels:
        app: springapp
    spec:
      containers:
      - name: springapp
        image: springapp:latest
        imagePullPolicy: Never # 외부 레지스트리에서 이미지를 가져오지 않도록 설정
        resources:
          limits:
            memory: "128Mi"
            cpu: "200m"  # CPU 리미트 줄이기
          requests:
            memory: "64Mi"
            cpu: "200m"  # CPU 요청 줄이기
        ports:
        - containerPort: 8080
```
`resources` 부분에 `cpu limit`를 기존 500m에서 200m으로 하향 조정해주었다.  

<div align ="center">
  <img src ="https://github.com/user-attachments/assets/fe2d8898-fa96-450e-92fb-4fc01c9095bb" width = 50%>
</div>

<div align ="center">
  <img src ="https://github.com/user-attachments/assets/086634fb-7e9d-4670-83df-e65b61ccad6c" width = 50%>
</div>

해당 yaml 파일을 적용한 뒤 확인 결과 정상적으로 `Running` 되는 것을 확인 할 수 있다.


### 결론 📖
Docker로 빌드된 SpringApp을 Kubernetes 클러스터에 배포하고 `NordPort`를 통해 외부에서 접근하는 실습을 진행하였다. 이를 통해 로컬 Kubernetes 환경에서 애플리케이션을 배포하고 관리하는 방법을 익힐 수 있었다. 
추후 예정사항으로 minikube를 사용한 로드밸런싱 실습을 추후에 진행해볼 예정이다.

