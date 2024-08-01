## OADP를 이용한 VM 백업 복구

### 1. Object Bucket 생성

![oadp-bucket](C:\Works\01_자료\01_OCP\2024_롯데이노베이트_OV_PoC\images\oadp-bucket.png)

### 2. Bucket 정보 수집

![oadp-bucket-info](C:\Works\01_자료\01_OCP\2024_롯데이노베이트_OV_PoC\images\oadp-bucket-info.png)

- 다음 정보 확인

  - bucket name

    ```bash
    $ oc get configmap oadp-bucket -n default -o jsonpath='{.data.BUCKET_NAME}{"\n"}'
    oadp-bucket-3e05a7bc-43fc-4ba1-a461-9be5d9f85158
    ```

  - bucket host

    ```bash
    oc get configmap oadp-bucket -n default -o jsonpath='{.data.BUCKET_HOST}{"\n"}'
    s3.openshift-storage.svc
    ```

  - AWS_ACCESS_KEY_ID

    ```bash
    oc get secret oadp-bucket -n default -o jsonpath='{.data.AWS_ACCESS_KEY_ID}{"\n"}' | base64 -d
    I8MU2nXa4Sx4ym8msRfq
    ```

  - AWS_SECRET_ACCESS_KEY

    ```bash
    oc get secret oadp-bucket -n default -o jsonpath='{.data.AWS_SECRET_ACCESS_KEY}{"\n"}' | base64 -d
    sl/ngcTNVyTPTMci1P0VqS1gZ11X+sAD+XFnNs3d
    ```

- OADP Operator가 사용할 자격 증명 암호 생성

  앞서 생성한 ODF 오브젝트 스토리지 값을 사용하여`openshift-adp` 프로젝트에 `cloud-credentials` 라는 이름으로 시크릿 생성

  콘솔 > 워크로드 > 시크릿 > 키/값 

  다음 필드 작성

  - secret name : `cloud-credentials`

  - key : `cloud`

  - value:

    ```bash
    [default]
    aws_access_key_id=I8MU2nXa4Sx4ym8msRfq
    aws_secret_access_key=sl/ngcTNVyTPTMci1P0VqS1gZ11X+sAD+XFnNs3d
    ```

    ![odap-secret](C:\Works\01_자료\01_OCP\2024_롯데이노베이트_OV_PoC\images\odap-secret.png)

- DataProtectionApplication 사용자 정의 리소스 생성

  ```yaml
  kind: DataProtectionApplication
  apiVersion: oadp.openshift.io/v1alpha1
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
            s3Url: 'http://10.1.137.210:8080/'
          credential:
            key: cloud
            name: cloud-credentials
          default: true
          objectStorage:
            bucket: oadp-bucket-3e05a7bc-43fc-4ba1-a461-9be5d9f85158
            prefix: velero
          provider: aws
    configuration:
      nodeAgent:
        enable: true
        uploaderType: restic
      velero:
        defaultPlugins:
          - openshift
          - aws
          - csi
          - kubevirt
        featureFlags:
          - EnableCSI
  ```

- 리소스가 반영되면 다음과 같이 상태가 변합니다.

  ![odap-deployment](C:\Works\01_자료\01_OCP\2024_롯데이노베이트_OV_PoC\images\odap-deployment.png)

### 3. 백업 생성

가상머신 `rhel9-vm`의 백업을 수행합니다. `app` 및 `vm.kubevirt.io/name` 레이블에 의해 정의됩니다. 여기에는 구성 맵 및 비밀과 같이 가상머신에서 사용되는 가상머신 정의, 디스크 및 추가 개체가 포함됩니다.

1. 오퍼레이터 이름을 클릭하여 한 화면으로 돌아가 기본 오퍼레이터 페이지로 이동합니다

   ![oadp-operator](C:\Works\01_자료\01_OCP\2024_롯데이노베이트_OV_PoC\images\oadp-operator.png)

2. **Backup** 탭으로 이동하여 **Backup 만들기**를 누릅니다.

   ![oadp-backup](C:\Works\01_자료\01_OCP\2024_롯데이노베이트_OV_PoC\images\oadp-backup.png)

3. **Backup 만들기** 폼을 확인하고 *YAML 보기*로 전환 합니다.

   ![backup-yaml](C:\Works\01_자료\01_OCP\2024_롯데이노베이트_OV_PoC\images\backup-yaml.png)

   콘텐츠를 다음 항목으로 바꿉니다.

   ```yaml
   apiVersion: velero.io/v1
   kind: Backup
   metadata:
     name: backup-rhel9-vm
     labels:
       velero.io/storage-location: default
     namespace: openshift-adp
   spec:
     hooks: {}
     orLabelSelectors:
     - matchLabels:
         app: rhel9-vm
     - matchLabels:
         vm.kubevirt.io/name: rhel9-vm
     includedNamespaces:
     - adp-vm
     storageLocation: velero-sample-1
     ttl: 720h0m0s
   ```

   이 YAML의 콘텐츠는 `adp-vm` 네임스페이스에 `app: rhel9-vm` 레이블이 있는 모든 객체가 `DataProtectionApplication` 구성에 지정된 위치에 백업된다는 것을 나타냅니다.
   **만들기**를 누릅니다.

   ![oadp-backup-yaml](C:\Works\01_자료\01_OCP\2024_롯데이노베이트_OV_PoC\images\oadp-backup-yaml.png)

4. 상태 탭을 확인하여 진행 사항을 확인합니다.

   ![backup-status](C:\Works\01_자료\01_OCP\2024_롯데이노베이트_OV_PoC\images\backup-status.png)

   > 상태가 Completed가 되면 백업이 완료된 것 입니다.

### 4. 백업으로 복구

1. Virtualization > VirtualMachines로 이동하여 `rhel9-vm`을 삭제합니다.

   ![vm-delete](C:\Works\01_자료\01_OCP\2024_롯데이노베이트_OV_PoC\images\vm-delete.png)

2. Oprators > 설치된 Operator로 돌아가서 OADP Operator를 선택합니다.

   ![odap-operators](C:\Works\01_자료\01_OCP\2024_롯데이노베이트_OV_PoC\images\odap-operators.png)

3. Restore 탭으로 전환하고 Restore 만들기를 누릅니다.

   ![oadp-restore](C:\Works\01_자료\01_OCP\2024_롯데이노베이트_OV_PoC\images\oadp-restore.png)

4. YAML 보기로 전환하고 콘텐츠를 다음 항목으로 바꿉니다.

   ```yaml
   apiVersion: velero.io/v1
   kind: Restore
   metadata:
     name: restore-rhel9-vm
     namespace: openshift-adp
   spec:
     backupName: backup-rhel9-vm
     includedResources: []
     excludedResources:
     - nodes
     - events
     - events.events.k8s.io
     - backups.velero.io
     - restores.velero.io
     restorePVs: true
   ```

   ![oadp-restore-yaml](C:\Works\01_자료\01_OCP\2024_롯데이노베이트_OV_PoC\images\oadp-restore-yaml.png)

   만들기를 누릅니다.

   
