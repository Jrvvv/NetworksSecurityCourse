# WAF

Prereq: docker, tcpdump, wireshark

1. download [link](https://nextcloud.ispras.ru/index.php/s/q87qd6rXzBiemzS/download)

2. unpack zip

3. 
```bash
cd laba-WAF
# in first terminal window
sudo docker compose up
```

4. go to http://localhost/setup.php (http://172.18.0.2/setup.php)

5. if site doesnt open

```bash
# in first terminal window
ctrl-c
sudo docker compose down
sudo docker compose up
```

5. click on "Create / Reset Database"

6. Run
```bash
# in second terminal
cd laba-WAF
sudo docker exec -it kali bash
weevely http://localhost/hackable/uploads/bd.php 123
:sql_console -user app -passwd vulnerables -host localhost -database dvwa -dbms mysql
# 7f778a28b22e0e3a0bdb7089acb1dd19 - is MD5(vericheveo)
echo "INSERT INTO users (user, first_name, last_name, password) VALUES ('vericheveo', 'vericheveo', 'vericheveo', '7f778a28b22e0e3a0bdb7089acb1dd19');" | mysql -u app -pvulnerables dvwa
# make sure entry was added
:sql_console -user app -passwd vulnerables -host localhost -database dvwa -dbms mysql -query "SELECT * FROM users"
```

7. go to http://localhost. Username - vericheveo, password - vericheveo.

8. next go to http://localhost/vulnerabilities/sqli/

9. Run
```bash
# in third terminal window
sudo tcpdump -i any -w vericheveo-sqli-access-all.pcap port 80
```

10. on http://localhost/vulnerabilities/sqli/ in "User ID" form paste
```
1' UNION SELECT first_name, last_name FROM users WHERE '1'='1
```

11. Click "Submit" button (you should see list of all users)

12. Stop tcpdump by ```ctrl-c``` in third terminal.

13. Open this pcap in Wireshark

15. Filter: ```(ip.src==127.0.0.1 || ip.dst==127.0.0.1) && http```

16. Save as "Export Specified Packets" in ```vericheveo-sqli-access.pcap```

17. In second terminal (kali with weevely) exit from weevely by ```ctrl-c```, back in kali shell.

18. Run
```bash
# in third terminal
sudo tcpdump -i any -w vericheveo-ws-access-all.pcap port 80
```

19. Run
```bash
# in second terminal
──(root㉿mylpl)-[/kali-data]
└─# weevely http://localhost/hackable/uploads/bd.php 123

[+] weevely 4.0.1

[+] Target:	www-data@4e6b790152ab:/var/www/html/hackable/uploads
[+] Session:	/root/.weevely/sessions/localhost/bd_0.session
[+] Shell:	System shell

[+] Browse the filesystem or execute commands starts the connection
[+] to the target. Type :help for more information.

weevely> pwd
```

20. Stop tcpdump by ```ctrl-c``` in third terminal.

21. Open this pcap in Wireshark

22. Filter: ```(ip.src==127.0.0.1 || ip.dst==127.0.0.1) && http```

23. Save as "Export Specified Packets" in ```vericheveo-ws-access.pcap```

24. Now it's time to write firewall rules and block SQL injections and web shell access and get pcap's of this.

25. Modify laba-WAF/traefik/dynamic_conf.yml to add just two lines.
```
http:
  routers:
    to-dvwa:
      rule: "Host(`localhost`) && PathPrefix(`/`)"
      middlewares:
      - waf
      service: dvwa

  middlewares:
    waf:
      plugin:
        coraza:
          directives:
            - SecRuleEngine On
            - SecDebugLog /dev/stdout
            - SecDebugLogLevel 9
            - SecRequestBodyAccess On
            - SecResponseBodyAccess On
            - SecResponseBodyMimeType application/json
            # Написать правило здесь. Документация: https://coraza.io/docs/seclang/directives/
            # - SecRule ...
            - SecRule ARGS "@detectSQLi" "id:200,phase:2,deny,status:403"
            - SecRule REQUEST_URI "@rx /hackable/uploads/.*\.php" "id:300,phase:1,deny,status:403"

  services:
    dvwa:
      loadBalancer:
        servers:
        - url: http://dvwa

```

26. Run
```bash
# in third terminal window
sudo tcpdump -i any -w vericheveo-sqli-block-all.pcap port 80
```

27. on http://localhost/vulnerabilities/sqli/ in "User ID" form paste
```
1' UNION SELECT first_name, last_name FROM users WHERE '1'='1
```

28. Click "Submit" button (you should get 403 http error code)

29. Stop tcpdump by ```ctrl-c``` in third terminal.

30. Open this pcap in Wireshark

31. Filter: ```(ip.src==127.0.0.1 || ip.dst==127.0.0.1) && http```

32. Save as "Export Specified Packets" in ```vericheveo-sqli-block.pcap```

33. In second terminal (kali with weevely) exit from weevely by ```ctrl-c```, back in kali shell.

34. Run
```bash
# in third terminal
sudo tcpdump -i any -w vericheveo-ws-block-all.pcap port 80
```

35. Run
```bash
# in second terminal
──(root㉿mylpl)-[/kali-data]
└─# weevely http://localhost/hackable/uploads/bd.php 123

[+] weevely 4.0.1

[+] Target:	www-data@4e6b790152ab:/var/www/html/hackable/uploads
[+] Session:	/root/.weevely/sessions/localhost/bd_0.session
[+] Shell:	System shell

[+] Browse the filesystem or execute commands starts the connection
[+] to the target. Type :help for more information.

weevely> pwd
The request triggers the error 403, please verify running code
Backdoor communication failed, check URL availability and password
```

36. Stop tcpdump by ```ctrl-c``` in third terminal.

37. Open this pcap in Wireshark

38. Filter: ```(ip.src==127.0.0.1 || ip.dst==127.0.0.1) && http````

39. Save as "Export Specified Packets" in ```vericheveo-ws-block.pcap```
