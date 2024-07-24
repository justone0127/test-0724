```bash
-rw-r--r--. 1 lab-user users 5558 May 31 03:54 repos.tar.gz

[lab-user@bastion ~]$

[lab-user@bastion ~]$

[lab-user@bastion ~]$ pwd

/home/lab-user
```

- 로컬 > OV로 파일 복사!!!

  ```bash
  [lab-user@bastion ~]$ virtctl scp ./repos.tar.gz cloud-user@rhel8-vm:/home/cloud-user/. -n ov-demo
  
  Warning: Permanently added 'rhel8-vm.ov-demo' (ECDSA) to the list of known hosts.
  
  no such identity: /home/lab-user/.ssh/bastion_6w2r8: No such file or directory
  
  cloud-user@rhel8-vm.ov-demo's password:
  
  repos.tar.gz
  ```

- /etc/ssh/ssh_config 설정 변경

  ```bash
  PasswordAuthentication yes
  
  PermitRootLogin yes 변경 
  
  systemctl restart sshd 
  ```

- 원격 OV > 로컬로 파일 다운로드

  ```bash
  virtctl -n ov-demo scp root@rhel8-vm:/var/www/html/repos.tar.gz .
  ```

- 다시 다른 VM으로 파일 전송

  ```bash
  virtctl scp -n ov-demo ./repos.tar.gz root@rhel8-vm2:/root/.
  ```

- 출력 메시지 예시)

  ```bash
  [lab-user@bastion ~]$ virtctl scp -n ov-demo ./repos.tar.gz root@rhel8-vm2:/root/.
  
  Warning: Permanently added 'rhel8-vm2.ov-demo' (ECDSA) to the list of known hosts.
  
  no such identity: /home/lab-user/.ssh/bastion_6w2r8: No such file or directory
  
  root@rhel8-vm2.ov-demo's password:
  
  repos.tar.gz
  ```
