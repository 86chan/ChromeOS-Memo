# ChromeOS

## service管理

- サービスはUpstartを採用している。  
    [Upstartのことも調べてみたよ](https://qiita.com/miyuki_samitani/items/ff81846f44c083564dbc)
  - 設定ファイルの場所は ```/etc/init/```
  - コマンドは ```initctl```

    ``` sh
    localhost ~ # initctl --help
    Usage: initctl [OPTION]... COMMAND [OPTION]... [ARG]...

    Options:
        --session               use D-Bus session bus to connect to init daemon
                                    (for testing)
        --system                use D-Bus system bus to connect to init daemon
        --dest=NAME             destination well-known name on D-Bus bus
    -q, --quiet                 reduce output to errors only
    -v, --verbose               increase output to include informational messages
        --help                  display this help and exit
        --version               output version information and exit

    For a list of commands, try `initctl help'.

    Report bugs to <upstart-devel@lists.ubuntu.com>
    ```

  - 設定例  
    sshd.conf

    ```sh sshd.conf
    # ssh - OpenBSD Secure Shell server
    #
    # The OpenSSH server provides secure shell access to the system.

    description     "OpenSSH server"

    # start on filesystem or runlevel [2345]
    start on started system-services
    stop on runlevel [!2345]

    respawn
    respawn limit 10 5
    umask 022

    env SSH_SIGSTOP=1
    expect stop

    # 'sshd -D' leaks stderr and confuses things in conjunction with 'console log'
    console none

    pre-start script
        test -x /usr/sbin/sshd || { stop; exit 0; }
        test -e /etc/ssh/sshd_not_to_be_run && { stop; exit 0; }

        mkdir -p -m 0755 /var/run/sshd
    end script

    # if you used to set SSHD_OPTS in /etc/default/ssh, you can change the
    # 'exec' line here instead
    exec /usr/sbin/sshd -D
    ```
  
    start on : 開始するイベント  
    stop on : 停止するイベント  
    pre-start script : メインの処理前の準備処理等

- ファイアウォールについて
  ファイアウォールの管理はiptablesを採用しているが、```/etc/sysconfig/iptables```等に設定ファイルが用意されていない。  
  ```/etc/init/iptables.conf```に記載され、Upstartで設定している模様。  
  なのでこのファイルに追記するか同様に別途作成する。  
  /etc/init/iptables.conf

  ```sh iptables.conf
  # Copyright (c) 2011 The Chromium OS Authors. All rights reserved.
  # Use of this source code is governed by a BSD-style license that can be
  # found in the LICENSE file.

  description     "Set iptables policies and add rules"
  author          "chromium-os-dev@chromium.org"

  start on starting network-services
  task

  script
  {
    iptables -P INPUT DROP -w
    iptables -P FORWARD DROP -w
    iptables -P OUTPUT DROP -w

    # Accept everything on the loopback
    iptables -I INPUT -i lo -j ACCEPT -w
    iptables -I OUTPUT -o lo -j ACCEPT -w

    # Accept return traffic inbound
    iptables -I INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT -w

    # Accept icmp echo (NB: icmp echo ratelimiting is done by the kernel)
    iptables -A INPUT -p icmp -j ACCEPT -w

    # Accept new and return traffic outbound
    iptables -I OUTPUT -m state --state NEW,ESTABLISHED,RELATED -j ACCEPT -w

    # Accept inbound mDNS traffic
    iptables -A INPUT -p udp --destination 224.0.0.251 --dport 5353 -j ACCEPT -w

    # Accept inbound SSDP traffic
    iptables -A INPUT -p udp --destination 239.255.255.250 --dport 1900 -j ACCEPT -w

    # netfilter-queue-helper is used for Linux < 3.12.
    # For Linux >= 3.12, conntrackd.conf is used instead.
    if [ -e /usr/sbin/netfilter-queue-helper ]; then
      . /usr/sbin/netfilter-common

      # Filter outgoing SSDP traffic for the DIAL protocol through a user-space
      # filter (netfilter-queue-helper) which will open up a port for reply
      # traffic.
      iptables -I OUTPUT -p udp --destination 239.255.255.250 --dport 1900 \
          -j NFQUEUE --queue-num ${NETFILTER_OUTPUT_NFQUEUE} -w

      # Ditto for outbound mDNS legacy unicast replies (source port != 5353).
      iptables -I OUTPUT -p udp --destination 224.0.0.251 --dport 5353 \
          -j NFQUEUE --queue-num ${NETFILTER_OUTPUT_NFQUEUE} -w

      # Send incoming UDP traffic (which has not passed any other rules) to the
      # user-space filter to test whether it was a reply to outgoing DIAL protocol
      # traffic.
      iptables -A INPUT -p udp -j NFQUEUE \
          --queue-num ${NETFILTER_INPUT_NFQUEUE} -w
    fi
  } 2>&1 | logger --priority daemon.info -t ${UPSTART_JOB}
  end script
  ```

- sshdについて  
  システム標準が存在している。 ```/usr/sbin/sshd```  
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


