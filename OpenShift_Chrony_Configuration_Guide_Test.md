## Customizing nodes

### 1. Installing Butane

- 파일 다운로드

  ```bash
  wget https://mirror.openshift.com/pub/openshift-v4/clients/butane/latest/butane-amd64
  ```

- 파일 위치 확인

  ```bash
  cd /root/files/butane  
  ```

- 권한 부여 및 파일 복사

  ```bash
  mv butane-amd64 butane
  chmod 755 butane
  cp butane /usr/local/bin
  ```

- 확인

  ```bash
  # butane -h   
  Usage: butane [options] [input-file]
  Options:
    -d, --files-dir string   allow embedding local files from this directory
    -h, --help               show usage and exit
    -o, --output string      write to output file instead of stdout
    -p, --pretty             output formatted json
    -r, --raw                never wrap in a MachineConfig; force Ignition output
    -s, --strict             fail on any warning
    -V, --version            print the version and exit
  ```

### 2. Creating a MachineConfig object by using Butane

- 시간 서비스에 대한 사용자 지정 설정을 지정하는 `99-worker-chrony.bu` 및 `99-master-chrony.bu` 파일을 생성합니다.

  - `99-worker-chrony.bu`

    ```yaml
    cat <<EOF > 99-worker-chrony.bu
    variant: openshift
    version: 4.15.0
    metadata:
      name: 99-worker-chrony 
      labels:
        machineconfiguration.openshift.io/role: worker 
    storage:
      files:
      - path: /etc/chrony.conf
        mode: 0644 
        overwrite: true
        contents:
          inline: |                   
            pool 192.168.123.100 iburst                  // pool의 IP 부분을 SDS NTP 서버 IP로 대체
            driftfile /var/lib/chrony/drift
            makestep 1.0 3
            rtcsync
            logdir /var/log/chrony
    EOF
    ```
    
  - `99-master-chrony.bu`
  
  ```yaml
    cat <<EOF > 99-master-chrony.bu
  variant: openshift
    version: 4.15.0
    metadata:
      name: 99-master-chrony 
      labels:
        machineconfiguration.openshift.io/role: master
    storage:
      files:
      - path: /etc/chrony.conf
        mode: 0644 
        overwrite: true
        contents:
          inline: |                   
            pool 192.168.123.100 iburst                  // pool의 IP 부분을 SDS NTP 서버 IP로 대체
            driftfile /var/lib/chrony/drift
            makestep 1.0 3
            rtcsync
            logdir /var/log/chrony
    EOF
    ```
  
- 위의 단계에서 만든 파일에 Butane을 제공하여 MachineConfing Object를 생성합니다.

  ```bash
  butane 99-worker-chrony.bu -o ./99-worker-chrony.yaml
  butane 99-master-chrony.bu -o ./99-master-chrony.yaml
  ```

- 설정 적용

  ```bash
  oc create -f 99-worker-chrony.yaml // 적용 후 완료 된 것 확인 후에 
  oc create -f 99-master-chrony.yaml // 설정 적용 
  ```

  > 한 번에 적용하시기 보다는 노드 별로 설정을 순차적으로 적용하여 정상적으로 진행되는 것을 확인해 주시는 것이 좋습니다.
  >
  > 노드가 적게 구성되어 있는 AI Cluster부터 진행부탁 드립니다.

- **OpenShfit Console 접속 > Compute > MachineConfigPool** 에서 진행 상태를 확인 하실 수 있습니다.

  ![13_mcp](C:\Works\01_자료\01_OCP\2023_삼성_GEN_AI\images\13_mcp.png)

- **CLI**로 확인 할 경우

  ```bash
  oc get mcp
  ```

- 이미지 레지스트리 노드 셀렉터

  ```bash
  oc patch configs.imageregistry.operator.openshift.io cluster --type=merge --patch '{"spec":{"nodeSelector":{"node-role.kubernetes.io/worker":""}}}'
  ```

  ```bash
  rolloutStrategy: RollingUpdate
  ```
  
  