### 1. OpenShift Data Foundation Operator 설치 확인

- 설치 확인

  ```bash
  oc get storagecluster -n openshift-storage ocs-storagecluster -o jsonpath='{.status.phase}{"\n"}'
  ```

  > Ready 출력 확인 

- MCG 확인

  ```bash
  oc get noobaa -n openshift-storage noobaa -o jsonpath='{.status.phase}{"\n"}'
  ```

  >  Ready 출력 확인

- 오브젝트 버킷 클레임 생성

  ObJect Bucket Claim은 백업된 Kubernetes 매니페스트를 저장하기 위해 Velero용 영구 스토리지 버킷을 생성 

  - Storage > Object Bucket Claim > Create Object Bucket Claim 선택 
  - 현재 있는 프로젝트를 기록해둡니다. 새 프로젝트를 만들거나 기본값으로 둘 수 있습니다.
  - 다음 값을 설정 
    - ObjectBucketClaim Name : `oadp-bucket`
    - StorageClass : `openshift-storage.noobaa.io`
    - BucketClass : `noobaa-default-bucket-class`

  ![14_create_bucket_claim](C:\Works\01_자료\01_OCP\2023_삼성_데모\images\14_create_bucket_claim.png)
  - 다음과 같이 Bound가 되면 완료

  ![15_bucket_claim](C:\Works\01_자료\01_OCP\2023_삼성_데모\images\15_bucket_claim.png)

### 2. Object Bucket에서 정보 수집

- Bucket 이름 및 호스트 수집 

- CLI 활용 

  - Bucket 이름 가져오기

    ```bash
    oc get configmap oadp-bucket -n default -o jsonpath='{.data.BUCKET_NAME}{"\n"}'
    ```

  - Bucket 호스트 가져오기

    ```bash
    oc get configmap oadp-bucket -n default -o jsonpath='{.data.BUCKET_HOST}{"\n"}'
    ```

- 콘솔 활용

  - Object Bucket obc-default-oadp-bucket을 선택하고 YAML 보기 선택

    ![15_object_buckets](C:\Works\01_자료\01_OCP\2023_삼성_데모\images\15_object_buckets.png)

- oadp-bucket secret 수집

- CLI 사용

  - AWS_ACCESS_KEY 가져오기

    ``` bash
    oc get secret oadp-bucket -n default -o jsonpath='{.data.AWS_ACCESS_KEY_ID}{"\n"}' | base64 -d
    ```

  - AWS_SECRET_ACCESS_KEY 가져오기

    ```bash
    oc get secret oadp-bucket -n default -o jsonpath='{.data.AWS_SECRET_ACCESS_KEY}{"\n"}' | base64 -d
    ```

- 콘솔 사용

  -  스토리지 > Object Bucket Claim > oadp-bucket 이동 > oadp-secert 확인

    ![16_oadp_secret](C:\Works\01_자료\01_OCP\2023_삼성_데모\images\16_oadp_secret.png)

- 다음 정보를 확인

  - bucket name

  - bucket host

  - AWS_ACCESS_KEY_ID

  - AWS_SECRET_ACCESS_KEY

    ```bash
    oc get configmap oadp-bucket -n default -o jsonpath='{.data.BUCKET_NAME}{"\n"}'
    oadp-bucket-8876d57e-8d27-4e62-a88d-d764fe09db4e
    
    oc get configmap oadp-bucket -n default -o jsonpath='{.data.BUCKET_HOST}{"\n"}'
    s3.openshift-storage.svc
    
    oc get secret oadp-bucket -n default -o jsonpath='{.data.AWS_ACCESS_KEY_ID}{"\n"}' | base64 -d
    XsEVa87KNUzueLhGgoJe
    
    oc get secret oadp-bucket -n default -o jsonpath='{.data.AWS_SECRET_ACCESS_KEY}{"\n"}' | base64 -d
    48+twe02WkScqUCU04chIDVUMLd8cxnuiU/ill3c
    ```

### 3. 애플리케이션 배포

- 애플리케이션 배포

  ```bash
  git clone https://github.com/kaovilai/mig-demo-apps --single-branch -b oadp-blog-rocketchat
  cd mig-demo-apps
  ```

- 로켓 챗 매니페스트를 적용

  ```bash
  oc new-project rocket-chat
  ```

- 설정 배포

  ```yaml
  ---
  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: rocketchat-data-claim
    namespace: rocket-chat
  spec:
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 10Gi
    storageClassName: ocs-storagecluster-ceph-rbd
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: rocket-chat-db
    namespace: rocket-chat
  spec:
    ports:
      - protocol: TCP
        port: 27017
        targetPort: 27017
    selector:
      app: rocketchat-db
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: rocket-chat
    namespace: rocket-chat
  spec:
    ports:
      - protocol: TCP
        port: 3000
        targetPort: 3000
    selector:
      app: rocketchat
  ---
  apiVersion: apps.openshift.io/v1
  kind: DeploymentConfig
  metadata:
    name: rocketchat-db
    labels:
      app: rocketchat-db
    namespace: rocket-chat
  spec:
    replicas: 1
    selector:
      app: rocketchat-db
    strategy:
      type: Recreate
      recreateParams:
        post:
          failurePolicy: ignore
          execNewPod:
            volumes:
              - rocketchat-data-vol
            containerName: rocketchat-db
            command:
              - /bin/sh
              - -c
              - "sleep 60 && mongo rocket-chat-db:27017 --eval \"rs.initiate({_id: 'rs0', members: [{_id:0, host:'localhost:27017'}]})\""
    template:
      metadata:
        labels:
          app: rocketchat-db
      spec:
        containers:
          - name: rocketchat-db
            image: docker.io/mongo:4.0
            command:
              - /bin/sh
              - -c 
              - mongod --bind_ip 0.0.0.0 --port 27017 --smallfiles --oplogSize 128 --replSet rs0 --storageEngine=mmapv1
            volumeMounts:
              - name: rocketchat-data-vol
                mountPath: /data/db
            ports:
              - containerPort: 27017
                protocol: TCP
        restartPolicy: Always
        volumes:
          - name: rocketchat-data-vol
            persistentVolumeClaim:
              claimName: rocketchat-data-claim
  ---
  apiVersion: apps.openshift.io/v1
  kind: DeploymentConfig
  metadata:
    name: rocketchat
    namespace: rocket-chat
    labels:
      app: rocketchat
  spec:
    replicas: 1
    selector:
      app: rocketchat
    template:
      metadata:
        labels:
          app: rocketchat
      spec:
        containers:
          - name: rocketchat
            image: rocket.chat:2.4.9
            env:
              - name: MONGO_URL
                value: "mongodb://rocket-chat-db:27017/rocketchat"
              - name: MONGO_OPLOG_URL
                value: "mongodb://rocket-chat-db:27017/local"
            ports:
              - containerPort: 3000
                protocol: TCP
            resources:
              requests:
                cpu: 100m
        restartPolicy: Always
  ---
  apiVersion: route.openshift.io/v1
  kind: Route
  metadata:
    name: rocket-chat
    namespace: rocket-chat
  spec:
    port:
      targetPort: 3000
    to:
      kind: Service
      name: rocket-chat
      weight: 100
    wildcardPolicy: None
  ```

- OADP Opreator가 사용할 자격 증명 암호 생성

  앞서 생성한 ODF 오브젝트 스토리지 값을 사용하여 `cloud-credentials`라는 이름으로 시크릿 생성

  콘솔 > 워크로드 > 시크릿 > 키/값 시크릿 선택

  다음 필드 작성 

  - Secret name : `cloud-credentials`

  - Key : `cloud`

  - value:

    ```bash
    [default]
    aws_access_key_id=XsEVa87KNUzueLhGgoJe
    aws_secret_access_key=48+twe02WkScqUCU04chIDVUMLd8cxnuiU/ill3c
    ```

  ![16_oadp_secret](C:\Works\01_자료\01_OCP\2023_삼성_데모\images\16_oadp_secret.png)

- DataProtectionApplication 사용자 정의 리소스 생성 

  ```yaml
  apiVersion: oadp.openshift.io/v1alpha1
  kind: DataProtectionApplication
  metadata:
    name: velero-sample
    namespace: openshift-adp
  spec:
    backupLocations:
      - velero:
          config:
            profile: default
            region: localstorage
            s3ForcePathStyle: 'true'
            s3Url: 'http://s3.openshift-storage.svc/'
          credential:
            key: cloud
            name: cloud-credentials
          default: true
          objectStorage:
            bucket: oadp-bucket-8876d57e-8d27-4e62-a88d-d764fe09db4e
            prefix: velero
          provider: aws
    configuration:
      restic:
        enable: true
      velero:
        defaultPlugins:
          - openshift
          - aws
          - csi
  ```
  
- 설치 확인

  ```bash
   oc get all -n openshift-adp
  Warning: kubevirt.io/v1 VirtualMachineInstancePresets is now deprecated and will be removed in v2.
  NAME                                                    READY   STATUS    RESTARTS   AGE
  pod/node-agent-8tbfc                                    1/1     Running   0          5m9s
  pod/node-agent-fhcng                                    1/1     Running   0          5m9s
  pod/node-agent-hrrgz                                    1/1     Running   0          5m9s
  pod/openshift-adp-controller-manager-7bff6bf658-lzsst   1/1     Running   0          49m
  pod/velero-5d77ddff68-lpr76                             1/1     Running   0          5m9s
  
  NAME                                                       TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
  service/openshift-adp-controller-manager-metrics-service   ClusterIP   172.30.179.163   <none>        8443/TCP   49m
  service/openshift-adp-velero-metrics-svc                   ClusterIP   172.30.240.119   <none>        8085/TCP   5m9s
  
  NAME                        DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
  daemonset.apps/node-agent   3         3         3       3            3           <none>          5m9s
  
  NAME                                               READY   UP-TO-DATE   AVAILABLE   AGE
  deployment.apps/openshift-adp-controller-manager   1/1     1            1           49m
  deployment.apps/velero                             1/1     1            1           5m9s
  
  NAME                                                          DESIRED   CURRENT   READY   AGE
  replicaset.apps/openshift-adp-controller-manager-7bff6bf658   1         1         1       49m
  replicaset.apps/velero-5d77ddff68                             1         1         1       5m9s
  ```

- 볼륨 스냅샷 클래스 수정

  VolumeSnapshotClass에 `DeletionPolicy`를 `Retain`으로 설정하면, VolumeSnapshot이 저장 시스템에서 영구히 유지되며, Velero 백업의 수명 동안 VolumeSnapshot의 삭제가 방지됩니다. 이는 네임스페이스와 VolumeSnapshot 객체가 손실될 수 있는 재해 상황에서도 저장 시스템에서 VolumeSnapshot의 삭제를 방지하기 위한 것입니다.
  
  Velero CSI 플러그인은 CSI를 백업하는 데 사용되는 PVCs(영구 볼륨)에 대해 클러스터 내에서 동일한 드라이버 이름을 가진 VolumeSnapShotClass를 선택하며, 또한 `velero.io/csi-volumesnapshot-class: "true"` 레이블이 설정된 VolumeSnapShotClass를 선택합니다.
  
```bash
  oc patch volumesnapshotclass ocs-storagecluster-rbdplugin-snapclass --type=merge -p'{"deletionPolicy": "Retain"}'
  oc label volumesnapshotclass ocs-storagecluster-rbdplugin-snapclass velero.io/csi-volumesnapshot-class="true"
  ```

- 콘솔에서 적용하는 경우 

  스토리지 > VolumeSnapshotClass 이동 > ocs-storagecluster-rdbplugin-snapclass 클릭 아래에 표시된 내용대로 값을 수정하려면 YAML 보기 클릭

  ```yaml
  apiVersion: snapshot.storage.k8s.io/v1
  deletionPolicy: Retain
  driver: openshift-storage.rbd.csi.ceph.com
  kind: VolumeSnapshotClass
  metadata:
    name: ocs-storagecluster-rbdplugin-snapclass
    resourceVersion: '39764'
    uid: ee6de0af-0405-4c71-8eef-588b32d8bd51
    labels:
      velero.io/csi-volumesnapshot-class: "true"
  parameters:
    clusterID: openshift-storage
    csi.storage.k8s.io/snapshotter-secret-name: rook-csi-rbd-provisioner
    csi.storage.k8s.io/snapshotter-secret-namespace: openshift-storage
  ```

- 백업 인스턴스 생성
- 네임스페이스 삭제
- 복구 인스턴스 생성

