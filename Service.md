
# service管理

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
