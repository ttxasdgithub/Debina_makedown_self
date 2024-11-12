iptable_sh


```
#!/bin/sh

# ---------------------------------------------------------------------------------------
# 適用環境：單機防火牆－僅使用一張網卡，無NAT功能
# 圖例：
# ----          ----
# |路|                 |伺|
# |由|  <---->  |服|
# |器|                 |器|
# ----          ----
# 網段說明：
# (1)1.2.3.0/24的意思是：1.2.3.0所屬的網路，子網路遮罩為255.255.255.0（24/8=3），整個網段的範圍是1.2.3.0到1.2.3.255。
# (2)1.2.3.0/16的意思是：1.2.3.0所屬的網路，子網路遮罩為255.255.0.0（16/8=2），整個網段的範圍是1.2.0.0到1.2.255.255。
# (3)1.2.3.0/8的意思是：1.2.3.0所屬的網路，子網路遮罩為255.0.0.0（8/8=1），整個網段的範圍是1.0.0.0到1.255.255.255。
# 使用說明：
# (1)將本檔放置於： /usr/local/bin
# (2)變更檔案權限： chmod +x /usr/local/bin/firewall.sh
# (3)設定開機啟動：於 /etc/rc.local 新增 /usr/local/bin/firewall.sh start
# ---------------------------------------------------------------------------------------

## -------------------------- 防火牆規則設定區段 -------------------------- ##
# 設定禁止連線的IP，可使用空白分隔多個IP，也可以使用網段的寫法
BADIPS="198.108.0.0/16 141.212.0.0/16 114.119.0.0/16 113.215.0.0/16 223.93.0.0/16 213.219.0.0/16 176.226.0.0/16 193.188.0.0/16"


# 設定不可能出現的私有IP，請依照您的環境自行刪減網段
# 若您的IP為 192.168.x.x ，請刪除 192.168.0.0/16
IMPOSSIBLE_IPS="10.0.0.0/8 192.168.0.0/16 172.16.0.0/16"



# 允許對內連線的 TCP 通訊埠
# (1)基本格式：通訊埠1 通訊埠2（中間用空白分隔），也可以使用其名稱（小寫）進行設定；如『ssh 25 80』。
# (2)連續埠號：使用『:』指定開放連續的埠號；如『7000:7009』共10個通訊埠。
# (3)指定來源：指定某個通訊埠只能供特定IP存取，IP也可以使用網段或網域名稱的寫法，且若前面加上『!』代表相反的意思，也就是不允許該IP連線，但是其他皆可連線；
#    如『ssh,192.168.0.1 ssh,192.168.0.0/16 smtp,!192.168.0.1』。
# 注意：若前後設定有重疊，則系統將以後設定者為準。
# 給Hinet動態IP能使用SSH， 22,1.0.0.0/8  就代表 1.*.*.*  都可用

IN_TCP_PORTALLOWED="22,你的IP 888,你的IP 80 443"



# 允許對內連線的 UDP 通訊埠
IN_UDP_PORTALLOWED=""

# 允許對內連線的 ICMP 類型
#  0：回應 PING 封包
#  3：無法到達目的端
#  4：要求來源端暫時停止發送封包
#  5：重新導向路由錯誤的封包
#  8：要求對方回應的 PING 封包
# 11：告知來源端其封包已逾時
# 12：告知來源端其封包的參數錯誤
# 13：請求對方送出時間標記以便同步時間
# 14：回應時間標記
# 15：請求對方傳送網路訊息
# 16：回應網路資訊
# 17：請求對方傳送子網路遮罩的設定資料
# 18：回應子網路遮罩的設定資料
IN_ICMP_ALLOWED="8"
# ---------------------------------------------------------------------------------------
# 以下為運作程式碼，如果對於 shell script 不太熟悉，建議您不要隨意更動

# 載入 ip_tables 模組
#modprobe ip_tables
# 載入連線追蹤模組
#modprobe ip_conntrack
# 載入追蹤 ftp 模組
#modprobe ip_conntrack_ftp
# 載入追蹤 irc 模組
#modprobe ip_conntrack_irc
# 清除目前 iptables 所有表格內的規則
echo -n "Initiating iptables..."
iptables -P INPUT ACCEPT
iptables -P OUTPUT ACCEPT
iptables -P FORWARD ACCEPT
iptables -t filter -F
iptables -t nat -F
iptables -t filter -X
iptables -t nat -X
echo "OK"
## ------------------------------- 測試模式 ------------------------------- ##
# 若加上『start』參數，則『$skiptest』變數將設定為『1』，將會跳過測試模式
[ "$1" = "start" ] && skiptest="1"
## ------------------------ 設定核心的安全相關參數 ------------------------ ##
# 忽略發送至廣播位址的 PING 封包
echo 1 > /proc/sys/net/ipv4/icmp_echo_ignore_broadcasts
# 忽略有問題的 ICMP 封包
echo 1 > /proc/sys/net/ipv4/icmp_ignore_bogus_error_responses
# 阻擋來源路由封包
  for i in /proc/sys/net/ipv4/conf/*/accept_source_route; do
	echo "0" > $i
  done
# 阻擋 ICMP Redirect 封包
  for i in /proc/sys/net/ipv4/conf/*/accept_redirects; do
	echo "0" > $i
  done
# 禁止送出 ICMP Redirect 封包
  for i in /proc/sys/net/ipv4/conf/*/send_redirects; do
	echo "0" > $i
  done
# 開啟核心的逆向路徑過濾功能
  for i in /proc/sys/net/ipv4/conf/*/rp_filter; do
	echo "1" > $i
  done
# 使用 SYN cookies 功能防止 SYN Flood 攻擊
echo 1 > /proc/sys/net/ipv4/tcp_syncookies
# 縮短 TCP 連線的重試次數與逾時時間
echo 3 > /proc/sys/net/ipv4/tcp_retries1
echo 30 > /proc/sys/net/ipv4/tcp_fin_timeout
echo 1400 > /proc/sys/net/ipv4/tcp_keepalive_time
echo 0 > /proc/sys/net/ipv4/tcp_window_scaling
echo 0 > /proc/sys/net/ipv4/tcp_sack
echo 0 > /proc/sys/net/ipv4/tcp_timestamps
# ---------------------------------------------------------------------------------------
## ---------------------- 預設阻擋所有連線的基本原則 ---------------------- ##
# 輸出開始設定訊息
echo -n "Setting firewall rules......" 
# 設定 INPUT 鏈的預設過濾規則，凡封包不符合 INPUT 鏈中的規則者，皆予以丟棄
iptables -P INPUT DROP
# 設定 OUTPUT 鏈的預設過濾規則，凡封包不符合 OUTPUT 鏈中的規則者，皆予以丟棄
iptables -P OUTPUT ACCEPT
# 設定 FORWARD 鏈的預設過濾規則，凡封包不符合 FORWARD 鏈中的規則者，皆予以丟棄
iptables -P FORWARD DROP
## ------------------ 設定本機內部 lookback 連線相關規則 ------------------ ##
# 從 lookback 介面輸入的封包允許通過
iptables -A INPUT -i lo -j ACCEPT
## -------------------------- 阻擋可疑狀態的封包 -------------------------- ##
# 新增一個名為 BADPKT 的新鏈，準備將所有可疑的封包都交由 BADPKT 處理
iptables -N BADPKT
# 丟棄所有進入 BADPKT 鏈的封包
iptables -A BADPKT -j DROP
# 將可疑封包交由 BADPKT 鏈處理
iptables -A INPUT -m state --state INVALID -j BADPKT
iptables -A INPUT -p tcp ! --syn -m state --state NEW -j BADPKT
iptables -A INPUT -p tcp --tcp-flags ALL NONE -j BADPKT
iptables -A INPUT -p tcp --tcp-flags SYN,FIN SYN,FIN -j BADPKT
iptables -A INPUT -p tcp --tcp-flags SYN,RST SYN,RST -j BADPKT
iptables -A INPUT -p tcp --tcp-flags FIN,RST FIN,RST -j BADPKT
iptables -A INPUT -p tcp --tcp-flags ACK,FIN FIN -j BADPKT
iptables -A INPUT -p tcp --tcp-flags ACK,URG URG -j BADPKT
iptables -A INPUT -p tcp --tcp-flags ACK,PSH PSH -j BADPKT
iptables -A INPUT -p tcp --tcp-flags ALL FIN,URG,PSH -j BADPKT
iptables -A INPUT -p tcp --tcp-flags ALL SYN,RST,ACK,FIN,URG -j BADPKT
iptables -A INPUT -p tcp --tcp-flags ALL ALL -j BADPKT
iptables -A INPUT -p tcp --tcp-flags ALL FIN -j BADPKT

# 允許已建立連線和回應的封包通過
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
## -------------------------- 阻擋特定 IP 的連線 -------------------------- ##
# 新增一個名為 BADIP 的新鏈
iptables -N BADIP

# 丟棄所有進入 BADIP 鏈的封包
iptables -A BADIP -j DROP

# 阻擋特定 IP 的連線
for ip in $BADIPS $IMPOSSIBLE_IPS ; do
   iptables -A INPUT -s $ip -j BADIP
done
## -------------------------- 允許特定 IP 的連線 -------------------------- ##
# 允許特定 TCP 埠號的對內新連線
# 若不希望開放給所有人，希望限制特定IP才能使用，只要加上參數 -s 即可
for i in $IN_TCP_PORTALLOWED ; do
   IFS=','
   set $i
   unset IFS ipt_option

   port="$1"
   [ -n "$2" ] && ipt_option="-s `echo $2 | sed 's/^!/! /'`"

	iptables -A INPUT -p tcp $ipt_option --dport $port --syn -m state --state NEW -j ACCEPT
done

# 允許特定 UDP 埠號的對內新連線
# 若是需要開放使用UDP通訊協定的通訊埠，只要將 "-p tcp" 改為 "-p udp"，然後刪除 "--syn" 參數即可
for i in $IN_UDP_PORTALLOWED ; do
   IFS=','
   set $i
   unset IFS ipt_option

   port="$1"
   [ -n "$2" ] && ipt_option="-s `echo $2 | sed 's/^!/! /'`"

	iptables -A INPUT -p udp $ipt_option --dport $port -m state --state NEW -j ACCEPT
done

# 允許特定 ICMP 類型封包進入
# 同理 ICMP 協定則改用 "-p icmp"，不過ICMP協定中沒有通訊埠存在，只有類型之別，所以必須將 "--dport" 參數改為 "--icmp-type"
for i in $IN_ICMP_ALLOWED ; do
   IFS=','
   set $i
   unset IFS ipt_option

   type="$1"
   [ -n "$2" ] && ipt_option="-s `echo $2 | sed 's/^!/! /'`"
   
	iptables -A INPUT -p icmp $ipt_option --icmp-type $type -m state --state NEW -j ACCEPT
done

# 不管制對外連線，所以開放所有對外連線
iptables -A OUTPUT -m state --state NEW -j ACCEPT
## ------------------------------- 結束訊息 ------------------------------- ##
echo "OK"
# -----------------------------------------------------------------------------
# 如果 $skiptest 變數的值不為『1』，也就是未使用『start』參數，則７秒後自動清除防火牆規則，可避免遠端操作將自己阻擋在防火牆外
if [ "$skiptest" = "1" ]; then exit ;fi

echo -e "\n     TEST MODE"
echo -n "All chains will be cleaned after 7 sec."

i=1; while [ "$i" -le "7" ]; do
   echo -n "."
   i=`expr $i + 1`
   sleep 1
done

echo -en "\nFlushing ruleset..."
iptables -P INPUT ACCEPT
iptables -P OUTPUT ACCEPT
iptables -P FORWARD ACCEPT
iptables -t filter -F
iptables -t nat -F
iptables -t filter -X
iptables -t nat -X
echo "OK"
# -----------------------------------------------------------------------------
# ---------------------------------------------------------------------------------------
```
