# sshd  

- システム標準が存在している  
  ```/usr/sbin/sshd```  
  開始する前に鍵の生成と設定の変更が必要  
  (更にファイアウォール設定が必要)  

  ``` sh
  localhost ~ # ssh-keygen -A
  ```

  /etc/ssh/sshd_config  
  
  ```sh sshd_config
  # Force protocol v2 only
  Protocol 2
  Port 22   # add


  # /etc is read-only.  Fetch keys from stateful partition
  # Not using v1, so no v1 key
  HostKey /etc/ssh/ssh_host_rsa_key       #fix
  HostKey /etc/ssh/ssh_host_ed25519_key   #fix

  PermitRootLogin yes
  PasswordAuthentication no
  UsePAM yes
  PrintMotd no
  PrintLastLog no
  UseDns no
  Subsystem sftp internal-sftp
  # Make DUT responsible to keep connection to server alive for at least half
  # a day, even if network is down. We don't care about leaking/ghost connections
  # as this is the config for the DUT which gets rebooted periodically.
  # Ping ssh client/autotest server once every 60 seconds.
  ClientAliveInterval 60
  # Do this 720 times for 12 hours.
  ClientAliveCountMax 720
  # Ignore temporary network outages.
  TCPKeepAlive no
  ```
