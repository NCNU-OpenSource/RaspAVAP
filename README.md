# RaspAVAP

你沒猜錯，自爽專案來惹！

## 窮人的防毒AP

對，沒錯，我存在D槽的那些精選影片差點被病毒幹掉。
不想被病毒毀了我珍藏的___(以下自行填空)，錢包又少到買不起防毒的我，只好想辦法做個能防毒的ＷｉｆｉＡＰ惹…

### 系統架構
-  python
-  [squid](http://www.squid-cache.org/)
-  clamav
-  clamav-deamon
-  squidclamav
-  [yara](http://virustotal.github.io/yara/)

### 設計原因

- 非常非常多人只用Windows Defender，甚至直接關掉
- 老人家也不懂"中毒"是什麼，然後電腦就變毒窟了
- 我窮，買不起WifiAP

### 設計概念

- 把 RaspBerry Pi 變成 WifiAP
- 讓 AP 擁有掃描流量的功能
- 因為這樣會被說啥都沒幹，所以加上能夠自己生成 ZeroDay Threat 特徵的功能

### 設備要求

- 一條你家的網路線
- 這台AP
- 好，沒了

### 運作原理和功能

1. 插上網路線開機後，會透過 Hostapd 和 Dnsmasq 啟動 Wifi AP，同時啟動後台的代理服務。
2. 當使用者連上網路並瀏覽時，Http的存取會轉發給後台的Squid，由Squid代為進行封包的收發。[相關資料：通透式代理](http://linux.vbird.org/linux_server/0420squid.php)
4. Squid會掃描收到的封包是否含有病毒，如果有就會引導用戶轉至Error Page提醒用戶有毒。
5. 每天晚上三點會透過 Script 從數個樣本蒐集點下載新發現的樣本，並建立為Yara規則，寫入到clamav內。

### 步驟

去看 Step.MD 吧，我把我受過的苦都放在那裡了。

### 優點

1. 除了盒子都免錢
2. 由於帶有自動生成新樣本特徵的功能，相對能夠提昇針對 ZeroDay Threat 的防護。
3. 代理服務器能夠幫忙 cache 網頁資料，對於帶有重複資源的網站可以加速存取。

### 缺點

1. 開機之後需要暖機數分鐘才能發揮防毒功能
2. 檢測能力終究有限
3. Wifi 連線的品質受限於機體，對上大量連線容易不堪重負然後爆炸

### 原本想加上但是最後沒加的功能

1. Malware Sandbox Analyze
   原本想要加上透過沙箱進行文件分析的功能，但記憶體不夠用阿阿阿阿
2. Auto Sample Send
   其實這功能很棒，但這跟我原本設計的概念有點衝突，就拔掉了
   
### 設計對象

1. 小家庭或個人居住，要有個簡易的WifiAP的
2. 辦公室環境，為了節儉只給你1000元還要你生出Wifi的
3. 老人家什麼都不懂，想要插上去就能動的

### 組別

第七組 106321009 羅罡兆

### 參考資料

[Squid-Cache - Official Wiki](https://wiki.squid-cache.org/)

[SquidClamAV - Official Website](http://squidclamav.darold.net/)

[C-ICAP - Official Wiki](https://sourceforge.net/p/c-icap/wiki/Home/)

[Yabin - Yara Rule Generator](https://github.com/AlienVault-OTX/yabin)

[Your Daily Malware Samples - Malware Samples Fetcher](https://github.com/notnop/Your-daily-malware-samples)

[Raspberry Pi 3 Wifi 架設](http://blog.itist.tw/2016/03/using-raspberry-pi-3-as-wifi-ap-with-raspbian-jessie.html)

[Terminal 28 : Setup SquidClamav](https://terminal28.com/how-to-install-and-configure-squid-proxy-server-clamav-squidclamav-c-icap-server-debian-linux/)

[Linux Digest - Configuration](https://sathisharthars.com/2013/07/31/configuring-proxy-server-with-antivirus-squidclamavsquidguarddansguardian/)

[ClamAV Documentation - Yara Part](https://www.clamav.net/documents/using-yara-rules-in-clamav)
[SquidClamAV 日本語ガイド](http://www.kozupon.pgw.jp/squidclamav/index.html)
[Server World - Configuration](https://www.server-world.info/en/note?os=Ubuntu_17.04&p=squid&f=5)
