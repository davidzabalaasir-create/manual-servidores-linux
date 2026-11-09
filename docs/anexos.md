# Anexos

## Anexo A. Tabla de puertos

| Puerto | Protocolo | Servicio | Programa | Cifrado | Abierto a Internet |
|---|---|---|---|---|---|
| 21 | TCP | FTP control (FTPS explícito) | vsftpd | TLS obligatorio | Sí (Módulo 15) |
| 22 | TCP | SSH | sshd | Sí | Sí |
| 25 | TCP | SMTP (recepción entre servidores) | Postfix | STARTTLS opcional | Sí |
| 80 | TCP | HTTP | Apache | No (redirige a 443) | Sí |
| 110 | TCP | POP3 | Dovecot | — | No (deshabilitado) |
| 143 | TCP | IMAP | Dovecot | — | No (deshabilitado) |
| 443 | TCP | HTTPS | Apache | TLS | Sí |
| 465 | TCP | SMTPS (envío autenticado) | Postfix | TLS implícito | Sí |
| 587 | TCP | Submission (envío autenticado) | Postfix | STARTTLS obligatorio | Sí |
| 993 | TCP | IMAPS | Dovecot | TLS implícito | Sí |
| 995 | TCP | POP3S | Dovecot | TLS implícito | Sí (opcional) |
| 3306 | TCP | MariaDB | mariadbd | — | **No** (127.0.0.1) |
| 8891 | TCP | Milter OpenDKIM | opendkim | — | **No** (127.0.0.1) |
| 40000–40100 | TCP | FTP pasivo (datos) | vsftpd | TLS obligatorio | Sí (Módulo 15) |

## Anexo B. Mapa de archivos de la infraestructura

| Ruta | Contenido | Módulo |
|---|---|---|
| `/etc/hosts`, `/etc/hostname` | Nombre del servidor | 1 |
| `/etc/ssh/sshd_config.d/00-endurecimiento.conf` | Endurecimiento de SSH | 1 |
| `/etc/apache2/conf-available/servername.conf` | ServerName global | 2 |
| `/etc/apache2/sites-available/000-default.conf` | Sitio comodín HTTP | 3 |
| `/etc/apache2/sites-available/ejemplo1.com.conf` / `ejemplo2.com.conf` | VirtualHosts | 3, 5 |
| `/etc/apache2/conf-available/cabeceras-seguridad.conf` | Cabeceras de seguridad | 12 |
| `/var/www/ejemploN.com/{public_html,config,app,logs,tmp}` | Webs | 3, 5 |
| `/etc/mysql/mariadb.conf.d/50-server.cnf` | Configuración de MariaDB | 4 |
| `/etc/php/8.3/fpm/pool.d/web1.conf` / `web2.conf` | Pools de PHP-FPM | 5 |
| `/etc/letsencrypt/live/*/` | Certificados | 3, 6, 10 |
| `/etc/letsencrypt/renewal-hooks/deploy/recargar-servicios.sh` | Recarga tras renovar | 10 |
| `/etc/postfix/main.cf`, `/etc/postfix/master.cf` | Postfix | 6, 7, 8, 9, 12 |
| `/etc/postfix/virtual_domains`, `vmailbox`, `virtual`, `sender_login` | Tablas de correo | 7, 8 |
| `/etc/aliases` | Alias del sistema | 6, 8 |
| `/etc/dovecot/conf.d/10-auth.conf`, `auth-passwdfile.conf.ext` | Autenticación de Dovecot | 7 |
| `/etc/dovecot/conf.d/10-mail.conf`, `15-mailboxes.conf` | Buzones | 7 |
| `/etc/dovecot/conf.d/10-master.conf` | Servicios y sockets | 7 |
| `/etc/dovecot/conf.d/10-ssl.conf`, `20-lmtp.conf` | TLS y LMTP | 7 |
| `/etc/dovecot/users` | Cuentas de correo | 7, 8 |
| `/var/mail/vhosts/dominio/usuario/Maildir` | Buzones | 7 |
| `/etc/opendkim.conf`, `/etc/opendkim/*` | DKIM | 9 |
| `/etc/systemd/system/opendkim.service.d/override.conf` | Socket de OpenDKIM | 9 |
| `/etc/fail2ban/jail.local` | Fail2ban | 12 |
| `/etc/apt/apt.conf.d/50unattended-upgrades` | Actualizaciones automáticas | 12 |
| `/usr/local/sbin/correo-alta`, `correo-clave`, `correo-baja` | Gestión de cuentas | 8 |
| `/usr/local/sbin/backup-servidor`, `estado-servidor` | Copias y monitorización | 12 |
| `/etc/cron.d/backup-servidor`, `/etc/cron.d/estado-servidor` | Tareas programadas | 12 |
| `/var/backups/servidor/` | Copias de seguridad | 12 |
| `/etc/vsftpd.conf` | Configuración de vsftpd | 15 |
| `/etc/vsftpd.userlist` | Usuarios con acceso FTP | 15 |
| `/etc/pam.d/vsftpd` | Autenticación PAM del FTP | 15 |
| `/var/log/vsftpd.log` | Registro del FTP | 15 |

## Anexo C. Comandos esenciales por servicio

| Servicio | Comprobar configuración | Recargar | Estado / diagnóstico |
|---|---|---|---|
| Apache | `sudo apachectl configtest` | `sudo systemctl reload apache2` | `sudo apachectl -S`, `-M` |
| PHP-FPM | `sudo php-fpm8.3 -t` | `sudo systemctl reload php8.3-fpm` | `ls /run/php/` |
| MariaDB | — | `sudo systemctl restart mariadb` | `sudo mariadb -e "STATUS"` |
| Postfix | `sudo postfix check` | `sudo systemctl reload postfix` | `postconf -n`, `mailq` |
| Dovecot | `sudo doveconf -n` | `sudo doveadm reload` | `sudo doveadm who`, `doveadm auth test` |
| OpenDKIM | `sudo opendkim -n` | `sudo systemctl restart opendkim` | `opendkim-testkey -d dominio -s mail -vvv` |
| SSH | `sudo sshd -t` | `sudo systemctl restart ssh` | `sudo sshd -T` |
| Fail2ban | `sudo fail2ban-client -t` | `sudo fail2ban-client reload` | `sudo fail2ban-client status` |
| UFW | — | `sudo ufw reload` | `sudo ufw status verbose` |
| vsftpd | — (comprobar con `journalctl -u vsftpd` tras reiniciar) | `sudo systemctl restart vsftpd` | `sudo tail /var/log/vsftpd.log` |
| Certbot | — | — | `sudo certbot certificates`, `renew --dry-run` |

## Anexo D. Lista de tareas para añadir un nuevo dominio

1. DNS: A (`@`, `www`, `mail`), MX, SPF, DMARC.
2. Usuario Linux `webN`, carpetas y permisos (PASOS 3.1–3.3).
3. Pool PHP-FPM `webN.conf` (PASO 5.3) y recarga de PHP-FPM.
4. VirtualHost HTTP, certificado con `certbot certonly --webroot`, VirtualHost HTTPS con PHP (PASOS 3.5, 3.11, 5.7).
5. Base de datos, usuario y permisos (PASOS 4.6–4.8).
6. Aplicación y `config.php` (PASOS 5.10–5.16).
7. Correo: añadir el dominio a `/etc/postfix/virtual_domains` + `postmap`; crear cuentas con `correo-alta`; alias `postmaster@`, `abuse@`, `dmarc@`, `noreply@`.
8. Ampliar el certificado del correo con `mail.nuevodominio` (`certbot certonly --webroot -w /var/www/html --cert-name mail.ejemplo1.com -d mail.ejemplo1.com -d mail.ejemplo2.com -d mail.nuevodominio`).
9. DKIM: generar clave, añadirla a `key.table` y `signing.table`, reiniciar OpenDKIM, publicar el TXT.
10. Si usas FTP: PASOS 15.4 y 15.5 para el nuevo usuario, añadirlo a `/etc/vsftpd.userlist` y ampliar el certificado `ftp.ejemplo1.com` con `ftp.nuevodominio`.
11. Verificar con la batería del Módulo 13.
