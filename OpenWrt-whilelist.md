### use firewall rules to setup an whitelist ip output for Openwrt

1. run "vi /etc/config/firewall", open the firewall config
2. append the whilte-ip at the file end
~~~
...

config rule
   option src              lan
   option dest             wan
   
   #your while-list ip    
   option dest_ip          123.123.123.123
   option target           ACCEPT
   
config rule
   option src              lan
   option dest             wan
   option target           REJECT
~~~

3. save the file
4. run "service firewall restart", restart the firewall
5. that's all, enjoy it.
