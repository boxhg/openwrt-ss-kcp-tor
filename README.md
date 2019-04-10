# openwrt-ss-kcp-tor
run ss&amp;kcp&amp;tor on openwrt, provide a transparent proxy for pc/phone

# Reference 
- tor-router-nexx-wt3020 https://github.com/enovella/tor-router-nexx-wt3020
- openwrt-ss-kcp https://github.com/boxhg/openwrt-ss-kcp

# Port list
- kcptun-client : 12345, (tcp tunnel for ss)
- ss-local : 1080, (Socks5 Proxy for tor)
- Tor-SocksPort: 9050  (Socks5 Proxy for tor)
- Tor-TransPort: 9040  (Transparent Proxy )
- Tor-DnsPort: 9053  (UDP Proxy)

  
The network Traffic is:
    
    client device(pc,mobile) ---> Openwrt { tor:9040 --> ss:1080 --> kcp:12345 } --->  Internet

Dns traffic: 
    
    client's dns query --> Openwrt { Router:53 -> Tor-Dns: 9053} --->  Internet

# Setup

### 1. install & Config kcp, ss on openwrt

See here: 
[Run OpenWrt 18.06.2 with SS+Kcp On VMware ](https://github.com/boxhg/openwrt-ss-kcp/blob/master/VMware-Openwrt.18.06.2-x86.md)

### 2. install tor

```
  opkg install tor
  opkg install tor-geoip
```
### 3.Setup Tor Configuration: /etc/tor/torrc
```
User tor
VirtualAddrNetwork 10.192.0.0/10
AutomapHostsSuffixes .onion,.exit
AutomapHostsOnResolve 1

SocksPort 127.0.0.1:9050
SocksPort 192.168.22.1:9050

TransPort 192.168.22.1:9040 IsolateClientAddr IsolateClientProtocol IsolateDestAddr IsolateDestPort
TransPort 127.0.0.1:9040 IsolateClientAddr IsolateClientProtocol IsolateDestAddr IsolateDestPort

DNSPort 127.0.0.1:9053
DNSPort 192.168.22.1:9053

# GeoIP for stats
GeoIPFile /usr/share/tor/geoip
GeoIPv6File /usr/share/tor/geoip6

# Tweaks
CircuitBuildTimeout 5
KeepalivePeriod 60
NewCircuitPeriod 15
NumEntryGuards 8

Socks5Proxy localhost:1080

ExcludeNodes {us},{hk},{mo},{sg},{th}
ExcludeExitNodes {us},{hk},{mo},{sg},{th}
ExitNodes {ru}

```

### 4. Configure transparent proxy, update firewall custom rules file: /etc/firewall.user
```
iptables -t nat -A PREROUTING -i br-lan -s $(uci get network.lan.ipaddr)/$(ipcalc.sh $(uci get network.lan.ipaddr) $(uci get network.lan.netmask)|grep PREFIX|cut -d "=" -f 2) -d $(uci get network.lan.ipaddr) -j RETURN # DNT
#iptables -t nat -A PREROUTING -i br-lan -p udp --dport 53 -j REDIRECT --to-ports 9053 # DNT
iptables -t nat -A PREROUTING -i br-lan -p tcp --syn -j REDIRECT --to-ports 9040 # DNT

# Drop ICMP # DNT
#iptables -A INPUT -p icmp --icmp-type 8 -j DROP # DNT

# security rules from https://lists.torproject.org/pipermail/tor-talk/2014-March/032507.html # DNT
iptables -A OUTPUT -m conntrack --ctstate INVALID -j DROP # DNT
#iptables -A OUTPUT -m state --state INVALID -j DROP # DNT

# security rules to prevent kernel leaks from link above # DNT
iptables -A OUTPUT ! -o lo ! -d 127.0.0.1 ! -s 127.0.0.1 -p tcp -m tcp --tcp-flags ACK,FIN ACK,FIN -j DROP # DNT
iptables -A OUTPUT ! -o lo ! -d 127.0.0.1 ! -s 127.0.0.1 -p tcp -m tcp --tcp-flags ACK,RST ACK,RST -j DROP # DNT

# disable chrome and firefox udp leaks # DNT
iptables -t nat -A PREROUTING -p udp -m multiport --dport 3478,19302 -j REDIRECT --to-ports 9999 # DNT
iptables -t nat -A PREROUTING -p udp -m multiport --sport 3478,19302 -j REDIRECT --to-ports 9999 # DNT
```

### 5. Openwrt Dns Config 

enter Network－DHCP/DNS:

>General Settings

    DNS Forwarder： 127.0.0.1#9053

>Resolv and Hosts Files:(Must checked)

    Ignore resolve file: checked
