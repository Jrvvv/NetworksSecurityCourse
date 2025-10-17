sudo docker rm -f kali

sudo docker run -d --name kali --network laba-waf_default yutony/kali:v1 tail -f /dev/null

1. Запустить стенд

```
docker compose up

# для остановки ctrl+c
# для сброса docker compose down
```

2. Переходим на http://localhost/setup.php и жмем Create / Reset Database

```
docker exec -it kali bash
```

3. Зайти через бекдор с помощью weevely по адресу http://localhost/hackable/uploads/bd.php, пароль 123

4. Использую mysql добавить пользоваткля. Поля `user`, `last_name`, `first_name` - имя которое используете для сдачи заданий. `password` - md5 хеш этого имени. Логин пароль для базы данных можно найти в файле `/var/www/html/config/config.inc.php`

5. Авторизироваться на http://localhost и перейти в http://localhost/vulnerabilities/sqli/. Вывести список пользователей 

```1' OR '1'='1```

6. Заблокировать эту SQL инекцию и возможность использовать бекдор написав правила для ModSecurity (Coraza WAF) в файле `traefik/dynamic_conf.yml` (изменения применяются автоматически, как только файл обновляется).
