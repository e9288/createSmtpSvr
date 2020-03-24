# createSmtpSvr
Ubuntu 14.04
postfix, dovecont, squirrelmail 을 이용한 smtp 서버 구현
참고 자료 : https://www.krizna.com/ubuntu/setup-mail-server-ubuntu-14-04/

1. postfix 설치
  sudo apt-get install postfix

  General type of main configuration : Internet Site
  System mail name : my_domain 
  ex) example.com


2. /etc/postfix/main.cf 파일에 아래 행을 추가함으로써 Dovecot SASL을 사용한 postfix SMTP-AUTH를 설정 할 수 있다.
  home_mailbox = Maildir/
  smtpd_sasl_type = dovecot
  smtpd_sasl_path = private/auth
  smtpd_sasl_local_domain =
  smtpd_sasl_security_options = noanonymous
  broken_sasl_auth_clients = yes
  smtpd_sasl_auth_enable = yes
  smtpd_recipient_restrictions = permit_sasl_authenticated,permit_mynetworks,reject_unauth_destination
  smtp_tls_security_level = may
  smtpd_tls_security_level = may
  smtp_tls_note_starttls_offer = yes
  smtpd_tls_loglevel = 1
  smtpd_tls_received_header = yes

3. tls를 위한 디지털 인증서 생성. 한 줄씩 실행하면서 도메인에 맞는 설정 정보를 넣어준다.
  $ openssl genrsa -des3 -out server.key 2048
  $ openssl rsa -in server.key -out server.key.insecure
  $ mv server.key server.key.secure
  $ mv server.key.insecure server.key
  $ openssl req -new -key server.key -out server.csr
    ---------------------------------------------------
    Country Name (2 letter code) [AU]:KO
    State or Province Name (full name) [Some-State]:Seoul     
    Locality Name (eg, city) []:gangnam
    Organization Name (eg, company) [Internet Widgits Pty Ltd]:Company Name
    Organizational Unit Name (eg, section) []:Department Name
    Common Name (e.g. server FQDN or YOUR name) []:example.com
    Email Address []:Mail Address

    Please enter the following 'extra' attributes
    to be sent with your certificate request
    A challenge password []:
    An optional company name []:
    두 란은 비워입력한다.
    ---------------------------------------------------
  $ openssl x509 -req -days 365 -in server.csr -signkey server.key -out server.crt
  $ sudo cp server.crt /etc/ssl/certs
  $ sudo cp server.key /etc/ssl/private

4. 인증 경로 설정
  $ sudo postconf -e 'smtpd_tls_key_file = /etc/ssl/private/server.key'
  $ sudo postconf -e 'smtpd_tls_cert_file = /etc/ssl/certs/server.crt'
  
5. smtp 465, 587 허용을 위해 /etc/postfix/master.cf 파일에서 아래 내용을 주석 처리한다.
    -o syslog_name=postfix/submission
    -o smtpd_tls_security_level=encrypt
    -o smtpd_sasl_auth_enable=yes
    -o smtpd_relay_restrictions=permit_sasl_authenticated,reject
    -o milter_macro_daemon_name=ORIGINATING
  smtps     inet  n       -       n       -       -       smtpd
    -o syslog_name=postfix/smtps
    -o smtpd_tls_wrappermode=yes
    -o smtpd_sasl_auth_enable=yes
    -o smtpd_relay_restrictions=permit_sasl_authenticated,reject
    -o milter_macro_daemon_name=ORIGINATING

6. dovecot SASL 설치
  $ sudo apt-get install dovecot-common
    -------------------------------------
    Yes
    Host name : example.com
    -------------------------------------
7. dovecot conf file 변경
  /etc/dovecot/conf.d/10-master.conf 파일에서 '# Postfix smtp-auth' 문구 아래에 아래 내용을 추가
    unix_listener /var/spool/postfix/private/auth {
    mode = 0660
    user = postfix
    group = postfix
    }
  /etc/dovecot/conf.d/10-auth.conf 파일에서 'auth_mechanisms = plain' 를 아래 내용으로 대체
    auth_mechanisms = plain login

8. Restart postfix and dovecot services
  $ sudo service postfix restart
  $ sudo service dovecot restart

9. 아래 명령어를 이용하여 SMTP-AUTH and smtp/pop3 포트 접근에 대한 테스트 수행
  $ telnet example.com smtp
  $ telnet example.com 587

------------------ postfix (mail Sender) 설정끝 --------------------

10. dovecot 설치
  $ sudo apt-get install dovecot-imapd dovecot-pop3d
  
11. /etc/dovecot/conf.d/10-mail.conf 파일의 'mail_location = mbox:~/mail:INBOX=/var/mail/%u' 행을 아래 행으로 대체
  mail_location = maildir:~/Maildir

12. /etc/dovecot/conf.d/20-pop3.conf 파일의 'pop3_uidl_format = %08Xu%08Xv' 주석을 해제

13. /etc/dovecot/conf.d/10-ssl.conf 파일의 'ssl = yes' 주석을 해제

14. dovecot restart
  sudo service dovecot restart

15. pop3, imap 포트 테스트
  $ telnet example.com 110
  $ telnet example.com 995
  $ telnet example.com 993
  $ telnet example.com 143

