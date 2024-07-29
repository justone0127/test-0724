## ApplicationSet을 이용한 OpenShift Virtualization 배포하기

- 사전 요구 사항
- ACM, Gitea 구성, VM 배포 Applicationset 정의 등

### 1. ApplicationSet

`ApplicationSet`은 GitOps Operator가 지원하는 ArgoCD의 하위 프로젝트입니다. `ApplicationSet`은 Argo CD 애플리케이션에 대한 다중 클러스터 지원을 추가합니다.  Red Hat Advanced Cluster Management 콘솔에서 `ApplicationSet`을 생성할 수 있습니다.



OpenShift Container Platform GitOps는 ArgoCD를 사용하여 클러스터 리소스를 유지 관리합니다. ArgoCD는 애플리케이션의 지속적 통합 및 지속적 배포(CI/CD)를 위한 오픈소스 선언적 도구입니다. OpenShift Container Platform GitOps는 ArgoCD를 컨트롤러(OpenShift Container Platform GitOps Operator)로 구현하여 Git 저장소에 정의된 애플리케이션 정의 및 구성을 지속적으로 모니터링합니다. 그런 다음 ArgoCD는 이러한 구성의 지정된 상태를 클러스터의 현재 상태와 비교합니다.



`ApplicationSet` 컨트롤러는 GitOps Operator 인스턴스를 통해 클러스터에 설치되며 클러스터 관리자 중심 시나리오를 지원하는 추가 기능을 추가하여 이를 보완합니다. `ApplicationSet` 컨트롤러는 다음 기능을 제공합니다.

- 단일 Kubernetes 매니페스트를 사용하여 GitOps Operator로 여러 Kubernetes 클러스터를 대상으로 하는 기능
- 단일 Kubernetes 매니페스트를 사용하여 GitOps Operator를 통해 하나 이상의 Git 리포지토리에서 여러 애플리케이션을 배포하는 기능
- 단일 Git 리포지토리 내에 정의된 여러 ArgoCD 애플리케이션 리소스인 ArgoCD의 컨텍스트에서 monorepo에 대한 지원이 향상되었습니다.
- 다중 테넌트 클러스터 내에서 대상 클러스터/네임스페이스를 활성화 할 때 권한 있는 클러스터 관리자를 포함할 필요 없이 ArgoCD를 사용하여 애플리케이션을 배포할 수 있는 개별 클러스터 테넌트의 기능이 향상되었습니다.



`ApplicationSet` Operator는 클러스터 결정 생성기를 활용하여 사용자 지정 리소스별 로직을 사용하여 배포할 관리 클러스터를 결정하는 Kubernetes 사용자 지정 리소스를 인터페이스 합니다. 클러스터 결정 리소스는 관리되는 클러스터 목록을 생성한 다음 `ApplicationSet` 리소스의 템플릿 필드로 렌더링됩니다. 이것은 참조된 Kubernetes 리소스의 전체 형태에 대한 지식이 필요하지 않은 duck-typing을 사용하여 수행됩니다.

### 2. Configuring Managed Clusters for OpenShift GitOps Operator

GitOps를 구성하기 위해 Red Hat OpenShift Container Platform GitOps Operator 인스턴스에 RHACM에서 관리되는 클러스터를 하나 이상의 세트로 등록할 수 있습니다. 등록한 후 해당 클러스터에 애플리케이션을 배포할 수 있습니다. 지속적인 GitOps 환경을 설정하여 개발, 스테이징 및 프로덕션 환경의 클러스터에서 애플리케이션 일관성을 자동화합니다.

### 2.1 Prerequisites

- RHACM(Red Hat Advanced Cluster Management)에 OpenShift GitOps Operator를 설치해야 합니다.
- 하나 이상의 관리형 클러스터를 가져옵니다.

### 2.2 Registering managed clusters to GitOps

- 관리형 클러스터 세트를 생성하고 관리형 클러스터를 해당 관리형 클러스터 세트에 추가합니다.

- **RHACM 콘솔 접속 > Infrastructure > Clusters > Cluster sets > Create cluster set**

  ![01_cluster_set](C:\Works\01_자료\01_OCP\05_OCP_Demo_hyou\ApplicationSet_RHACM\01_cluster_set.png)
  - Cluster set name 입력

    ![02_create_cluster_set](C:\Works\01_자료\01_OCP\05_OCP_Demo_hyou\ApplicationSet_RHACM\02_create_cluster_set.png)

- Manage resource assignments 선택

  위에서 생성한 Cluster  sets가 생성되면 `Managed resource assignments`를 선택합니다.

  ![03_manage_resource_assignments](C:\Works\01_자료\01_OCP\05_OCP_Demo_hyou\ApplicationSet_RHACM\03_manage_resource_assignments.png)

  대상 클러스터를 선택합니다.

  ![04_argocd_manage_cluster](C:\Works\01_자료\01_OCP\2024_롯데이노베이트_OV_PoC\images\04_argocd_manage_cluster.png)

- **RHACM 콘솔 > Infrastructure > Clusters > Cluster sets > 생성한 Cluster set > Actions > Edit namespace bindings**

  ![05_edit_namespace_bindings01](C:\Works\01_자료\01_OCP\05_OCP_Demo_hyou\ApplicationSet_RHACM\05_edit_namespace_bindings01.png)
  - Namespaces : GitOps Operator Instance가 구성된 네임스페이스 (`openshift-gitops`)

    ![06_edit_namespace_bindings02](C:\Works\01_자료\01_OCP\05_OCP_Demo_hyou\ApplicationSet_RHACM\06_edit_namespace_bindings02.png)

- ManagedClusterSet에 사용자 또는 그룹 역할 기반 액세스 제어 권한 할당(RBAC)
  - **ACM 콘솔 접속 > Clusters > Cluster set > `acm-argoset` > Access management > Add User or group > Select User : `admin` > Select role : `Cluster set admin`**

    ![07_access_management](C:\Works\01_자료\01_OCP\05_OCP_Demo_hyou\ApplicationSet_RHACM\07_access_management.png)

  - 생성 확인

    ![08_confirm_access_management](C:\Works\01_자료\01_OCP\05_OCP_Demo_hyou\ApplicationSet_RHACM\08_confirm_access_management.png)

- ManagedClusterSet binding에 사용되는 네임스페이스에서 사용자 지정 리소스를 생성하여 OpenShift GitOps Operator 인스턴스에 등록할 배치 사용자 리소스 생성

  **ACM에서 GitOps Operator Instance로 등록해서 사용할 리소스**

  ```yaml
  cat << EOF > managedcluster-argoset.yaml
  apiVersion: cluster.open-cluster-management.io/v1beta1
  kind: Placement
  metadata:
    name: acm-argoset
    namespace: openshift-gitops
  spec:
    predicates:
    - requiredClusterSelector:
        labelSelector:
          matchExpressions:
          - key: vendor
            operator: "In"
            values:
            - OpenShift
  EOF
  
  oc apply -f managedcluster-argoset.yaml
  ```

- `GitOpsCluster` 배치 결정에서 지정된 인스턴스까지 관리되는 ClusterSet을 등록하기 위한 사용자 정의 리소스 생성

  ```yaml
  cat << EOF > gitops-cluster.yaml
  apiVersion: apps.open-cluster-management.io/v1beta1
  kind: GitOpsCluster
  metadata:
    name: gitops-cluster
    namespace: openshift-gitops
  spec:
    argoServer:
      cluster: local-cluster
      argoNamespace: openshift-gitops
    placementRef:
      kind: Placement
      apiVersion: cluster.open-cluster-management.io/v1beta1
      name: acm-argoset  
  EOF
  
  oc apply -f gitops-cluster.yaml
  ```

  > 참고)
  >
  > spec 부분
  >
  > - argoserver > cluster :  이 부분이 gitOps 인스턴스가 설치된 클러스터를 지정!! 
  >   - placementRef > name: GitOps 인스턴스에서 관리될 클러스터 (acm-argoset)

- `local cluster`의 GitOps 인스턴스 콘솔로 접속하여 새로 추가된 클러스터(`primary`) 정보 확인

  ![03_primary_cluster](C:\Works\01_자료\01_OCP\2024_롯데이노베이트_OV_PoC\images\03_primary_cluster.png)

### 3. Configure ArgoCD RBAC on OpenShift

### 3.1 Configure ArgoCD RBAC

기본적으로 RHSSO를 사용하여 ArgoCD에 로그인 한 경우에는 읽기 전용(read-only) 사용자입니다. 사용자 수준의 액세스를 변경하고 관리 할 수 있습니다.

- CLI 명령어로 확인 (편집 모드)

  ```bash
  oc edit argocd openshift-gitops -n openshift-gitops
  ```

- rbac 부분의 defaultPolicy 확인

  ```bash
    rbac:
      defaultPolicy: {}
  ```

  > 값이 할당되어 있지 않은 것으로 보여질 것입니다.

- `role:admin` 권한을 추가합니다.

  ```bash
    rbac:
      defaultPolicy: 'role:admin'
  ```

- 콘솔에서 작업하는 경우에는 **CustomResourceDefinitions** > **ArgoCD** > **openshift-gitops** 를 선택하여 rbac 부분을 수정합니다.

  ![01_argocd_rbac](C:\Works\01_자료\01_OCP\2024_롯데이노베이트_OV_PoC\images\01_argocd_rbac.png)

- 콘솔에서 **argocd-rbac-cm** 에 **policy.default**에 값이 반영된 것을 확인 할 수 있습니다.

  ![02_argocd_rabc_cm](C:\Works\01_자료\01_OCP\2024_롯데이노베이트_OV_PoC\images\02_argocd_rabc_cm.png)
  - LOG IN VIA OPENSHFIT

    OpenShift 계정으로도 ArgoCD 인스턴스에 접속하여 Admin 권한으로 배포가 가능해집니다.

    ![04_argocd_login_openshift](C:\Works\01_자료\01_OCP\05_OCP_Demo_hyou\brown_bag_gitops\gitops-rbac\04_argocd_login_openshift.png)

### 4. Deploying OpenShift Virtualization ApplicationSet

ApplicationSet을 이용하여 RHACM에서 대상 클러스터에 애플리케이션을 배포합니다.

4.15버전에서는 ApplicationSet으로 배포할 경우 Pull model, Push model이 존재합니다. Pull model의 경우 local-cluster에 애플리케이션 배포를 지원하지 않습니다.

- Push model : Push model을 사용하여 ACM Hub Cluster의 Argo CD 서버는 관리되는 클러스터에 애플리케이션 리소스를 배포합니다. Push model 구현에는 관리되는 클러스터에 대한 자격 증명이 있는 Hub Cluster의 Argo CD 애플리케이션만 포함됩니다. 

- Pull model: Pull mode의 경우 OpenShift Cluster Manager registration, placement, `manifestWork` API를 적용하므로 Hub Cluster는 Hub Cluster와 관리되는 Cluster 간의 보안 통신 채널을 사용하여 리소스를 배포할 수 있습니다. Pull model의 경우 Argo CD 서버는 각 관리 대상 클러스터에서 실행되어야 합니다. Argo CD 애플리케이션 리소스는 관리되는 클러스터에 복제된 후 local Argo CD 서버에 의해 배포됩니다. 관리되는 클러스터의 분산 Argo CD 애플리케이션은 Hub Cluster의 단일 Argo CD `ApplicationSet `리소스를 사용하여 생성됩니다. 

  - 필수 접근 권한 : Cluster Administrator

  - 중요 : `openshift-gitops-ArgoCD-application-controller` service account 계정이 클러스터 관리자로 할당되지 않은 경우 GitOps 애플리케이션 컨트롤러가 리소스를 배포하지 않을 수 있습니다. 권한이 없는 경우 다음과 유사한 오류를 내보낼 수 있습니다.

    ```bash
    cannot create resource "services" in API group "" in the namespace
    "mortgage",deployments.apps is forbidden: User
    "system:serviceaccount:openshift-gitops:openshift-gitops-Argo CD-application-controller"
    ```

    > 이 가이드에서는 위에서 cluster-admin 권한을 주었기 때문에 에러가 발생하지 않을 것 입니다.

  - 만약, 클러스터 관리자 권한이 아니고 이 문제를 해결하기 위해서는 다음과 같이 조치를 취할 수 있습니다.

    1. ArgoCD 애플리케이션이 배포될 각 관리클러스터에 네임스페이스를 생성합니다.

    2. 각 네임스페이스에 `managed-by` 레이블을 추가합니다. ArgoCD 애플리케이션이 여러 네임스페이스에 배포된 경우 각 네임스페이스는 ArgoCD로 관리 되어야 합니다.

       - 예시

         ```yaml
         apiVersion: v1
         kind: Namespace
         metadata:
           name: mortgage2
           labels:
             argocd.argoproj.io/managed-by: openshift-gitops
         ```

    3. 애플리케이션 저장소에서 모든 애플리케이션 대상 네임스페이스를 선언하고 네임스페이스에 `managed-by` 레이블을 포함해야 합니다. 네임스페이스를 선언하는 방법을 알아보려면 추가 리소스를 참조하세요.

**ACM 콘솔 접속 > Applications > Create application > ApplicationSet**

- **General**

  ![05_create_applicaion_set_general](C:\Works\01_자료\01_OCP\2024_롯데이노베이트_OV_PoC\images\05_create_applicaion_set_general.png)

  - ApplicationSet name : test-vm
  - Argo server: openshift-gitops
  - Requeue time: 180 (default)

- **Template**

  ![06_create_application_set_template](C:\Works\01_자료\01_OCP\2024_롯데이노베이트_OV_PoC\images\06_create_application_set_template.png)

  - Source
    - Resource Type : Git
    - URL : ${GIT_REPOSITORY}
    - Revision : main
    - Path : ov
  - Destination
    - Remote namespace : test-vm

- **Source Policy (기본값으로 두고 진행)**

  ![07_sync_policy](C:\Works\01_자료\01_OCP\2024_롯데이노베이트_OV_PoC\images\07_sync_policy.png)

- **Placement**

  ![08_placement](C:\Works\01_자료\01_OCP\2024_롯데이노베이트_OV_PoC\images\08_placement.png)

  - Cluster sets : `acm-argoset` (위에서 생성한 대상 클러스터 세트 선택)

- **Review**

  ![09_review](C:\Works\01_자료\01_OCP\2024_롯데이노베이트_OV_PoC\images\09_review.png)

- **Topology 확인 (ACM Console)**

  ![10_topolozy](C:\Works\01_자료\01_OCP\2024_롯데이노베이트_OV_PoC\images\10_topolozy.png)

- **ArgoCD 콘솔 확인 (4.15)**

  ![11_argocd_application](C:\Works\01_자료\01_OCP\2024_롯데이노베이트_OV_PoC\images\11_argocd_application.png)

- Primary 클러스터 확인

  Primary 클러스터의 `test-vm` 프로젝트를 확인해보면 다음과 같이 VM이 배포되어 실행중인 것을 확인할 수 있습니다.

  ![12_primary_vm](C:\Works\01_자료\01_OCP\2024_롯데이노베이트_OV_PoC\images\12_primary_vm.png)