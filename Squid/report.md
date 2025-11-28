# Preparing
* Installing tools:
```
sudo apt update
sudo apt install squid curl wireshark -y
```

# Basic Squid settings

* Create Squid config backup:
```
sudo cp /etc/squid/squid.conf /etc/squid/squid.conf.backup
```

* Remove comments in new file:
```
sudo grep -Ev '^#|^$' /etc/squid/squid.conf > /tmp/squid_temp.conf && \
sudo mv /tmp/squid_temp.conf /etc/squid/squid.conf
```

* Allow all localnet connections:
```
# ... Previous rules ...

http_access deny !Safe_ports
http_access deny CONNECT !SSL_ports
http_access allow localhost manager
http_access deny manager

include /etc/squid/conf.d/*.conf

# !!! ADD HERE !!!
http_access allow localhost
http_access allow localnet

# ... Next rules ...
```

* Disable caching in the end of config:
```
maximum_object_size 0 KB
refresh_pattern -i . 0 0% 0
cache deny all
```

* Check rules, reconfigure squid, check status:
```
sudo pkill -9 squid
sudo squid -z
sudo systemctl restart squid
sudo squid -k reconfigure
sudo systemctl reload squid
sudo squid -k parse
```

## Access limitation

* Add to the end of file `/etc/squid/squid.conf`:
```
# ... Previous rules ...

# Set ACL for forbidden domain
acl blocked_domain dstdomain ident.me
# Set ACL for available URL
acl allowed_url url_regex -i httpbin.org/get.*bio=vericheveo

http_access deny !Safe_ports
http_access deny CONNECT !SSL_ports
http_access allow localhost manager
http_access deny manager

# Access rule 1 (IN ORDER!!!)
http_access deny blocked_domain
# Access rule 2 (IN ORDER!!!)
http_access allow allowed_url

include /etc/squid/conf.d/*.conf
# ... Next rules ...
```

* Check access to verify config is right: 
```
# Req to FORBIDDEN resource
curl -v --proxy http://localhost:3128 -H "User-Agent: vericheveo" http://ident.me
```
```
(base) ➜  Squid git:(development) ✗ curl -v --proxy http://localhost:3128 http://ident.me
*   Trying 127.0.0.1:3128...
* Connected to (nil) (127.0.0.1) port 3128 (#0)
> GET http://ident.me/ HTTP/1.1
> Host: ident.me
> User-Agent: curl/7.81.0
> Accept: */*
> Proxy-Connection: Keep-Alive
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Access-Control-Allow-Origin: *
< Alt-Svc: h3=":443"; ma=3600
< Cache-Control: no-cache, no-store, must-revalidate
< Date: Thu, 27 Nov 2025 17:02:00 GMT
< Content-Length: 12
< Content-Type: text/plain; charset=utf-8
< X-Cache: MISS from jrvv-desktop
< X-Cache-Lookup: MISS from jrvv-desktop:3128
< Via: 1.1 jrvv-desktop (squid/5.9)
< Connection: keep-alive
< 
* Connection #0 to host (nil) left intact
72.56.67.251%      
```

```
# Req to AVAILABLE resource
curl -v --proxy http://localhost:3128 "http://httpbin.org/get?bio=vericheveo"
```
```
*   Trying 127.0.0.1:3128...
* Connected to (nil) (127.0.0.1) port 3128 (#0)
> GET http://ident.me/ HTTP/1.1
> Host: ident.me
> Accept: */*
> Proxy-Connection: Keep-Alive
> User-Agent: vericheveo
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 403 Forbidden
< Server: squid/5.9
< Mime-Version: 1.0
< Date: Thu, 27 Nov 2025 22:29:47 GMT
< Content-Type: text/html;charset=utf-8
< Content-Length: 3060
< X-Squid-Error: ERR_ACCESS_DENIED 0
< Vary: Accept-Language
< Content-Language: en
< X-Cache: MISS from jrvv-desktop
< X-Cache-Lookup: NONE from jrvv-desktop:3128
< Via: 1.1 jrvv-desktop (squid/5.9)
< Connection: keep-alive
```

* Check resources ip:
```
dig +short ident.me | head -1
# 65.108.151.63
dig +short httpbin.org | head -1
# 3.232.74.21
```

* Run Wireshark for any interface:
```
sudo wireshark -k -i any
```

* Filter for Wireshark:
```
(tcp) and
(tcp.port == 3128 or
  (tcp.port == 80) or
  ((http.host contains "ident.me" or http.host contains "httpbin.org") and (ip.addr == 65.108.151.63 or ip.addr == 3.232.74.21))
) and 
!tcp.analysis.flags
```

* Or run with BPF filter (not reccomended):
```
IDENT_IP=dig +short ident.me | head -1
HTTPBIN_IP=dig +short httpbin.org | head -1
sudo wireshark -k -i any -f "tcp and (port 3128 or (port 80 and (host $IDENT_IP or host $HTTPBIN_IP)))"
```

## Header modification

* Remove previous config:
```
sudo rm /etc/squid/squid.conf && sudo cp /etc/squid/squid.conf.backup /etc/squid/squid.conf
```

* Add path to resources list to update user agent for:
```
acl ua_rewrite dstdomain "/etc/squid/rewrite.acl"
```

* Change user agent header in all requests:
```
# ... Previous rules ...
http_access allow localhost
http_access allow localnet

# Change user agent header in all requests
request_header_access User-Agent deny ua_rewrite
request_header_replace User-Agent vericheveo

http_access deny all
# ... Next rules ...
```

* Create `/etc/squid/rewrite.acl` file with host name:
```
sudo echo "httpbin.org" > /etc/squid/rewrite.acl
```

* Send request:
```
curl -v --proxy http://localhost:3128 http://httpbin.org/ip
```
```
*   Trying 127.0.0.1:3128...
* Connected to (nil) (127.0.0.1) port 3128 (#0)
> GET http://httpbin.org/ip HTTP/1.1
> Host: httpbin.org
> User-Agent: curl/7.81.0
> Accept: */*
> Proxy-Connection: Keep-Alive
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Date: Thu, 27 Nov 2025 23:38:02 GMT
< Content-Type: application/json
< Content-Length: 44
< Server: gunicorn/19.9.0
< Access-Control-Allow-Origin: *
< Access-Control-Allow-Credentials: true
< X-Cache: MISS from jrvv-desktop
< X-Cache-Lookup: MISS from jrvv-desktop:3128
< Via: 1.1 jrvv-desktop (squid/5.9)
< Connection: keep-alive
<
```
