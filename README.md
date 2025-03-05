

## Инструкция по развёртыванию почтового сервиса на VPS с Debian



Эта инструкция поможет вам настроить лёгкий и безопасный почтовый сервер на VPS с Debian, используя ваш собственный домен. Мы обеспечим максимальную безопасность и установим веб-интерфейс с поддержкой двухфакторной аутентификации через Passkey (на основе WebAuthn).



---



### Требования

- VPS с Debian 11 или 12.

- Домен (например, `example.com`), который вы контролируете.

- Доступ к настройкам DNS вашего домена.

- SSH-доступ к VPS.



---



### Шаг 1: Подготовка системы



1. **Обновите систему**  

   Подключитесь к VPS через SSH и выполните:  

   ```

   sudo apt update && sudo apt upgrade -y

   ```



2. **Установите необходимые пакеты**  

   Установим Postfix (MTA), Dovecot (IMAP-сервер), Nginx (веб-сервер), PHP (для веб-интерфейса) и Certbot (для SSL):  

   ```

   sudo apt install postfix dovecot-core dovecot-imapd nginx php-fpm php-common php-pdo php-sqlite3 php-mbstring php-openssl php-json php-xml php-dom php-zip php-gd php-imap certbot python3-certbot-nginx -y

   ```  

   - Во время установки Postfix выберите **"Internet Site"** и укажите ваш домен (например, `example.com`).



---



### Шаг 2: Настройка DNS



Для работы почтового сервера нужно настроить DNS вашего домена:



1. **Добавьте MX-запись**  

   - Укажите, что ваш VPS обрабатывает почту для домена:  

     ```

     example.com. IN MX 10 mail.example.com.

     ```  

   - Создайте A-запись для `mail.example.com`, указывающую на IP-адрес вашего VPS:  

     ```

     mail.example.com. IN A <IP_VPS>

     ```



2. **Проверка**  

   Убедитесь, что DNS-записи применились, используя команду:  

   ```

   nslookup -type=mx example.com

   ```



---



### Шаг 3: Настройка Postfix



Postfix будет отправлять и принимать почту.



1. **Отредактируйте конфигурацию**  

   Откройте файл `/etc/postfix/main.cf` и настройте:  

   ```

   myhostname = mail.example.com

   mydomain = example.com

   myorigin = $mydomain

   inet_interfaces = all

   mydestination = $myhostname, localhost.$mydomain, localhost, $mydomain

   mynetworks = 127.0.0.0/8 [::ffff:127.0.0.0]/104 [::1]/128

   home_mailbox = Maildir/

   smtpd_sasl_type = dovecot

   smtpd_sasl_path = private/auth

   smtpd_sasl_auth_enable = yes

   smtpd_recipient_restrictions = permit_sasl_authenticated, permit_mynetworks, reject_unauth_destination

   ```



2. **Перезапустите Postfix**  

   ```

   sudo systemctl restart postfix

   ```



---



### Шаг 4: Настройка Dovecot



Dovecot обеспечит доступ к почте через IMAP.



1. **Настройте расположение почты**  

   Отредактируйте `/etc/dovecot/conf.d/10-mail.conf`:  

   ```

   mail_location = maildir:~/Maildir

   ```



2. **Включите механизмы аутентификации**  

   В `/etc/dovecot/conf.d/10-auth.conf` укажите:  

   ```

   auth_mechanisms = plain login

   ```



3. **Настройте интеграцию с Postfix**  

   В `/etc/dovecot/conf.d/10-master.conf` добавьте:  

   ```

   service auth {

     unix_listener /var/spool/postfix/private/auth {

       mode = 0660

       user = postfix

       group = postfix

     }

   }

   ```



4. **Перезапустите Dovecot**  

   ```

   sudo systemctl restart dovecot

   ```



---



### Шаг 5: Создание почтовых пользователей



Для каждого почтового ящика создайте системного пользователя:



1. **Создайте пользователя**  

   ```

   sudo adduser user1

   ```  

   - После этого почтовый адрес будет `user1@example.com`.



2. **Повторите для других пользователей при необходимости**.



---



### Шаг 6: Установка Roundcube (веб-интерфейс)



Roundcube — лёгкий веб-клиент для работы с почтой.



1. **Скачайте и установите Roundcube**  

   ```

   wget https://github.com/roundcube/roundcubemail/releases/download/1.5.0/roundcubemail-1.5.0-complete.tar.gz

   tar -xvf roundcubemail-1.5.0-complete.tar.gz

   sudo mv roundcubemail-1.5.0 /var/www/html/roundcube

   sudo chown -R www-data:www-data /var/www/html/roundcube

   ```



2. **Настройте Roundcube**  

   - Откройте в браузере `http://<IP_VPS>/roundcube/installer`.  

   - Выберите **SQLite** как базу данных (лёгкая альтернатива MySQL).  

   - Укажите настройки IMAP и SMTP:  

     - IMAP: `localhost`, порт 143, без SSL (локальное соединение).  

     - SMTP: `localhost`, порт 587, с TLS.  

   - Завершите настройку и удалите папку `installer`:  

     ```

     sudo rm -rf /var/www/html/roundcube/installer

     ```



---



### Шаг 7: Добавление двухфакторной аутентификации через Passkey



Мы используем плагин `twofactor_webauthn` для Roundcube, который поддерживает WebAuthn (Passkey).



1. **Установите плагин**  

   ```

   cd /var/www/html/roundcube/plugins

   git clone https://github.com/alexandregz/twofactor_webauthn.git

   sudo chown -R www-data:www-data twofactor_webauthn

   ```



2. **Активируйте плагин**  

   Отредактируйте `/var/www/html/roundcube/config/config.inc.php`:  

   ```

   $config['plugins'] = array('twofactor_webauthn');

   ```



3. **Проверка**  

   Войдите в Roundcube через браузер, перейдите в настройки и включите двухфакторную аутентификацию с помощью Passkey (например, USB-ключа или биометрии).



---



### Шаг 8: Настройка Nginx и HTTPS



Nginx будет обслуживать Roundcube с шифрованием HTTPS.



1. **Создайте конфигурацию Nginx**  

   Создайте файл `/etc/nginx/sites-available/roundcube.conf`:  

   ```

   server {

       listen 80;

       server_name example.com;

       return 301 https://$host$request_uri;

   }



   server {

       listen 443 ssl;

       server_name example.com;



       ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;

       ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;



       root /var/www/html/roundcube;

       index index.php;



       location / {

           try_files $uri $uri/ /index.php;

       }



       location ~ \.php$ {

           include snippets/fastcgi-php.conf;

           fastcgi_pass unix:/var/run/php/php7.4-fpm.sock;  # Проверьте версию PHP (например, php8.1-fpm)

       }

   }

   ```



2. **Активируйте сайт**  

   ```

   sudo ln -s /etc/nginx/sites-available/roundcube.conf /etc/nginx/sites-enabled/

   sudo nginx -t

   sudo systemctl reload nginx

   ```



3. **Получите SSL-сертификат**  

   Используйте Certbot для автоматической настройки HTTPS:  

   ```

   sudo certbot --nginx -d example.com

   ```



---



### Шаг 9: Настройка SSL/TLS для Postfix и Dovecot



Для повышения безопасности включим шифрование.



1. **Postfix**  

   Добавьте в `/etc/postfix/main.cf`:  

   ```

   smtpd_tls_cert_file = /etc/letsencrypt/live/example.com/fullchain.pem

   smtpd_tls_key_file = /etc/letsencrypt/live/example.com/privkey.pem

   smtpd_use_tls = yes

   smtpd_tls_auth_only = yes

   smtp_tls_security_level = may

   ```



2. **Dovecot**  

   Отредактируйте `/etc/dovecot/conf.d/10-ssl.conf`:  

   ```

   ssl = yes

   ssl_cert = </etc/letsencrypt/live/example.com/fullchain.pem

   ssl_key = </etc/letsencrypt/live/example.com/privkey.pem

   ```



3. **Перезапустите службы**  

   ```

   sudo systemctl restart postfix dovecot

   ```



---



### Шаг 10: Настройка SPF, DKIM и DMARC



Эти механизмы повысят доставляемость и безопасность почты.



1. **Установите OpenDKIM**  

   ```

   sudo apt install opendkim opendkim-tools -y

   ```



2. **Сгенерируйте DKIM-ключи**  

   ```

   sudo opendkim-genkey -d example.com -s mail -b 2048

   sudo mv mail.private mail.txt /etc/opendkim/keys/

   sudo chown opendkim:opendkim /etc/opendkim/keys/mail.private

   ```  

   - Добавьте содержимое `mail.txt` в DNS как TXT-запись для `mail._domainkey.example.com`.



3. **Настройте OpenDKIM**  

   Отредактируйте `/etc/opendkim.conf`:  

   ```

   Domain example.com

   KeyFile /etc/opendkim/keys/mail.private

   Selector mail

   ```



4. **Интегрируйте с Postfix**  

   В `/etc/postfix/main.cf` добавьте:  

   ```

   milter_default_action = accept

   milter_protocol = 2

   smtpd_milters = inet:localhost:8891

   non_smtpd_milters = inet:localhost:8891

   ```



5. **Перезапустите службы**  

   ```

   sudo systemctl restart opendkim postfix

   ```



6. **Добавьте SPF и DMARC в DNS**  

   - SPF: `v=spf1 ip4:<IP_VPS> ~all`  

   - DMARC: `v=DMARC1; p=none; rua=mailto:dmarc-reports@example.com`



---



### Шаг 11: Дополнительные меры безопасности



1. **Настройте брандмауэр**  

   ```

   sudo ufw allow 22,25,80,143,443,587/tcp

   sudo ufw enable

   ```



2. **Отключите root-доступ по SSH**  

   В `/etc/ssh/sshd_config` установите `PermitRootLogin no` и перезапустите SSH:  

   ```

   sudo systemctl restart ssh

   ```



3. **Регулярные обновления**  

   ```

   sudo apt update && sudo apt upgrade -y

   ```



---



### Шаг 12: Тестирование



1. **Проверьте отправку и получение писем**  

   Отправьте тестовое письмо на `user1@example.com` и проверьте его через IMAP.



2. **Проверьте веб-интерфейс**  

   Откройте `https://example.com` и войдите как `user1`.



3. **Настройте Passkey**  

   В настройках Roundcube включите двухфакторную аутентификацию.



---



Теперь у вас есть безопасный почтовый сервер на Debian с веб-интерфейсом и поддержкой Passkey!

