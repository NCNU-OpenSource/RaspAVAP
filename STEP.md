# 實作過程

欸嘿，惡意賣萌

## WifiAP架設工作

1.sudo apt install hostapd dnsmasq
2.edit /etc/dhcpcd.conf
  - interfaces wlan0
  - static_ip.＝192.168.0.1

## Squid 安裝

1. sudo apt install squid ( 可使用自行編譯版，但怕麻煩的請直接用官方版本 )
2. edit with code below
  - cache_dir ufs (size:MB)
