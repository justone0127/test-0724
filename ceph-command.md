[ CEPH 전체 node에서 작업 ]

# cat /etc/hosts

:
10.210.54.67 sl-ceph01
10.210.54.68 sl-ceph02
10.210.54.69 sl-ceph03
10.210.54.70 sl-ceph-arbiter
10.210.54.79 yi-ceph01
10.210.54.80 yi-ceph02
10.210.54.81 yi-ceph03

10.238.222.160 sl-st-ceph
:


1. RHEL9에서 podman registry 서버 설정(podman 설치 완료되어 있음)

- /etc/containers/registries.conf 에 아래 내용 추가

[[registry]]
location="10.238.222.10:5050"
insecure=false 

(테스트 방법 1)

- 모든 ceph 서버에서

# podman images

:

# podman login 10.238.222.10:5050 -u admin -p [password]

Success ...

# curl -k 10.238.222.10:5050/v2/_catalog

:

(테스트 방법 2)

- sl-ceph01 서버에서

# podman login 10.238.222.10:5050 -u admin -p [password]

Success ...

# podman tag registry.redhat.io/openshift4/ose-prometheus:v4.12 10.238.222.10:5050/admin/openshift4/ose-prometheus:v4.12

# podman push 10.238.222.10:5050/admin/openshift4/ose-prometheus:v4.12

....
....
:

# podman pull 10.238.222.10:5050/admin/openshift4/ose-prometheus:v4.12

- 모든 ceph 서버에서

# podman pull 10.238.222.10:5050/admin/openshift4/ose-prometheus:v4.12

....
....

2. repo 설정 방법

1) 기존에 아래와 같이 설정된 부분의 enabled=0을 enabled=1 변경
baseurl=http://10.238.222.10:8081/rhel-9-for-x86_64-baseos-rpms
enabled=1
:
baseurl=http://10.238.222.10:8081/rhel-9-for-x86_64-appstream-rpms
enabled=1

2) 기존에 아래와 같이 설정된 부분의 enabled=1을 enabled=0 변경
baseurl=http://10.238.222.10:8081/rhel94/BaseOS
enabled=0
:
baseurl=http://10.238.222.10:8081/rhel94/AppStream
enabled=0

3) 전체 ceph 서버에서 작업 완료 후 아래 작업

# dnf clean all

# dnf repolist -v

3. 설치 작업(notion도 같이 참조)

* sl-ceph01에서 작업

1) update 및 reboot 

# dnf update -y

# sync;sync;reboot

2) ansible 확인

# su - deploy-user

$ sudo ansible -i /usr/share/cephadm-ansible/inventory -m ping all -b
$ exit

# ansible -i /usr/share/cephadm-ansible/inventory -m ping all -b

3) pre flight 수행

# ansible-playbook -i /usr/share/cephadm-ansible/inventory /usr/share/cephadm-ansible/cephadm-preflight.yml --extra-vars "ceph_origin=rhcs"

4) /root/registry.json 파일 수정

# vi /root/registry.json

{
 "url":"10.238.222.10:5050",
 "username":"admin",
 "password":"quay 암호"
}

5) /root/cluster-spec.yaml 파일 확인(아래 부분 반드시 수정 필요)
주의) datacenter: 부분을 아래와 같이 반드시 수정해야함
ceph 서버에 따라 datacenter를 아래와 같이 수정

- sl-ceph01, sl-ceph02, sl-ceph03은 
  :
  location:
  datacenter: seouldc

- yi-ceph01, yi-ceph02, yi-ceph03은 
  :
  location:
  datacenter: yongindc

- sl-ceph-arbiter 는 
  :
  location:
  datacenter: daejeondc
  :
  location:
  datacenter: seouldc
  :

6) Cephadm bootstrap 명령 수행(설치 명령)

# cephadm  bootstrap --ssh-user=deploy-user --mon-ip 10.210.54.67 --apply-spec /root/cluster-spec.yaml --registry-json /root/registry.json 

:
:

* 위 작업이 끝나면 화면에 dashboard에 접속할 수 있는 user명과 password가 나옴, 반드시 캡쳐 해야 함
* 이 작업 후 약 1 시간 이상 지켜 볼것

* 확인 방법은 

- sl-ceph01 서버에서 error 이 있는지 확인

# ceph -s

# journalctl -f

:
:

- ceph 전체 서버에서 podman images로 확인
  -> image가 계속 만들어 지면 수행 중임
- podman ps가 수행되면 잘 수행 중임

7) 위의 작업에서

- ceph -s 명령에서 아래와 유사하게 출력되면 성공

# ceph -s

  cluster:
    id:     4bddce50-436a-11ef-a937-525400ae214f
    health: HEALTH_OK

  services:
    mon: 5 daemons, quorum ceph1,ceph2,ceph4,ceph5,ceph7 (age 2d)
    mgr: ceph1.fxrkfz(active, since 2d), standbys: ceph4.rajrun
    mds: 1/1 daemons up, 4 standby
    osd: 30 osds: 30 up (since 2d), 30 in (since 8d)
    rgw: 2 daemons active (2 hosts, 1 zones)

  data:
    volumes: 1/1 healthy
    pools:   8 pools, 225 pgs
    objects: 267 objects, 16 MiB
    usage:   1.6 GiB used, 1.5 TiB / 1.5 TiB avail
    pgs:     225 active+clean

- ceph osd tree 명령에서 아래와 유사하게 출력되면 성공

# ceph os tree

ID  CLASS  WEIGHT   TYPE NAME           STATUS  REWEIGHT  PRI-AFF
-1         1.46399  root default
-3         0.73199      datacenter DC1
-2         0.24399          host ceph1
 2    hdd  0.04900              osd.2       up   1.00000  1.00000
 6    hdd  0.04900              osd.6       up   1.00000  1.00000
12    hdd  0.04900              osd.12      up   1.00000  1.00000
18    hdd  0.04900              osd.18      up   1.00000  1.00000
24    hdd  0.04900              osd.24      up   1.00000  1.00000
-4         0.24399          host ceph2
 0    hdd  0.04900              osd.0       up   1.00000  1.00000
 9    hdd  0.04900              osd.9       up   1.00000  1.00000
13    hdd  0.04900              osd.13      up   1.00000  1.00000
19    hdd  0.04900              osd.19      up   1.00000  1.00000
27    hdd  0.04900              osd.27      up   1.00000  1.00000
-5         0.24399          host ceph3
 1    hdd  0.04900              osd.1       up   1.00000  1.00000
 5    hdd  0.04900              osd.5       up   1.00000  1.00000
11    hdd  0.04900              osd.11      up   1.00000  1.00000
17    hdd  0.04900              osd.17      up   1.00000  1.00000
23    hdd  0.04900              osd.23      up   1.00000  1.00000
-7         0.73199      datacenter DC2
-6         0.24399          host ceph4
 4    hdd  0.04900              osd.4       up   1.00000  1.00000
 8    hdd  0.04900              osd.8       up   1.00000  1.00000
15    hdd  0.04900              osd.15      up   1.00000  1.00000
21    hdd  0.04900              osd.21      up   1.00000  1.00000
26    hdd  0.04900              osd.26      up   1.00000  1.00000
-8         0.24399          host ceph5
 3    hdd  0.04900              osd.3       up   1.00000  1.00000
 7    hdd  0.04900              osd.7       up   1.00000  1.00000
14    hdd  0.04900              osd.14      up   1.00000  1.00000
20    hdd  0.04900              osd.20      up   1.00000  1.00000
25    hdd  0.04900              osd.25      up   1.00000  1.00000
-9         0.24399          host ceph6
10    hdd  0.04900              osd.10      up   1.00000  1.00000
16    hdd  0.04900              osd.16      up   1.00000  1.00000
22    hdd  0.04900              osd.22      up   1.00000  1.00000
28    hdd  0.04900              osd.28      up   1.00000  1.00000
29    hdd  0.04900              osd.29      up   1.00000  1.00000


8) telemetry on(sl-ceph01 서버에서)

# ceph telemetry on

9) Verify if all the nodes are part of the cephadm cluster.

# ceph orch host ls

HOST   ADDR            LABELS              STATUS
sl-ceph01  10.210.54.67  _admin osd mon mgr
sl-ceph02  10.210.54.68  osd mon mds
sl-ceph03  10.210.54.69  osd mds rgw
yi-ceph01  10.210.54.79  osd mon mgr
yi-ceph02  10.210.54.80  osd mon
yi-ceph03  10.210.54.81  osd mds rgw
sl-ceph-arbiter  10.210.54.70  mon
7 hosts in cluster

10) mon node 확인

# ceph orch ps | grep mon | awk '{print $1 " " $2}'

* 위와같이 osd, mgr, mds도 확인

11) RBD block pool 생성

# ceph osd pool create rbdpool 32 32

# ceph osd pool application enable rbdpool rbd

12) CephFS volume 생성

# ceph fs volume create cephfs

# ceph fs status

13) Red Hat Ceph Storage stretch cluster 설정

# ceph mon dump | grep election_strategy

:

# ceph mon set election_strategy connectivity

:

# ceph mon dump | grep election_strategy

:

* 비교 해 보세요

14) Set the location for all our Ceph monitors

# ceph mon set_location sl-ceph01 datacenter=seouldc

# ceph mon set_location sl-ceph02 datacenter=seouldc

# ceph mon set_location yi-ceph01 datacenter=yongindc

# ceph mon set_location yi-ceph02 datacenter=yongindc

# ceph mon set_location sl-ceph-arbiter datacenter=daejeondc

아래 명령으로 확인

# ceph mon dump

:

15) crush map을 수정하기 위한 패키지 설치

# dnf -y install ceph-base

16) crush map binary 파일 다운 로드

# ceph osd getcrushmap > /etc/ceph/crushmap.bin

17) 수정하기 위한 text  파일로 변환

# crushtool -d /etc/ceph/crushmap.bin -o /etc/ceph/crushmap.txt

18) 수정 (파일 맨 아래에 아래 내용 추가)

# vi /etc/ceph/crushmap.txt

:
:
rule stretch_rule {
        id 1
        type replicated
        min_size 1
        max_size 10
        step take seouldc
        step chooseleaf firstn 2 type host
        step emit
        step take yongindc
        step chooseleaf firstn 2 type host
        step emit
}

19) 다시 binary 파일로 변환

# crushtool -c /etc/ceph/crushmap.txt -o /etc/ceph/crushmap20240726.bin

20) crush map 변경

# ceph osd setcrushmap -i /etc/ceph/crushmap20240726.bin

21) 확인

# ceph osd crush rule ls

:
22) Enable stretch cluster mode

# ceph mon enable_stretch_mode sl-ceph-arbiter stretch_rule datacenter

23) 전체 확인(여기서 각라인의 "crush_rule: stretch_rule" 확인)

# for pool in $(rados lspools);do echo -n "Pool: ${pool}; ";ceph osd pool get ${pool} crush_rule;done

Pool: device_health_metrics; crush_rule: stretch_rule
Pool: .rgw.root; crush_rule: stretch_rule
Pool: default.rgw.log; crush_rule: stretch_rule
Pool: default.rgw.control; crush_rule: stretch_rule
Pool: default.rgw.meta; crush_rule: stretch_rule
Pool: rbdpool; crush_rule: stretch_rule
Pool: cephfs.cephfs.meta; crush_rule: stretch_rule
Pool: cephfs.cephfs.data; crush_rule: stretch_rule


23) 전체 노드에 ceph.client.admin.keyring 등 복사

- sl-ceph01에서

# for i in sl-ceph01 sl-ceph02 sl-ceph03 sl-ceph-arbiter yi-ceph0s yi-ceph02 yi-ceph03

do
scp /etc/ceph/ceph.* ${i}:/etc/ceph/
done

24) ceph.conf 확인

# cat /etc/ceph/ceph.conf

[global]
        fsid = 4bddce50-436a-11ef-a937-525400ae214f (이건 틀림)
        mon_host = [v2:10.210.54.67:3300/0,v1:10.210.54.67:6789/0] [v2:10.210.54.68:3300/0,v1:10.210.54.68:6789/0] [v2:10.210.54.79:3300/0,v1:10.210.54.79:6789/0] [v2:10.210.54.80:3300/0,v1:10.210.54.80:6789/0] [v2:10.210.54.70:3300/0,v1:10.210.54.70:6789/0]
