# Preparing
* Installing tools:
```
sudo apt update
sudo apt install squid squid-openssl curl wireshark openssl -y
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

# SSL Squid settings

## Create root CA
* According to corresponding part of `openSSL/report.md`

* Generate Squid CA certificate
```
export name=vericheveo
export group=mstpr251
export prefix="$name-$group"
export email=vericheveo@mail.ru
export ca_crt="$prefix-ca.crt"

openssl genrsa -out "$prefix"-bump.key 4096

openssl req -new -key "$prefix"-bump.key -passin pass:"$name" \
    -subj "/C=RU/ST=Moscow/L=Moscow/O=$name/OU=$name P3_2/CN=$name Squid CA/emailAddress=$email" \
    -addext "basicConstraints=critical,pathlen:0,CA:TRUE" \
    -addext "keyUsage=critical,digitalSignature,keyCertSign,cRLSign" \
    -out "$prefix"-bump.csr

openssl x509 -req -days 365 -CA "$prefix"-ca.crt -CAkey "$prefix"-ca.key \
    -CAcreateserial -CAserial serial -in "$prefix"-bump.csr \
    -out "$prefix"-bump.crt -passin pass:"$name" -copy_extensions copy
```

* Generate certificates chain for Squid
```
cat "$prefix"-bump.crt "$prefix"-ca.crt > "$prefix"-chain.crt
```

## Access limitation with SNI
* Repeat actions from `Basic Squid settings` part.

* Add to `/etc/squid/squid.conf`:
```
acl localnet src 0.0.0.1-0.255.255.255	# RFC 1122 "this" network (LAN)
acl localnet src 10.0.0.0/8		# RFC 1918 local private network (LAN)
acl localnet src 100.64.0.0/10		# RFC 6598 shared address space (CGN)
acl localnet src 169.254.0.0/16 	# RFC 3927 link-local (directly plugged) machines
acl localnet src 172.16.0.0/12		# RFC 1918 local private network (LAN)
acl localnet src 192.168.0.0/16		# RFC 1918 local private network (LAN)
acl localnet src fc00::/7       	# RFC 4193 local private network range
acl localnet src fe80::/10      	# RFC 4291 link-local (directly plugged) machines

acl SSL_ports port 443
acl Safe_ports port 80		# http
acl Safe_ports port 21		# ftp
acl Safe_ports port 443		# https
acl Safe_ports port 70		# gopher
acl Safe_ports port 210		# wais
acl Safe_ports port 1025-65535	# unregistered ports
acl Safe_ports port 280		# http-mgmt
acl Safe_ports port 488		# gss-http
acl Safe_ports port 591		# filemaker
acl Safe_ports port 777		# multiling http

http_access deny !Safe_ports

http_access allow localhost manager
http_access deny manager

http_access allow localhost
http_access deny to_localhost

http_access allow localnet

acl identme ssl::server_name ident.me
acl httpbin ssl::server_name httpbin.org

http_access allow identme
http_access allow httpbin

http_access deny all
http_port 3128 ssl-bump dynamic_cert_mem_cache_size=4MB cert=/squid/vericheveo-mstpr251-chain.crt key=/squid/vericheveo-mstpr251-bump.key generate-host-certificates=on
sslcrtd_program /usr/lib/squid/security_file_certgen -s /var/spool/squid/ssl_db -M 4MB

acl step1 at_step SslBump1
ssl_bump peek step1
ssl_bump splice httpbin
ssl_bump terminate all

refresh_pattern ^ftp:		1440	20%	10080
refresh_pattern -i (/cgi-bin/|\?) 0	0%	0
refresh_pattern .		0	20%	4320
```

* Start squid container:
```
docker run -d --name squid -v .:/squid -p 3128:3128 -it yutony/squid:4.10 \
    squid -f /squid/"$prefix"-acl.conf -NYC
```

* Create empty log file:
```
touch "$prefix"-acl.log
```

* Add it in wireshark
```
sudo wireshark -k -i any
# Edit -> Preferences -> Protocols
```

* Send queries:
```
export proxy="http://127.0.0.1:3128"
SSLKEYLOGFILE="$prefix"-acl.log curl --tlsv1.2 --tls-max 1.2 -v --proxy $proxy https://ident.me
SSLKEYLOGFILE="$prefix"-acl.log curl --tlsv1.2 --tls-max 1.2 -v --proxy $proxy -k https://httpbin.org/get?bio="$name"
```

* Stop Squid:
```
docker rm "$(docker stop squid)"
```

## Traffic sniffing
* Add to `/etc/squid/squid.conf`:
```
acl localnet src 0.0.0.1-0.255.255.255	# RFC 1122 "this" network (LAN)
acl localnet src 10.0.0.0/8		# RFC 1918 local private network (LAN)
acl localnet src 100.64.0.0/10		# RFC 6598 shared address space (CGN)
acl localnet src 169.254.0.0/16 	# RFC 3927 link-local (directly plugged) machines
acl localnet src 172.16.0.0/12		# RFC 1918 local private network (LAN)
acl localnet src 192.168.0.0/16		# RFC 1918 local private network (LAN)
acl localnet src fc00::/7       	# RFC 4193 local private network range
acl localnet src fe80::/10      	# RFC 4291 link-local (directly plugged) machines

acl SSL_ports port 443
acl Safe_ports port 80		# http
acl Safe_ports port 21		# ftp
acl Safe_ports port 443		# https
acl Safe_ports port 70		# gopher
acl Safe_ports port 210		# wais
acl Safe_ports port 1025-65535	# unregistered ports
acl Safe_ports port 280		# http-mgmt
acl Safe_ports port 488		# gss-http
acl Safe_ports port 591		# filemaker
acl Safe_ports port 777		# multiling http

http_access deny !Safe_ports

http_access allow localhost manager
http_access deny manager

http_access allow localhost
http_access deny to_localhost

http_access allow localnet

http_access deny all

http_port 3128 ssl-bump dynamic_cert_mem_cache_size=4MB cert=/squid/vericheveo-mstpr251-chain.crt key=/squid/vericheveo-mstpr251-bump.key generate-host-certificates=on
sslcrtd_program /usr/lib/squid/security_file_certgen -s /var/spool/squid/ssl_db -M 4MB

acl httpbin ssl::server_name httpbin.org
http_access allow httpbin
ssl_bump bump httpbin
sslproxy_cert_error allow httpbin
ssl_bump stare httpbin

refresh_pattern ^ftp:		1440	20%	10080
refresh_pattern -i (/cgi-bin/|\?) 0	0%	0
refresh_pattern .		0	20%	4320
```

* Run Squid:
```
docker run -d --name squid -v .:/squid -p 3128:3128 -it yutony/squid:4.10 \
    squid -f /squid/$prefix-bump.conf -NYC
```

* Create empty log file:
```
touch "$prefix"-bump.log
```

* Add it in wireshark
```
sudo wireshark -k -i any
# Edit -> Preferences -> Protocols
```

* Send queries:
```
SSLKEYLOGFILE="$prefix"-bump.log curl --tlsv1.2 --tls-max 1.2 -v --proxy "$proxy" -k https://httpbin.org/get?bio="$name"
```

* Stop Squid:
```
docker rm "$(docker stop squid)"
```