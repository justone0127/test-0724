## Yum Repository 준비하기

- 인터넷 접속 환경에서 필요한 repo를 다운 받고 Restricted Network 환경의 Bastion Node로 업로드합니다.
- 해당 작업은 인터넷이 되는 Red Hat Enterprise Linux 8 또는 9가 설치된 호스트에서 수행하면 됩니다.
- 해당 Repository는 Bastion 구성에 사용됩니다.



### 1. Yum Repository 구성

### 1.1 설치에 필요한 RPMS 파일 목록

- OpenShift 4.15.x 설치에 필요한 RMPS 목록 리스트

  - RHEL8

    ```bash
    subscription-manager repos \
        --enable="rhel-8-for-x86_64-baseos-rpms" \
        --enable="rhel-8-for-x86_64-appstream-rpms" \
        --enable="rhocp-4.15-for-rhel-8-x86_64-rpms" \
        --enable="fast-datapath-for-rhel-8-x86_64-rpms"
    ```

  - RHEL9

    ```bash
    subscription-manager repos \
        --enable="rhel-9-for-x86_64-baseos-rpms" \
        --enable="rhel-9-for-x86_64-appstream-rpms" \
        --enable="rhocp-4.15-for-rhel-9-x86_64-rpms" \
        --enable="fast-datapath-for-rhel-9-x86_64-rpms"
    ```

### 1.1 Subscription 등록

```bash
$ rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release  
// Red Hat 패키지를 확인하려면 Red Hat GPG KEY를 import 해야함 (Optional)

$ subscription-manager register
Registering to: subscription.rhsm.redhat.com:443/subscription
Username: ${USERNAME} // access.redhat.com 아이디
Password: ${PASSWORD} // access.redhat.com 패스워드
The system has been registered with ID: 6b924884-039f-408e-8fad-8d6b304bc2b5
The registered system name is: localhost.localdomain
```

> 등록 시 redhat customer portal ID/PWD를 입력합니다.

### 1.2 OpenShift 설치에 필요한 RPM 활성화 (OCP 4.15)

```bash
$ subscription-manager list --available --matches '*OpenShift*' > aaa.txt

// OpenShift Subscription만 따로 뽑아서 그 중 하나를 선택하여 등록 합니다.

$ subscription-manager attach --pool=2c94b3a58d5c53ce018d5cdcc3082004

$ subscription-manager repos --disable="*"

$ subscription-manager repos \
    --enable="rhel-9-for-x86_64-baseos-rpms" \
    --enable="rhel-9-for-x86_64-appstream-rpms" \
    --enable="rhocp-4.15-for-rhel-9-x86_64-rpms" \
    --enable="fast-datapath-for-rhel-9-x86_64-rpms"
```

### 1.3 Repo sync를 위한 패키지 다운로드

```bash
$ yum -y install yum-utils createrepo
```

### 1.4 Repo Sysnc

- Scripts 생성

  ```bash
  #!/bin/bash
  
  # 먼저 디렉토리를 생성합니다.
  mkdir -p /var/www/html/repos
  chmod -R +r /var/www/html/repos
  restorecon -vR /var/www/html
  
  for repo in rhel-9-for-x86_64-baseos-rpms rhel-9-for-x86_64-appstream-rpms rhocp-4.15-for-rhel-9-x86_64-rpms fast-datapath-for-rhel-9-x86_64-rpms
  do
      reposync --gpgcheck --downloadcomps --repoid=${repo} --newest-only --destdir=/var/www/html/repos/${repo}
      createrepo -v /var/www/html/repos/${repo}
  done
  ```

- Scripts 실행 권한 부여

  ```bash
  chmod 755 repo.sh
  ```

- Scripts 실행

  ```bash
  ./repo.sh
  ```

- 실행 시 오류는 무시해도 됩니다.

- yum update 하기

  ```bash
  yum update -y
  ```

- repositoy tar 압축

  - 압축 파일을 이동식 디스크에 복사하여 Restricted Network 환경으로 반입합니다.

    ```bash
    tar cvfz repos.tar.gz repos
    ```

### 2. Repo 설정 

위에서 압축한 파일을 서버로 업로드한 후, Repository를 구성합니다.

아래 두 가지 방법 중 선택하여 구성이 가능하며, 파일을 업로드 할 때 해당 위치에 맞게 디렉토리를 생성한 후 업로드를 합니다.

- 웹서버로 구성할 경우 (/etc/yum.repos.d/ocp4.repo)

  ```bash
  [rhel-8-for-x86_64-baseos-rpms]
  name = Red Hat Enterprise Linux 8 for x86_64 - BaseOS (RPMs)
  baseurl=http://10.76.168.19:8080/repos/rhel-8-for-x86_64-baseos-rpms/
  gpgcheck=0
  enabled=1
  module_hotfixes=1
  
  [rhel-8-for-x86_64-appstream-rpms]
  name = Red Hat Enterprise Linux 8 for x86_64 - AppStream (RPMs)
  baseurl=http://10.76.168.19:8080/repos/rhel-8-for-x86_64-appstream-rpms/
  gpgcheck=0
  enabled=1
  module_hotfixes=1
  
  [rhocp-4.15-for-rhel-8-x86_64-rpms]
  name = Red Hat OpenShift Container Platform 4.15 for RHEL 8 x86_64 (RPMs)
  baseurl=http://10.76.168.19:8080/repos/rhocp-4.15-for-rhel-8-x86_64-rpms/
  gpgcheck=0
  enabled=1
  module_hotfixes=1
  
  [fast-datapath-for-rhel-8-x86_64-rpms]
  name = Fast Datapath for RHEL 8 x86_64 (RPMs)
  baseurl=http://10.76.168.19:8080/repos/fast-datapath-for-rhel-8-x86_64-rpms/
  gpgcheck=0
  enabled=1
  module_hotfixes=1
  ```

- 로컬에 구성할 경우 (/etc/yum.repos.d/local.repo)

  ```bash
  [rhel-8-for-x86_64-baseos-rpms]
  name = Red Hat Enterprise Linux 8 for x86_64 - BaseOS (RPMs)
  baseurl=file:///opt/openshift/download/repos/rhel-8-for-x86_64-baseos-rpms/
  gpgcheck=0
  enabled=1
  module_hotfixes=1
  
  [rhel-8-for-x86_64-appstream-rpms]
  name = Red Hat Enterprise Linux 8 for x86_64 - AppStream (RPMs)
  baseurl=file:///opt/openshift/download/repos/rhel-8-for-x86_64-appstream-rpms/
  gpgcheck=0
  enabled=1
  module_hotfixes=1
  
  [rhocp-4.15-for-rhel-8-x86_64-rpms]
  name = Red Hat OpenShift Container Platform 4.15 for RHEL 8 x86_64 (RPMs)
  baseurl=file:///opt/openshift/download/repos/rhocp-4.15-for-rhel-8-x86_64-rpms/
  gpgcheck=0
  enabled=1
  module_hotfixes=1
  
  [fast-datapath-for-rhel-8-x86_64-rpms]
  name = Fast Datapath for RHEL 8 x86_64 (RPMs)
  baseurl=file:///opt/openshift/download/repos/fast-datapath-for-rhel-8-x86_64-rpms/
  gpgcheck=0
  enabled=1
  module_hotfixes=1
  ```

- yum cache 업데이트 : 새로운 yum repository를 사용하기 전에 yum cache를 업데이트 해야 함

  ```bash
  yum clean all
  yum makecache
  ```

- 패키지 설치 테스트

  ```bash
  yum install -y httpd
  ```

- 패키지 설치 시 다음 오류가 나는 경우에는 repo 설정 파일에 옵션을 추가해 주어야 합니다.

  - 에러 메시지

    ```bash
    ==========================================================================================================================================================
    Install  48 Packages
    
    Total size: 27 M
    Installed size: 81 M
    Downloading Packages:
    Running transaction check
    No available modular metadata for modular package 'perl-IO-Socket-SSL-2.066-4.module+el8.3.0+6446+594cad75.noarch', it cannot be installed on the system
    No available modular metadata for modular package 'perl-Mozilla-CA-20160104-7.module+el8.3.0+6498+9eecfe51.noarch', it cannot be installed on the system
    No available modular metadata for modular package 'perl-Net-SSLeay-1.88-2.module+el8.6.0+13392+f0897f98.x86_64', it cannot be installed on the system
    Error: No available modular metadata for modular package
    ```

    > yum module은 8버전부터 도입된 새로운 기능으로, 패키지 그룹의 모듈 현태로 소프트웨어를 제공하는 패키지를 관리할 수 있는 도구입니다. 모듈은 특정 버전이나 프로파일에 따라 다른 기능 집합을 제공하며, 사용자는 필요한 모듈을 선택하여 시스템에 설치하고 업데이트할 수 있습니다.

  - 비활성화 방법

    - /etc/yum.repo.d/local.repo 파일에 다음 옵션을 추가합니다.

      - 옵션 : `module_hotfixes=1`

        ```bash
        [rhel-8-for-x86_64-baseos-rpms]
        name = Red Hat Enterprise Linux 8 for x86_64 - BaseOS (RPMs)
        baseurl=file:///opt/openshift/download/repos/rhel-8-for-x86_64-baseos-rpms/
        gpgcheck=0
        enabled=1
        module_hotfixes=1
        
        [rhel-8-for-x86_64-appstream-rpms]
        name = Red Hat Enterprise Linux 8 for x86_64 - AppStream (RPMs)
        baseurl=file:///opt/openshift/download/repos/rhel-8-for-x86_64-appstream-rpms/
        gpgcheck=0
        enabled=1
        module_hotfixes=1
        
        [rhocp-4.15-for-rhel-8-x86_64-rpms]
        name = Red Hat OpenShift Container Platform 4.15 for RHEL 8 x86_64 (RPMs)
        baseurl=file:///opt/openshift/download/repos/rhocp-4.15-for-rhel-8-x86_64-rpms/
        gpgcheck=0
        enabled=1
        module_hotfixes=1
        
        [fast-datapath-for-rhel-8-x86_64-rpms]
        name = Fast Datapath for RHEL 8 x86_64 (RPMs)
        baseurl=file:///opt/openshift/download/repos/fast-datapath-for-rhel-8-x86_64-rpms/
        gpgcheck=0
        enabled=1
        module_hotfixes=1
        ```

        > 모듈 옵션 비활성화를 추가하면 패키지가 정상적으로 설치 됨!!!
