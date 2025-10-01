# Preparing
* Installing tools:
```
sudo apt install wireshark openssl nginx
```

# Generate X.509 certificates chain

## Root CA

### Expected
  * cert name: `vericheveo-mstpr251-ca.crt`
  * key name:  `vericheveo-mstpr251-ca.key`

```
Ключевая пара:
– RSA 4096 бит;
– Зашифрован AES 256, пароль vericheveo;
● Сертификат
– Самоподписной (Issuer = Subject);
– Срок действия 3 года;
– C=RU, ST=Moscow, L=Moscow, O=vericheveo, OU=vericheveo P1_1, CN=vericheveo CA, email=vericheveo@example.com;
– X.509 v3 расширения:
● Basic Constrains:
– Critical
– CA=True (сертификат центра сертификации)
● Key Usage:
– Critical
– Digital Signature (может подписывать данные)
– Certificate Sign (может подписывать другие сертификаты)
– CRL sign (может подписывать списки отозванных сертификатов (CRL)).
```

### Process
* Gen private key
```
openssl genrsa -aes256 -out vericheveo-mstpr251-ca.key 4096
```

* Generate root CA
```
openssl req -new -x509 -days 1095 -sha256 \
  -key vericheveo-mstpr251-ca.key \
  -out vericheveo-mstpr251-ca.crt \
  -subj "/C=RU/ST=Moscow/L=Moscow/O=vericheveo/OU=vericheveo P1_1/CN=vericheveo CA/emailAddress=vericheveo@example.com" \
  -addext "basicConstraints=critical,CA:true" \
  -addext "keyUsage=critical,digitalSignature,keyCertSign,cRLSign"
```

* Check root CA
```
openssl x509 -text -noout -in vericheveo-mstpr251-ca.crt
```

* Text form:
```
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            0f:91:47:47:f3:63:59:dd:4d:2a:04:32:4d:a1:74:58:76:34:e8:f3
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: C = RU, ST = Moscow, L = Moscow, O = vericheveo, OU = vericheveo P1_1, CN = vericheveo CA, emailAddress = vericheveo@example.com
        Validity
            Not Before: Sep 23 01:00:50 2025 GMT
            Not After : Sep 22 01:00:50 2028 GMT
        Subject: C = RU, ST = Moscow, L = Moscow, O = vericheveo, OU = vericheveo P1_1, CN = vericheveo CA, emailAddress = vericheveo@example.com
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (4096 bit)
                Modulus:
                    00:b3:1c:a4:3d:2f:99:6e:c7:64:f6:6f:9f:76:90:
                    ...
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Subject Key Identifier: 
                38:EF:A9:07:A5:88:1D:80:01:F5:DB:E6:F6:95:55:D1:6B:40:F6:D1
            X509v3 Authority Key Identifier: 
                38:EF:A9:07:A5:88:1D:80:01:F5:DB:E6:F6:95:55:D1:6B:40:F6:D1
            X509v3 Basic Constraints: critical
                CA:TRUE
            X509v3 Key Usage: critical
                Digital Signature, Certificate Sign, CRL Sign
    Signature Algorithm: sha256WithRSAEncryption
    Signature Value:
        65:e8:38:6d:3e:00:68:1d:b1:f1:b2:ff:0f:f2:83:9e:c9:d5:
        ...
```

## Inter CA
### Expected

  * cert name: `vericheveo-mstpr251-intr.crt`
  * key name:  `vericheveo-mstpr251-intr.key`

```
Ключевая пара:
– RSA 4096 бит;
– Зашифрован AES 256, пароль vericheveo;ru
● Сертификат
– Подписан корневым сертификатом (Issuer = Root CA, Subject = Intermediate CA);
– Срок действия 1 год;
– C=RU, ST=Moscow, L=Moscow, O=vericheveo, CN=vericheveo Intermediate CA, OU=vericheveo P1_1,
email=vericheveo@example.com;
– X.509 v3 расширения:
● Basic Constrains:
– Critical
– PathLen=0 (промежуточный CA не может выпускать другие промежуточные сертификаты)
– CA=True
● Key Usage:
– Critical
– Digital Signature (может подписывать данные)
– Certificate Sign (может подписывать другие сертификаты)
– CRL sign (может подписывать списки отозванных сертификатов (CRL))
```

### Process
* Generate private key
```
openssl genrsa -aes256 -out vericheveo-mstpr251-intr.key 4096
```

* Generating certificate signing request (CSR) to send it to root CA
```
openssl req -new -sha256 \
  -key vericheveo-mstpr251-intr.key \
  -out vericheveo-mstpr251-intr.csr \
  -subj "/C=RU/ST=Moscow/L=Moscow/O=vericheveo/OU=vericheveo P1_1/CN=vericheveo Intermediate CA/emailAddress=vericheveo@example.com"
```

* Sign inter CA using CSR and root CA
```
openssl x509 -req -in vericheveo-mstpr251-intr.csr \
  -CA vericheveo-mstpr251-ca.crt -CAkey vericheveo-mstpr251-ca.key \
  -CAcreateserial -out vericheveo-mstpr251-intr.crt \
  -days 365 -sha256 \
  -extfile <(printf "basicConstraints=critical,CA:true,pathlen:0\nkeyUsage=critical,digitalSignature,keyCertSign,cRLSign")
```

* Check inter CA
```
openssl x509 -text -noout -in vericheveo-mstpr251-intr.crt
```

* Text form
```
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            72:5d:54:39:fb:6e:87:34:25:43:df:1e:f6:a8:01:38:21:82:0e:73
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: C = RU, ST = Moscow, L = Moscow, O = vericheveo, OU = vericheveo P1_1, CN = vericheveo CA, emailAddress = vericheveo@example.com
        Validity
            Not Before: Sep 23 01:28:37 2025 GMT
            Not After : Sep 23 01:28:37 2026 GMT
        Subject: C = RU, ST = Moscow, L = Moscow, O = vericheveo, OU = vericheveo P1_1, CN = vericheveo Intermediate CA, emailAddress = vericheveo@example.com
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (4096 bit)
                Modulus:
                    00:b6:ba:b9:98:02:34:cd:b7:ed:a0:45:82:11:04:
                    ...
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Basic Constraints: critical
                CA:TRUE, pathlen:0
            X509v3 Key Usage: critical
                Digital Signature, Certificate Sign, CRL Sign
            X509v3 Subject Key Identifier: 
                F3:7C:8E:6C:DE:9A:D4:0F:07:E8:62:ED:8A:30:DA:6E:DC:A3:C0:A9
            X509v3 Authority Key Identifier: 
                38:EF:A9:07:A5:88:1D:80:01:F5:DB:E6:F6:95:55:D1:6B:40:F6:D1
    Signature Algorithm: sha256WithRSAEncryption
    Signature Value:
        a0:75:70:e0:bb:1a:38:18:b0:68:fa:6a:af:7b:be:5d:48:b6:
        ...
```

## Basic CA
### Expected

  * cert name: `vericheveo-mstpr251-basic.crt`
  * key name:  `vericheveo-mstpr251-basic.key`
```
● Ключевая пара:
– RSA 2048 бит;
● Сертификат
– Подписан промежуточным сертификатом;
– Срок действия 90 дней;
– C=RU, ST=Moscow, L=Moscow, O=vericheveo, OU=vericheveo P1_1, CN=vericheveo Basic, email=vericheveo@example.com;
– X.509 v3 расширения:
● Basic Constrains:
– CA=False (конечный сертификат, не может быть центром сертификации)
● Key Usage (Critical):
– Digital Signature (может подписывать данные)
● Extended Key Usage (Critical):

(EKU - расширение X.509v3, которое уточняет для каких целей может использоваться сертификат)

– TLS Web Server Authentication (cертификат может использоваться сервером)
– TLS Web Client Authentication (cертификат может использоваться клиентом)
● Subject Alternative Name: 

(SAN (альтернативные имена субъекта) — расширение X.509v3, в котором перечисляются имена (DNS, IP, email, URI), на которые действителен сертификат)

– basic.vericheveo.ru
– basic.vericheveo.com
```

### Process

* Gen private key
```
openssl genrsa -out vericheveo-mstpr251-basic.key 2048
```

* Create basic.cnf
```
[req]
prompt = no
req_extensions = v3_req
distinguished_name = dn

[dn]
C = RU
ST = Moscow
L = Moscow
O = vericheveo
OU = vericheveo P1_1
CN = vericheveo Basic
emailAddress = vericheveo@example.com

[v3_req]
basicConstraints = critical,CA:FALSE
keyUsage = critical,digitalSignature
extendedKeyUsage = critical,serverAuth,clientAuth
subjectAltName = @alt_names

[alt_names]
DNS.1 = basic.vericheveo.ru
DNS.2 = basic.vericheveo.com
```

* Generating CSR to send it to inter CA
```
openssl req -new -sha256 \
  -key vericheveo-mstpr251-basic.key \
  -out vericheveo-mstpr251-basic.csr \
  -config basic.cnf
```

* Sign basic CA using CSR and inter CA
```
openssl x509 -req -in vericheveo-mstpr251-basic.csr \
  -CA vericheveo-mstpr251-intr.crt -CAkey vericheveo-mstpr251-intr.key \
  -CAcreateserial -out vericheveo-mstpr251-basic.crt \
  -days 90 -sha256 \
  -extfile basic.cnf -extensions v3_req
```

* Check basic CA
```
openssl x509 -text -noout -in vericheveo-mstpr251-basic.crt
```

* Text form:
```
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            48:38:82:e3:ce:bd:b4:5e:e4:43:26:20:87:49:c9:cb:14:21:89:46
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: C = RU, ST = Moscow, L = Moscow, O = vericheveo, OU = vericheveo P1_1, CN = vericheveo Intermediate CA, emailAddress = vericheveo@example.com
        Validity
            Not Before: Sep 23 02:17:18 2025 GMT
            Not After : Dec 22 02:17:18 2025 GMT
        Subject: C = RU, ST = Moscow, L = Moscow, O = vericheveo, OU = vericheveo P1_1, CN = vericheveo Basic, emailAddress = vericheveo@example.com
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:8b:61:b5:db:de:57:48:07:14:c2:83:b0:e0:16:
                    ...
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Basic Constraints: critical
                CA:FALSE
            X509v3 Key Usage: critical
                Digital Signature
            X509v3 Extended Key Usage: critical
                TLS Web Server Authentication, TLS Web Client Authentication
            X509v3 Subject Alternative Name: 
                DNS:basic.vericheveo.ru, DNS:basic.vericheveo.com
            X509v3 Subject Key Identifier: 
                42:49:82:34:54:71:69:1F:C7:94:AF:C0:71:E7:58:DA:74:4D:DE:F9
            X509v3 Authority Key Identifier: 
                F3:7C:8E:6C:DE:9A:D4:0F:07:E8:62:ED:8A:30:DA:6E:DC:A3:C0:A9
    Signature Algorithm: sha256WithRSAEncryption
    Signature Value:
        03:12:63:41:84:d8:3e:43:57:a6:12:54:fe:b7:0f:b1:6c:51:
        ...
```

# Generate X.509 certificates with Certificate Revocation List (CRL)

* Create chain root --> intr --> crl-valid/revoked

* Gen private keys
```
openssl genrsa -out vericheveo-mstpr251-crl-valid.key 2048
openssl genrsa -out vericheveo-mstpr251-crl-revoked.key 2048
```

* Generating CSR to send it to inter CA
```
openssl req -new -sha256 \
  -key vericheveo-mstpr251-crl-valid.key \
  -out vericheveo-mstpr251-crl-valid.csr \
  -config valid.cnf
```
```
openssl req -new -sha256 \
  -key vericheveo-mstpr251-crl-revoked.key \
  -out vericheveo-mstpr251-crl-revoked.csr \
  -config revoke.cnf
```

* valid.cnf:
```
[req]
prompt = no
req_extensions = v3_req
distinguished_name = dn

[dn]
C = RU
ST = Moscow
L = Moscow
O = vericheveo
OU = vericheveo P1_2
CN = vericheveo CRL Valid
emailAddress = vericheveo@example.com

[v3_req]
basicConstraints = critical,CA:FALSE
keyUsage = critical,digitalSignature
extendedKeyUsage = critical,serverAuth,clientAuth
subjectAltName = @alt_names
crlDistributionPoints = URI:http://crl.vericheveo.ru:8080/vericheveo-mstpr251.crl

[alt_names]
DNS.1 = crl.valid.vericheveo.ru
```

* revoke.cnf:
```
[req]
prompt = no
req_extensions = v3_req
distinguished_name = dn

[dn]
C = RU
ST = Moscow
L = Moscow
O = vericheveo
OU = vericheveo P1_2
CN = vericheveo CRL Revoked
emailAddress = vericheveo@example.com

[v3_req]
basicConstraints = critical,CA:FALSE
keyUsage = critical,digitalSignature
extendedKeyUsage = critical,serverAuth,clientAuth
subjectAltName = @alt_names
crlDistributionPoints = URI:http://crl.vericheveo.ru:8080/vericheveo-mstpr251.crl

[alt_names]
DNS.1 = crl.revoked.vericheveo.ru
```

* Sign basic valid and revoked CAs using CSR and inter CA
```
openssl x509 -req -in vericheveo-mstpr251-crl-valid.csr \
  -CA vericheveo-mstpr251-intr.crt -CAkey vericheveo-mstpr251-intr.key \
  -CAcreateserial -out vericheveo-mstpr251-crl-valid.crt \
  -days 90 -sha256 \
  -extfile valid.cnf -extensions v3_req
```
```
openssl x509 -req -in vericheveo-mstpr251-crl-revoked.csr \
  -CA vericheveo-mstpr251-intr.crt -CAkey vericheveo-mstpr251-intr.key \
  -CAcreateserial -out vericheveo-mstpr251-crl-revoked.crt \
  -days 90 -sha256 \
  -extfile revoke.cnf -extensions v3_req
```

* Create CRL and revoke vericheveo-mstpr251-crl-revoked.crt
```
touch index.txt
echo 01 > crlnumber

openssl ca -config ca.cnf -revoke vericheveo-mstpr251-crl-revoked.crt

openssl ca -config ca.cnf -gencrl \
    -cert vericheveo-mstpr251-intr.crt \
    -keyfile vericheveo-mstpr251-intr.key \
    -out vericheveo-mstpr251.crl
```

* Check CRL for the same serial number
* crl file
```
openssl crl -noout -text -in vericheveo-mstpr251.crl
```
Output:
```
...
Revoked Certificates:
    Serial Number: 5CF62D1D1EB160A795887941F7C043FC6E92BA2E
...
```
* revoked crt file
```
openssl x509 -in vericheveo-mstpr251-crl-revoked.crt -text -noout
```
Output:
```
...
Serial Number:
            5c:f6:2d:1d:1e:b1:60:a7:95:88:79:41:f7:c0:43:fc:6e:92:ba:2e
...
```

* Create certs chain:
```
cat vericheveo-mstpr251-intr.crt vericheveo-mstpr251-ca.crt > vericheveo-mstpr251-chain.crt
```

* Check valid and revoked certs:
```
openssl verify -crl_check \
-CRLfile vericheveo-mstpr251.crl \
-CAfile vericheveo-mstpr251-chain.crt \
vericheveo-mstpr251-crl-valid.crt
# vericheveo-mstpr251-crl-valid.crt: OK

openssl verify -crl_check \
-CRLfile vericheveo-mstpr251.crl \
-CAfile vericheveo-mstpr251-chain.crt \
vericheveo-mstpr251-crl-revoked.crt
# error 23 at 0 depth lookup: certificate revoked
```

* Check chain:
```
openssl verify -CAfile vericheveo-mstpr251-chain.crt vericheveo-mstpr251-crl-valid.crt
# vericheveo-mstpr251-crl-valid.crt: OK
```

# Build test environment with OCSP-server (Online Certificate Status Protocol) and generate certificates

## Configure certs

* Generate public keys for certs:
```
# OCSP responder (RSA 4096, AES-256, пароль = "vericheveo")
openssl genrsa -aes256 -out vericheveo-mstpr251-ocsp-resp.key 4096

# Valid server cert (RSA 2048)
openssl genrsa -out vericheveo-mstpr251-ocsp-valid.key 2048

# Revoked server cert (RSA 2048)
openssl genrsa -out vericheveo-mstpr251-ocsp-revoked.key 2048
```

* Config `revoke.cnf` for revoked cert:
```
[req]
prompt = no
req_extensions = v3_req
distinguished_name = dn

[dn]
C = RU
ST = Moscow
L = Moscow
O = vericheveo
OU = vericheveo P1_3
CN = vericheveo OCSP Revoked
emailAddress = vericheveo@example.com

[v3_req]
basicConstraints = critical,CA:FALSE
keyUsage = critical,digitalSignature
extendedKeyUsage = critical,serverAuth,clientAuth
subjectAltName = @alt_names
authorityInfoAccess = OCSP;URI:http://ocsp.vericheveo.ru:2560

[alt_names]
DNS.1 = ocsp.revoked.vericheveo.ru
```

* Config `valid.cnf` for valid cert:
```
[req]
prompt = no
req_extensions = v3_req
distinguished_name = dn

[dn]
C = RU
ST = Moscow
L = Moscow
O = vericheveo
OU = vericheveo P1_3
CN = vericheveo OCSP Valid
emailAddress = vericheveo@example.com

[v3_req]
basicConstraints = critical,CA:FALSE
keyUsage = critical,digitalSignature
extendedKeyUsage = critical,serverAuth,clientAuth
subjectAltName = @alt_names
authorityInfoAccess = OCSP;URI:http://ocsp.vericheveo.ru:2560

[alt_names]
DNS.1 = ocsp.valid.vericheveo.ru
```

* Config `ocsp-resp.cnf` for ocsp cert:
```
[req]
prompt = no
req_extensions = v3_req
distinguished_name = dn

[dn]
C = RU
ST = Moscow
L = Moscow
O = vericheveo
OU = vericheveo P1_3
CN = vericheveo OCSP Responder
emailAddress = vericheveo@example.com

[v3_req]
basicConstraints = critical,CA:FALSE
keyUsage = critical,digitalSignature
extendedKeyUsage = OCSPSigning
```

* Generate CSRs:
```
openssl req -new -sha256 -key vericheveo-mstpr251-ocsp-resp.key \
  -out vericheveo-mstpr251-ocsp-resp.csr -config ocsp-resp.cnf

openssl req -new -sha256 -key vericheveo-mstpr251-ocsp-valid.key \
  -out vericheveo-mstpr251-ocsp-valid.csr -config valid.cnf

openssl req -new -sha256 -key vericheveo-mstpr251-ocsp-revoked.key \
  -out vericheveo-mstpr251-ocsp-revoked.csr -config revoke.cnf
```

* Sign certificates with inter cert:
```
# OCSP responder
openssl x509 -req -in vericheveo-mstpr251-ocsp-resp.csr \
  -CA vericheveo-mstpr251-intr.crt -CAkey vericheveo-mstpr251-intr.key \
  -CAcreateserial -out vericheveo-mstpr251-ocsp-resp.crt \
  -days 365 -sha256 -extfile ocsp-resp.cnf -extensions v3_req

# Валидный
openssl x509 -req -in vericheveo-mstpr251-ocsp-valid.csr \
  -CA vericheveo-mstpr251-intr.crt -CAkey vericheveo-mstpr251-intr.key \
  -CAcreateserial -out vericheveo-mstpr251-ocsp-valid.crt \
  -days 90 -sha256 -extfile valid.cnf -extensions v3_req

# Отозванный
openssl x509 -req -in vericheveo-mstpr251-ocsp-revoked.csr \
  -CA vericheveo-mstpr251-intr.crt -CAkey vericheveo-mstpr251-intr.key \
  -CAcreateserial -out vericheveo-mstpr251-ocsp-revoked.crt \
  -days 90 -sha256 -extfile revoke.cnf -extensions v3_req
```

* Create CRL and revoke vericheveo-mstpr251-ocsp-revoked.crt
```
touch index.txt
echo 01 > crlnumber

openssl ca -config ca.cnf -revoke vericheveo-mstpr251-ocsp-revoked.crt

openssl ca -config ca.cnf -gencrl \
    -cert vericheveo-mstpr251-intr.crt \
    -keyfile vericheveo-mstpr251-intr.key \
    -out vericheveo-mstpr251.crl
```

* Add valid cert to `index.txt`:
```
printf "V\t251228021343Z\t\t65F3F1B1FDC85AF66AC5ABE46F2D42DED4EFFB06\tunknown\t/C=RU/ST=Moscow/L=Moscow/O=vericheveo/OU=vericheveo P1_3/CN=vericheveo OCSP Valid/emailAddress=vericheveo@example.com\n" >> index.txt
```

* Create certs chain:
```
cat vericheveo-mstpr251-intr.crt vericheveo-mstpr251-ca.crt > vericheveo-mstpr251-chain.crt
```

* Run OCSP-responder:
```
openssl ocsp -index index.txt \
  -port 2560 \
  -rsigner vericheveo-mstpr251-ocsp-resp.crt \
  -rkey vericheveo-mstpr251-ocsp-resp.key \
  -CA vericheveo-mstpr251-chain.crt -text
```

* Check OCSP-responder:
```
openssl ocsp -issuer vericheveo-mstpr251-intr.crt \
  -cert vericheveo-mstpr251-ocsp-valid.crt \
  -url http://ocsp.vericheveo.ru:2560 \
  -CAfile vericheveo-mstpr251-chain.crt \
  -text
```
```
...
-----END CERTIFICATE-----
Response verify OK
vericheveo-mstpr251-ocsp-valid.crt: good
	This Update: Sep 29 00:16:02 2025 GMT
```

```
openssl ocsp -issuer vericheveo-mstpr251-intr.crt \
  -cert vericheveo-mstpr251-ocsp-revoked.crt \
  -url http://ocsp.vericheveo.ru:2560 \
  -CAfile vericheveo-mstpr251-chain.crt \
  -text
```
```
...
-----END CERTIFICATE-----
Response verify OK
vericheveo-mstpr251-ocsp-revoked.crt: revoked
	This Update: Sep 29 00:16:47 2025 GMT
	Revocation Time: Sep 28 23:49:08 2025 GMT
```

## Configure and run Nginx server

* Create site dirs:
```
sudo mkdir -p /var/www/valid
sudo mkdir -p /var/www/revoked
```

* Create html pages
```
echo "<h1>OCSP VALID SITE</h1>" | sudo tee /var/www/valid/index.html
echo "<h1>OCSP REVOKED SITE</h1>" | sudo tee /var/www/revoked/index.html
```

* Configure `/etc/hosts`:
```
127.0.0.1   ocsp.valid.vericheveo.ru
127.0.0.1   ocsp.revoked.vericheveo.ru
127.0.0.1   ocsp.vericheveo.ru
```

* Create `/etc/nginx/sites-available/ocsp.conf`:
```
server {
    listen 443 ssl;
    server_name ocsp.valid.vericheveo.ru;

    root /var/www/valid;

    ssl_certificate     /etc/nginx/certs/vericheveo-mstpr251-ocsp-valid.crt;
    ssl_certificate_key /etc/nginx/certs/vericheveo-mstpr251-ocsp-valid.key;
    ssl_trusted_certificate /etc/nginx/certs/vericheveo-mstpr251-chain.crt;
}

server {
    listen 443 ssl;
    server_name ocsp.revoked.vericheveo.ru;

    root /var/www/revoked;

    ssl_certificate     /etc/nginx/certs/vericheveo-mstpr251-ocsp-revoked.crt;
    ssl_certificate_key /etc/nginx/certs/vericheveo-mstpr251-ocsp-revoked.key;
    ssl_trusted_certificate /etc/nginx/certs/vericheveo-mstpr251-chain.crt;
}
```

* Cretae symbol link `/etc/nginx/sites-enabled/ocsp.conf`:
```
sudo mkdir -p /etc/nginx/certs
sudo cp vericheveo-mstpr251-ocsp-*.crt vericheveo-mstpr251-ocsp-*.key vericheveo-mstpr251-chain.crt /etc/nginx/certs/
sudo ln -s /etc/nginx/sites-available/ocsp.conf /etc/nginx/sites-enabled/ocsp.conf
sudo rm /etc/nginx/sites-enabled/default
```

* Check configuration and run server:
```
sudo nginx -t
sudo systemctl reload nginx.service
```

# Run firefox with wireshark and collect trace

* Run firefox:
```
# Export path to log
export SSLKEYLOGFILE=vericheveo-mstpr251-ocsp-valid.log

# Run firefox
pkill firefox
firefox -ProfileManager -new-instance
```

* Add certificate:
  * about:preferences#privacy
  * "View Certificates"
  * "Authorities" → "Import"
  * Select vericheveo-mstpr251-chain.crt
  * Enable ""Trust this CA to identify websites"
  * OK

* Set OCSP settings:
  * about:config
  * security.OCSP.require = true
  * security.OCSP.enabled = 1
  * security.ssl.enable_ocsp_stapling = false
```
