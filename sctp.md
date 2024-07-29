## Using the Stream Control Transmission Protocol (SCTP) on a bare metal cluster

### 1. OpenShift에서 SCTP (Stream Control Transmission)

클러스터 관리자는 클러스터의 호스트에서 SCTP를 활성화 할 수 있습니다. RHCOS(Red Hat Enterprise Linux CoreOS)에서 SCTP 모듈은 기본적으로 비활성화되어 있습니다.



SCTP는 IP 네트워크에서 실행되는 안정적인 메시지 기반 프로토콜입니다.



활성화하면 Pod, 서비스, 네트워크 정책에서 SCTP를 프로토콜로 사용할 수 있습니다. `type` 매개변수를 `ClusterIP` 또는 `NodePort` 값으로 설정하여 `Service`를 정의해야 합니다.



**1) SCTP 프로토콜을 사용하는 구성의 예**

`protocol` 매개변수 Pod 또는 서비스 오브젝트의 `SCTP` 값으로 설정하여 SCTP를 사용하도록 Pod 또는 서비스를 구성할 수 있습니다.

- Pod가 SCTP를 사용하도록 구성

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    namespace: project1
    name: example-pod
  spec:
    containers:
      - name: example-pod
  ...
        ports:
          - containerPort: 30100
            name: sctpserver
            protocol: SCTP
  ```

- 서비스가 SCTP를 사용하도록 구성

  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
    namespace: project1
    name: sctpserver
  spec:
  ...
    ports:
      - name: sctpserver
        protocol: SCTP
        port: 30100
        targetPort: 30100
    type: ClusterIP
  ```

- `NetworkPolicy` 오브젝트는 특정 레이블이 있는 모든 Pod의 포트 `80`에서 SCTP 네트워크 트래픽에 적용되도록 구성되어 있습니다.

  ```yaml
  kind: NetworkPolicy
  apiVersion: networking.k8s.io/v1
  metadata:
    name: allow-sctp-on-http
  spec:
    podSelector:
      matchLabels:
        role: web
    ingress:
    - ports:
      - protocol: SCTP
        port: 80
  ```

  

### 2. SCTP  활성화

- 다음 YAML 정의가 포함된 `load-sctp-module.yaml` 파일을 생성합니다.

  ```yaml
  cat <<EOF>> load-sctp-module.yaml
  
  apiVersion: machineconfiguration.openshift.io/v1
  kind: MachineConfig
  metadata:
    name: load-sctp-module
    labels:
      machineconfiguration.openshift.io/role: worker
  spec:
    config:
      ignition:
        version: 3.2.0
      storage:
        files:
          - path: /etc/modprobe.d/sctp-blacklist.conf
            mode: 0644
            overwrite: true
            contents:
              source: data:,
          - path: /etc/modules-load.d/sctp-load.conf
            mode: 0644
            overwrite: true
            contents:
              source: data:,sctp
  EOF
  ```

- `MachineConfig` 오브젝트를 생성하려면 다음 명령을 입력합니다.

  ```bash
  $ oc create -f load-sctp-module.yaml
  ```

- 선택 사항 : Machine Config Operator가 구성 변경 사항을 적용하는 동안 노드의 상태를 보려면 다음 명령을 입력합니다. 노드 상태가 `Ready`로 전환되면 구성 업데이트가 적용됩니다.

  ```bash
  $ oc get nodes
  ```



### 3. SCTP 활성화 여부 확인 및 서비스 배포

- 프로젝트 생성

  ```bash
  $ oc new-project sctp-demo
  ```

  SCTP 리스너를 시작하는 Pod를 생성합니다.

- 다음 YAML로 pod를 정의하는 `sctp-server.yaml` 파일을 생성합니다.

  ```yaml
cat <<EOF>> sctp-server.yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: sctpserver
    labels:
      app: sctpserver
  spec:
    containers:
      - name: sctpserver
        image: registry.access.redhat.com/ubi8/ubi
        command: ["/bin/sh", "-c"]
        args:
          ["dnf install -y nc && sleep inf"]
        ports:
          - containerPort: 30102
            name: sctpserver
            protocol: SCTP
  EOF
  ```
  
- 다음 명령을 입력하여 pod를 생성합니다.

  ```bash
  $ oc create -f sctp-server.yaml
  ```

- 리스너에 대한 서비스를 생성합니다.

  다음 YAML을 사용하여 서비스를 정의하는 `sctp-service.yaml` 파일을 생성합니다.

  ```yaml
  cat <<EOF>> sctp-service.yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: sctpservice
    labels:
      app: sctpserver
  spec:
    type: NodePort
    selector:
      app: sctpserver
    ports:
      - name: sctpserver
        protocol: SCTP
        port: 30102
        targetPort: 30102
  EOF
  ```

- 서비스 생성 명령을 입력합니다.

  ```bash
  $ oc create -f sctp-service.yaml
  ```

- SCTP 클라이언트에 대한 pod를 생성합니다.

  다음 YAML을 사용하여 `sctp-client.yaml` 파일을 생성합니다.

  ```yaml
  cat <<EOF>> sctp-client.yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: sctpclient
    labels:
      app: sctpclient
  spec:
    containers:
      - name: sctpclient
        image: registry.access.redhat.com/ubi8/ubi
        command: ["/bin/sh", "-c"]
        args:
          ["dnf install -y nc && sleep inf"]
  EOF
  ```

- `Pod` 오브젝트를 생성합니다.

  ```bash
  $ oc apply -f sctp-client.yaml
  ```

- 서버에서 SCTP 리스너를 실행합니다.

  - 서버 Pod에 연결하려면 다음 명령을 입력합니다.

    ```bash
    $ oc rsh sctpserver
    ```

  - SCTP 리스너를 시작하려면 다음 명령을 입력합니다.

    ```bash
    $ nc -l 30102 --sctp 
    ```

- 서버의 SCTP 리스너에 연결합니다.

  - 터미널 창에서 다음을 실행합니다.

  - `sctpservice` 서비스의 IP 주소를 얻습니다.

    ```bash
    $ oc get services sctpservice -o go-template='{{.spec.clusterIP}}{{"\n"}}'
    ```

    또는 

    ```bash
    $ oc get svc -wide
    ```

  - 클라이언트 Pod에 연결하려면 다음 명령을 입력합니다.

    ```bash
    $ oc rsh sctpclient
    ```

  - SCTP 클라이언트를 시작하려면 다음 명령을 입력합니다. `<cluster_IP>`를  `sctpservice` 서비스의 클러스터 IP 주소로 변경합니다.

    ```bash
    # nc <cluster_IP> 30102 --sctp 
    ```

- 테스트 확인

  - sctpclinet에서 메시지 전송 확인

    

### 4. 네트워크 폴리시 설정

- `istio-system` 프로젝트의 80 Port만 SCTP 프로토콜로 허용하는 네트워크 정책 추가

  - 외부의 모든 프로젝트에서 들어오는 기본적인 트래픽 거부 정책 추가

    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata:
      name: sctp-deny-other-namespaces
      spec:
        podSelector: null
        ingress:
          - from:
              - podSelector: {}
    ```

  - `istio-system`의 `ingressgateway`에서 들어오는 80 Port에 대해서 SCTP 통신 허용 정책 추가

    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata:
      name: sctp-demo-allow
    spec:
      podSelector:
        matchLabels:
          app: sctpserver
      ingress:
        - ports:
            - protocol: SCTP
              port: 80
          from:
            - namespaceSelector:
                matchLabels:
                  kubernetes.io/metadata.name: istio-system
            - podSelector:
                matchLabels:
                  app: istio-ingressgateway
    ```

- 확인 방법

  - `istio-system` 프로젝트의 `ingressgateway` 서비스 IP 포트 확인
- 
  
  sctp-server pod 접속 -> nc 명령어로 포트 확인
  
  ```bash
  sh-4.4# nc -v -z 172.30.236.155 8080
  Ncat: Version 7.70 ( https://nmap.org/ncat )
Ncat: Connected to 172.30.236.155:8080.
  Ncat: 0 bytes sent, 0 bytes received in 0.01 seconds.
  ```
```
  
  8080으로 들어오는 port에 대해 sctp 통신을 허용
  
  ```bash
  sh-4.4# nc -v -z 172.30.25.20 8081
Ncat: Version 7.70 ( https://nmap.org/ncat )
  Ncat: Connection refused.
```

  8081로 들어오는 포트에 대해서는 sctp 통신을 허용하지 않음
