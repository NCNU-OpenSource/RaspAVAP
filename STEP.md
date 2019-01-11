# 實作過程

欸嘿，惡意賣萌

## WifiAP架設工作

1.sudo apt install hostapd dnsmasq

2.edit /etc/dhcpcd.conf
  - interfaces wlan0
  - static_ip=192.168.0.1  `或是你要的主機IP`
  - nohook wpa_supplicant
  > 注意！新版 RaspberryPi 須加上 nohook wpa_supplicant 以防止 wlan0 無法加上固定IP
  
3.edit /etc/dnsmasq.conf
  - interface=wlan0
  - listen-address=192.168.0.1 `主機IP`
  - bind-dynamic
  > 注意！請不要使用參考資料的 bind-interface，容易導致開機後 dnsmasq 無法啟動
  - server=8.8.8.8 `DNS伺服器`
  - bogus-priv `防止私人網段的封包流入BBI中`
  - dhcp-range=192.168.0.10,192.168.0.50,8h `網段範圍 & 地址租期`

4.edit /etc/resolv.dnsmasq 
  - nameserver 192.168.0.1 `主機IP`
  
5.edit /etc/hostapd/hostapd.conf
  - interface=wlan0
  - driver=nl80211 `驅動`
  - ssid=AVAP `廣播名稱`
  - hw_mode=g `指定協議為802.11g`
  - channel=11 `指定廣播頻道為11`
  - macaddr_acl=0 `允許外部MAC連線`
  - auth_algs=1 `指定只接受WPA`
  - ignore_broadcast_id=0 `關閉隱藏ssid`
  - wpa=2 `指定只接受WPA2`
  - wpa_passphrase=avap-pass `Wifi密碼`
  - wpa_key_mgnt=WPA_PSK `指定算法為WPA-PSK`
  - wpa_pairwise=CCMP TKIP `指定加密密鑰`
  
6.Install RaspAP Control WebUI  `非必須`
  - sudo apt install git lighttpd vnstat
  - sudo lighttpd-enable-mod fastcgi-php
  - add www-data to sudoer
  > 注意！請使用Visudo進行修改，直接修改容易導致sudo毀損，如果已經發生，請拔出SD卡並插入至另一台電腦，並刪除修改之全部內容，即可恢復。
  - sudo git clone https://github.com/billz/raspap-webgui /var/www/html
  - sudo chown -R www-data:www-data /var/www/html
  - sudo mv /var/www/html/raspap.php /etc/raspap/
  - sudo chown -R www-data:www-data /etc/raspap
  - sudo mv /var/www/html/installers/*log.sh /etc/raspap/hostapd 
  
7.Restart and Testing Your AP

## Squid 安裝

1. sudo apt install squid ( 可使用自行編譯版，但怕麻煩的請直接用官方版本 )

2. edit /etc/squid/squid.conf
  - cache_dir ufs (size:MB) 16 256 `快取位置`
  - http_port 3128 intercept `通透代理`
  - acl CONNECT method CONNECT
  - acl LAN src 192.168.0.0/24 `網段範圍`
  - http_access allow LAN `允許網段內連線`
  - visible_hostname RaspAVAP
  
3. 確保 iptables 有以下規則，使流量能通過 Squid
  - iptables -t nat -A PREROUTING -i wlan0 -p tcp --dport 80 -j REDIRECT --to-port 3128
  - iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

## ClamAV server 安裝

1. sudo apt install clamav clamdscan clamav-deamon

好，沒了

~~欸嘿☆~~

## C-ICAP 安裝

1. sudo wget https://sourceforge.net/projects/c-icap/files/c-icap/0.5.x/c_icap-0.5.5.tar.gz/download
> 注意！由於我們需要直接設定C-ICAP的位置，因此請使用自行編譯版，不要偷懶上apt抓

2. sudo apt install gcc automake libtool

3. sudo tar -xvf c_icap-0.5.5.tar.gz

4. cd c_icap-0.5.5

5. sudo ./configure --prefix=/usr/local/c-icap --enable-large-files `指定ICAP位置`

6. sudo make

7. sudo make install

8. edit /usr/local/c-icap/etc/c-icap.conf

  - ServerAdmin root@localhost
  - ServerName RaspAVAP
  - DebugLevel 3 `開啟更多Debug報告`
  - Service squidclamav squidclamav.so
  
## SquidClamAV 安裝

1. sudo wget https://https://sourceforge.net/projects/squidclamav/files/squidclamav/6.16/squidclamav-6.16.tar.gz/download
> 注意！跟 ICAP 原因一樣，請不要偷懶用apt

2. tar -xvf squidclamav-6.16.tar.gz

3. cd squidclamav

4. sudo ./configure --with-c-icap=/usr/local/c-icap `ICAP位置`

5. sudo make

6. sudo make install

7. edit /etc/c-icap/squidclamav.conf
> 注意！網路上版本指稱的位置是不正確的，請跟隨本指南設置
  - #redirect `開啟預設移轉頁面`
  
8. edit /etc/squid/squid.conf
  - icap_enable on
  - icap_send_client_ip on
  - icap_send_client_username on
  - icap_client_username_header X-Authenticated-User
  - icap_preview_enable on
  - icap_preview_size 1024
  - icap_service service_req reqmod_precache bypass=1 icap://127.0.0.1:21344/squidclamav
  - adaptation_access service_req allow_all
  - icap_service service_resp respmod_precache bypass=1 icap://127.0.0.1:21344/squidclamav
  - adaptation_access service_resp allow_all
  
9. Add /etc/init.d/c-icap `請設定為executable`

```
#! /bin/sh

### BEGIN INIT INFO
# Provides:          c-icap
# Required-Start:    $network $remote_fs $syslog
# Required-Stop:     $network $remote_fs $syslog
# Should-Start:      $named
# Should-Stop:       $named
# Default-Start:     2 3 4 5
# Default-Stop:      0 1 6
# Short-Description: C-ICAP Server Version 0.1.3
### END INIT INFO

PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin
DAEMON=/usr/local/bin/c-icap
NAME=c-icap
DESC=c-icap

test -x $DAEMON || exit 0

LOGDIR=/var/log/c-icap
PIDFILE=/var/run/c-icap/$NAME.pid
DODTIME=3                   # Time to wait for the server to die, in seconds
                            # If this value is set too low you might not
                            # let some servers to die gracefully and
                            # 'restart' will not work
STARTUPTIME=1               # Time to wait to decide if daemon is up and running

# Include c-icap defaults if available
if [ -f /etc/default/c-icap ] ; then
        . /etc/default/c-icap
fi

check_ctl_dir() {
    # Create the ctl empty dir if necessary
    if [ ! -d /var/run/c-icap ]; then
        mkdir /var/run/c-icap
        chown proxy /var/run/c-icap
        chmod 0755 /var/run/c-icap
    fi
}

# If the daemon is not enabled, give the user a warning and stop.
# Check to create /var/run directory if someone wants to run c-icap
# in debug mode / foreground to test some functions without start it from init.d
if [ "$START" != "yes" ]; then
    check_ctl_dir
    echo "To enable $NAME, edit /etc/default/c-icap and set START=yes"
    exit 0
fi

set -e

running_pid()
{
    # Check if a given process pid's cmdline matches a given name
    pid=$1
    name=$2
    [ -z "$pid" ] && return 1
    [ ! -d /proc/$pid ] &&  return 1
    cmd=`cat /proc/$pid/cmdline | tr "\000" "\n"|head -n 1 |cut -d : -f 1`
    # Is this the expected child?
    [ "$cmd" != "$name" ] &&  return 1
    return 0
}


running()
{
# Check if the process is running looking at /proc
# (works for all users)

    # No pidfile, probably no daemon present
    [ ! -f "$PIDFILE" ] && return 1
    # Obtain the pid and check it against the binary name
    pid=`cat $PIDFILE`
    running_pid $pid $DAEMON || return 1
    return 0
}

force_stop() {
# Forcefully kill the process
    [ ! -f "$PIDFILE" ] && return
    if running ; then
        kill -15 $pid
        # Is it really dead?
        [ -n "$DODTIME" ] && sleep "$DODTIME"s
        if running ; then
            kill -9 $pid
            [ -n "$DODTIME" ] && sleep "$DODTIME"s
            if running ; then
                echo "Cannot kill $LABEL (pid=$pid)!"
                exit 1
            fi
        fi
    fi
    rm -f $PIDFILE
    return 0
}

case "$1" in
  start)
        check_ctl_dir
        echo -n "Starting $DESC: "
        start-stop-daemon --start --quiet --pidfile $PIDFILE \
                --exec $DAEMON -- $DAEMON_OPTS
        [ -n "$STARTUPTIME" ] && sleep "$STARTUPTIME"s
        if running ; then
            echo "$NAME."
        else
            echo " ERROR."
        fi
        ;;
  stop)
        echo -n "Stopping $DESC: "
        start-stop-daemon --stop --quiet --pidfile $PIDFILE \
                --exec $DAEMON
        [ -n "$DODTIME" ] && sleep "$DODTIME"s
        echo "$NAME."
        ;;
  force-stop)
        echo -n "Forcefully stopping $DESC: "
        force_stop
        if ! running ; then
            echo "$NAME."
        else
            echo " ERROR."
        fi
        ;;
  #reload)
        #
        #       If the daemon can reload its config files on the fly
        #       for example by sending it SIGHUP, do it here.
        #
        #       If the daemon responds to changes in its config file
        #       directly anyway, make this a do-nothing entry.
        #
        # echo "Reloading $DESC configuration files."
        # start-stop-daemon --stop --signal 1 --quiet --pidfile \
        #       /var/run/$NAME.pid --exec $DAEMON
          #;;
  force-reload)
        #
        #       If the "reload" option is implemented, move the "force-reload"
        #       option to the "reload" entry above. If not, "force-reload" is
        #       just the same as "restart" except that it does nothing if the
        #   daemon isn't already running.
        # check wether $DAEMON is running. If so, restart
        start-stop-daemon --stop --test --quiet --pidfile \
                /var/run/$NAME.pid --exec $DAEMON \
        && $0 restart \
        || exit 0
        ;;
  restart)
        check_ctl_dir
        echo -n "Restarting $DESC: "
        if running ; then
                start-stop-daemon --stop --quiet --pidfile \
                        $PIDFILE --exec $DAEMON
        fi
        [ -n "$DODTIME" ] && sleep $DODTIME
        start-stop-daemon --start --quiet --pidfile \
                $PIDFILE --exec $DAEMON -- $DAEMON_OPTS
        echo "$NAME."
        ;;
  status)
    echo -n "$NAME is "
    if running ;  then
        echo "running"
    else
        echo " not running."
        exit 1
    fi
    ;;
  *)
        N=/etc/init.d/$NAME
        # echo "Usage: $N {start|stop|restart|reload|force-reload}" >&2
        echo "Usage: $N {start|stop|restart|force-reload|status|force-stop}" >&2
        exit 1
        ;;
esac

exit 0
```

10. Add /etc/default/c-icap

```
# sourced by /etc/init.d/c-icap
# installed at /etc/default/c-icap by the maintainer scripts

# Should c-icap daemon run automatically on startup? (default: no)
START=yes

# Additional options that are passed to the Daemon.
DAEMON_OPTS=""
```

11. start squid & c-icap service and start testing

## crontab 設定

由於系統包含生成資料庫的功能，因此需要額外設置指令，請留意
> 注意！以下Crontab指令皆基於root權限，請用sudo crontab -e 打開

```
0 3 * * * python /home/pi/signgen.py #執行樣本採集與資料庫生成
0 6 * * * \cp /home/pi/custom_rule.yara /var/lib/clamav/custom_rule.yara #將生成之資料庫寫入clamav之病毒庫位置
0 7 * * * clamdscan --reload #載入新資料庫
0 15 * * 4 rm -r /home/pi/samples/* #每周定期清理樣本檔
```
