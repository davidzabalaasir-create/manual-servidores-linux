# Módulo 13. Pruebas y diagnóstico

## Objetivos del módulo

- Disponer de un procedimiento de verificación completo y ordenado de toda la infraestructura.
- Saber qué comando usar para comprobar cada servicio y cómo interpretar su resultado.
- Aplicar un método de diagnóstico sistemático ante cualquier fallo.
- Reconocer los errores reales más frecuentes y su solución.

## 13.1 Método de diagnóstico

Ante cualquier problema, sigue siempre el mismo orden, «de abajo arriba», como las capas de la red:

```text title="Diagrama"
 1. ¿El servicio está funcionando?        systemctl status / is-active
 2. ¿Escucha en el puerto correcto?       ss -tlnp
 3. ¿El cortafuegos lo permite?           ufw status
 4. ¿El DNS resuelve bien?                dig
 5. ¿Se llega por red desde fuera?        nc -vz / curl / openssl s_client (desde otro equipo)
 6. ¿La configuración es válida?          apachectl configtest / postfix check / doveconf -n
 7. ¿Qué dice el log?                     tail / grep / journalctl
 8. ¿Son correctos permisos y propietarios? ls -l / namei -l / sudo -u USUARIO
```

!!! tip "Consejo"
    Cambia **una sola cosa** cada vez y vuelve a probar. Si cambias cinco cosas a la vez y funciona, no sabrás cuál era el problema; si deja de funcionar, no sabrás cuál lo rompió. Y antes de cambiar un archivo, haz una copia: `sudo cp archivo archivo.$(date +%F)`.

## 13.2 Procedimiento de verificación completo

Ejecuta esta batería de pruebas al terminar la instalación, tras cada cambio importante y tras cada actualización mayor. Las pruebas marcadas como «externa» deben hacerse desde **otro** equipo.

### PASO 13.1 – Apache funciona

```bash title="Terminal"
systemctl is-active apache2 && systemctl is-enabled apache2
sudo apachectl configtest
sudo apachectl -S 2>&1 | grep -E 'namevhost|default server'
sudo ss -tlnp | grep -E ':(80|443) '
```

**Resultado esperado:** `active`, `enabled`, `Syntax OK`, los VirtualHosts de ambos dominios y Apache escuchando en 80 y 443.

### PASO 13.2 – Cada dominio carga su propia web

```bash title="Terminal"
for d in www.ejemplo1.com www.ejemplo2.com; do
  echo "== $d"
  curl -s -o /dev/null -w "HTTP %{http_code}  ->  %{redirect_url}\n" http://$d/
  curl -s https://$d/ | grep -o '<h1>[^<]*</h1>'
done
curl -s -o /dev/null -w "%{http_code} %{redirect_url}\n" https://ejemplo2.com/
```

**Resultado esperado:** 

```text title="Salida esperada"
== www.ejemplo1.com
HTTP 301  ->  https://www.ejemplo1.com/
<h1>Agenda de contactos · ejemplo1.com</h1>
== www.ejemplo2.com
HTTP 301  ->  https://www.ejemplo2.com/
<h1>Agenda de contactos · ejemplo2.com</h1>
301 https://www.ejemplo2.com/
```

Prueba externa: abre las cuatro direcciones (`ejemplo1.com`, `www.ejemplo1.com`, `ejemplo2.com`, `www.ejemplo2.com`) en un navegador y en el móvil con datos móviles.

### PASO 13.3 – PHP se ejecuta correctamente y en su pool

```bash title="Terminal"
systemctl is-active php8.3-fpm
sudo php-fpm8.3 -t
ls -l /run/php/*.sock
ps -eo user,cmd | grep '[p]hp-fpm: pool'
```

**Resultado esperado:** `active`, `test is successful`, los dos sockets y, si hay visitas recientes, procesos `php-fpm: pool web1` del usuario `web1` y `php-fpm: pool web2` del usuario `web2` (con `pm = ondemand` puede no haber procesos si nadie ha visitado las webs en los últimos 30 segundos; recarga una web y repite).

### PASO 13.4 – MariaDB responde y cada aplicación accede a su base de datos

```bash title="Terminal"
systemctl is-active mariadb
sudo ss -tlnp | grep 3306
sudo mariadb -e "SELECT table_schema AS bd, COUNT(*) AS tablas FROM information_schema.tables WHERE table_schema LIKE 'web%' GROUP BY table_schema;"
sudo mariadb -e "SHOW GRANTS FOR 'web1_user'@'localhost'; SHOW GRANTS FOR 'web2_user'@'localhost';"
sudo grep -h SQLSTATE /var/www/*/logs/php-error.log 2>/dev/null | tail -5
```

**Resultado esperado:** `active`, escucha solo en `127.0.0.1:3306`, dos tablas en cada base (`contactos` y `mensajes`), permisos limitados y ningún `SQLSTATE` reciente en los logs de PHP. La prueba definitiva es funcional: crear, editar y borrar un contacto en cada web (PASO 5.15).

### PASO 13.5 – Postfix recibe mensajes

```bash title="Terminal"
systemctl is-active postfix@-
sudo postfix check
postconf -n | grep -E '^(myhostname|mydestination|virtual_mailbox_domains|virtual_transport|smtpd_milters)'
postmap -q ejemplo2.com hash:/etc/postfix/virtual_domains
postmap -q soporte@ejemplo2.com hash:/etc/postfix/vmailbox
```

Prueba externa: envía desde Gmail a `info@ejemplo1.com` y a `soporte@ejemplo2.com`, y sigue la llegada:

```bash title="Terminal"
sudo tail -f /var/log/mail.log | grep -E 'postfix/smtpd|lmtp|Saved'
```

Debe aparecer la conexión desde un servidor de Google y `status=sent (250 2.0.0 ... Saved)`.

Prueba de relay abierto desde un equipo externo (debe ser **rechazado**):

```bash title="Terminal"
swaks --server mail.ejemplo1.com --port 25 --from prueba@example.org --to alguien@example.net
```

```text title="Salida esperada"
<** 454 4.7.1 <alguien@example.net>: Relay access denied
```

### PASO 13.6 – Postfix envía mensajes (con autenticación)

```bash title="Terminal"
swaks --server mail.ejemplo1.com --port 587 --tls --auth LOGIN --auth-user administracion@ejemplo1.com --from administracion@ejemplo1.com --to tu-direccion@gmail.com --header "Subject: Verificacion 587"
swaks --server mail.ejemplo2.com --port 465 --tlsc --auth LOGIN --auth-user soporte@ejemplo2.com --from soporte@ejemplo2.com --to tu-direccion@gmail.com --header "Subject: Verificacion 465"
sudo grep -E 'status=(sent|deferred|bounced)' /var/log/mail.log | tail -4
```

**Resultado esperado:** `235 Authentication successful` en swaks y `status=sent (250 2.0.0 OK ...)` hacia Gmail en el log.

### PASO 13.7 – Dovecot permite consultar los buzones

```bash title="Terminal"
systemctl is-active dovecot
sudo doveconf -n > /dev/null && echo "configuración OK"
sudo ss -tlnp | grep dovecot
sudo doveadm user '*'
sudo doveadm auth test info@ejemplo1.com
sudo doveadm mailbox status -u soporte@ejemplo2.com "messages unseen" INBOX
```

Prueba externa: abre las cuentas en Thunderbird o en el móvil con IMAP 993, comprueba que ves los mensajes y que al borrar uno en un dispositivo desaparece en el otro.

### PASO 13.8 – Los certificados son válidos

```bash title="Terminal"
sudo certbot certificates | grep -E 'Certificate Name|Domains|Expiry'
for hp in www.ejemplo1.com:443 www.ejemplo2.com:443; do
  echo | openssl s_client -connect $hp -servername ${hp%:*} -verify_return_error 2>&1 | grep -E '^subject|Verify return code'
done
for p in 465 993 995; do
  echo | openssl s_client -connect mail.ejemplo1.com:$p -verify_return_error 2>&1 | grep 'Verify return code'
done
echo | openssl s_client -connect mail.ejemplo2.com:587 -starttls smtp -verify_return_error 2>&1 | grep 'Verify return code'
sudo certbot renew --dry-run 2>&1 | tail -4
```

**Resultado esperado:** `Verify return code: 0 (ok)` en todos los puertos y `all simulated renewals succeeded`.

### PASO 13.9 – SPF, DKIM y DMARC están correctamente configurados

```bash title="Terminal"
for d in ejemplo1.com ejemplo2.com; do
  echo "== $d"
  echo -n "MX:    "; dig +short MX $d
  echo -n "SPF:   "; dig +short TXT $d | grep spf1
  echo -n "DMARC: "; dig +short TXT _dmarc.$d
  sudo opendkim-testkey -d $d -s mail -vvv 2>&1 | tail -1
done
echo -n "PTR:   "; dig +short -x 203.0.113.10
```

**Resultado esperado:** MX, SPF y DMARC con los valores del Módulo 9, `key OK` para ambos dominios y PTR `mail.ejemplo1.com.`.

Prueba externa: envía un correo desde cada dominio a `check-auth@verifier.port25.com` o a la dirección que te dé `mail-tester.com`. Debes obtener `SPF check: pass`, `DKIM check: pass` y `DMARC check: pass` (y una puntuación de 9 o 10 sobre 10 en mail-tester).

### PASO 13.10 – Los servicios arrancan automáticamente

```bash title="Terminal"
for s in apache2 php8.3-fpm mariadb postfix dovecot opendkim fail2ban ssh certbot.timer; do
  printf '%-14s %s\n' "$s" "$(systemctl is-enabled $s)"
done
```

**Resultado esperado:** `enabled` en todos. La prueba definitiva:

```bash title="Terminal"
sudo reboot
```

Espera un par de minutos, vuelve a conectarte y ejecuta:

```bash title="Terminal"
sudo estado-servidor
```

Todo debe aparecer `[OK]` sin que hayas arrancado nada a mano.

## 13.3 Hoja de verificación final

| # | Comprobación | Comando o prueba | ✓ |
|---|---|---|---|
| 1 | Apache activo y sintaxis correcta | PASO 13.1 | |
| 2 | Cada dominio carga su web y redirige a HTTPS www | PASO 13.2 | |
| 3 | PHP se ejecuta en el pool de cada web | PASO 13.3 | |
| 4 | MariaDB solo local; cada app usa su base | PASO 13.4 | |
| 5 | Postfix recibe de Internet y no es relay abierto | PASO 13.5 | |
| 6 | Envío autenticado por 587 y 465 | PASO 13.6 | |
| 7 | Lectura de buzones por IMAP/POP3 con TLS | PASO 13.7 | |
| 8 | Certificados válidos y renovación simulada correcta | PASO 13.8 | |
| 9 | SPF, DKIM, DMARC y PTR correctos | PASO 13.9 | |
| 10 | Todo arranca tras reiniciar | PASO 13.10 | |
| 11 | Fail2ban con 5 jails activos | `sudo fail2ban-client status` | |
| 12 | Copia de seguridad de hoy completa | `sudo ls -lh /var/backups/servidor/$(date +%F)` | |

## 13.4 Catálogo de errores reales y soluciones

### Web

| Mensaje / síntoma | Dónde aparece | Causa | Solución |
|---|---|---|---|
| `AH00558: Could not reliably determine the server's fully qualified domain name` | `apachectl configtest` | Sin `ServerName` global. | PASO 2.5. |
| `AH00526: Syntax error on line 12 of /etc/apache2/sites-enabled/ejemplo1.com.conf: Invalid command 'RewriteEngine'` | configtest | Módulo no activado. | `sudo a2enmod rewrite`. |
| `AH00072: make_sock: could not bind to address [::]:80` | journal de apache2 | Otro programa usa el puerto. | `sudo ss -tlnp \| grep :80`; detén el otro servicio. |
| `AH01630: client denied by server configuration` | error.log | Falta `Require all granted` en `<Directory>` o la ruta no coincide. | Revisa el bloque `<Directory>`. |
| `AH01797`/`AH00035: access to / denied ... because search permissions are missing` | error.log | Falta permiso `x` en alguna carpeta de la ruta. | `namei -l ruta`; `chmod 750` + grupo `www-data`. |
| `AH01079: failed to make connection to backend: httpd-UDS` + 503 | error.log | Socket de PHP-FPM inexistente. | `systemctl status php8.3-fpm`, revisa `listen` del pool. |
| `Primary script unknown` | error.log | PHP-FPM no encuentra el archivo (ruta o permisos). | El usuario del pool debe poder leer el script. |
| `PHP Warning: require(): open_basedir restriction in effect` | log PHP de la web | El código intenta salir de su carpeta. | Corrige la ruta en el código. |
| `PHP Fatal error: Uncaught PDOException: SQLSTATE[HY000] [1045] Access denied` | log PHP | Credenciales incorrectas. | Revisa `config.php` y PASO 4.11. |
| `PHP Fatal error: Uncaught PDOException: could not find driver` | log PHP | Falta `php-mysql`. | Instala y reinicia FPM. |
| `ERR_TOO_MANY_REDIRECTS` | navegador | Bucle de redirección (proxy/CDN en modo HTTP). | Ver PASO 3.14. |
| `NET::ERR_CERT_COMMON_NAME_INVALID` | navegador | El nombre no está en el certificado o SNI cae en el comodín. | Revisa `ServerName/ServerAlias` y el certificado del VirtualHost. |

### Base de datos

| Mensaje | Causa | Solución |
|---|---|---|
| `ERROR 1698 (28000): Access denied for user 'root'@'localhost'` | Entrar como root sin sudo. | `sudo mariadb`. Es el comportamiento correcto. |
| `ERROR 1044 (42000): Access denied for user 'web1_user'@'localhost' to database 'web2_db'` | Intento de acceder a otra base. | Correcto: es el aislamiento. |
| `ERROR 1142 (42000): DROP command denied` | Permisos limitados. | Correcto, salvo que la aplicación necesite ese permiso. |
| `ERROR 2002 (HY000): Can't connect to local server through socket` | MariaDB detenido. | `systemctl start mariadb`; si no arranca, `/var/log/mysql/error.log` (suele ser disco lleno). |
| `ERROR 1062 (23000): Duplicate entry ... for key 'uk_email'` | Valor repetido en un campo único. | Comportamiento esperado; la aplicación lo traduce a un aviso. |

### Correo

| Mensaje en `/var/log/mail.log` | Causa | Solución |
|---|---|---|
| `NOQUEUE: reject: RCPT from ...: 454 4.7.1 <x@gmail.com>: Relay access denied` | Alguien sin autenticar intenta enviar a fuera. | Correcto si es un desconocido. Si es tu usuario, está usando el puerto 25 en vez del 587. |
| `NOQUEUE: reject: RCPT ...: 550 5.1.1 <x@ejemplo1.com>: Recipient address rejected: User unknown in virtual mailbox table` | La dirección no existe. | Créala o añade un alias. |
| `warning: SASL authentication failure: ...` / `SASL LOGIN authentication failed` | Contraseña incorrecta (o ataque). | Si es masivo, Fail2ban lo bloqueará. |
| `warning: SASL: Connect to private/auth failed: No such file or directory` | Dovecot detenido o sin socket auth. | PASO 7.8. |
| `connect to private/dovecot-lmtp: No such file or directory` + `status=deferred` | Dovecot detenido. | Arranca Dovecot; Postfix reintentará solo. |
| `connect to gmail-smtp-in.l.google.com[...]:25: Connection timed out` | Puerto 25 saliente bloqueado. | PASO 6.1 / relayhost. |
| `said: 550-5.7.26 Unauthenticated email from ejemplo1.com is not accepted due to domain's DMARC policy` | SPF y DKIM fallan. | Módulo 9: revisa firma DKIM y SPF. |
| `said: 550-5.7.25 ... The IP address sending this message does not have a PTR record setup` | Falta PTR (a menudo en IPv6). | PASO 9.2, o `inet_protocols = ipv4`. |
| `warning: hostname X does not resolve to address Y` | El cliente remoto tiene DNS inverso incoherente. | Informativo; es problema del remitente. |
| `warning: connect to Milter service inet:localhost:8891: Connection refused` | OpenDKIM detenido o en otro socket. | PASO 9.8. |
| `dovecot: imap-login: Disconnected (auth failed, 1 attempts in 2 secs): user=<info@ejemplo1.com>` | Contraseña incorrecta en IMAP. | `correo-clave` si el usuario la olvidó. |
| `dovecot: lmtp(...): Error: ... Permission denied` | Propietario de los buzones incorrecto. | `sudo chown -R vmail:vmail /var/mail/vhosts`. |
| `dovecot: auth: Error: passwd-file /etc/dovecot/users: User ... has invalid UID` | Campo uid vacío o erróneo. | Revisa el formato de la línea (8 campos). |

## 13.5 Caja de herramientas de diagnóstico

| Herramienta | Uso | Ejemplo |
|---|---|---|
| `systemctl` / `journalctl` | Estado y logs de servicios. | `journalctl -u dovecot -n 50` |
| `ss` | Puertos en escucha y conexiones. | `sudo ss -tlnp` |
| `dig` | Consultas DNS. | `dig +short MX ejemplo1.com @1.1.1.1` |
| `curl` | Pruebas HTTP/HTTPS. | `curl -vI https://www.ejemplo1.com` |
| `openssl s_client` | Pruebas TLS en cualquier puerto. | `openssl s_client -connect mail.ejemplo1.com:993` |
| `nc` | Conectividad TCP y diálogos manuales. | `nc -vz mail.ejemplo1.com 25` |
| `swaks` | Pruebas SMTP completas. | Ver PASO 7.19 |
| `doveadm` | Usuarios, autenticación y buzones de Dovecot. | `doveadm auth test usuario` |
| `postconf` / `postmap -q` / `mailq` | Configuración, tablas y cola de Postfix. | `postmap -q x@ejemplo1.com hash:/etc/postfix/vmailbox` |
| `namei -l` | Permisos de toda una ruta. | `namei -l /var/www/ejemplo1.com/public_html/index.php` |
| `sudo -u usuario` | Probar como otro usuario. | `sudo -u web1 cat /var/www/ejemplo1.com/config/config.php` |
| `fail2ban-client` | Estado de bloqueos. | `fail2ban-client status sshd` |
| `htop`, `free -h`, `df -h` | Recursos (CPU, memoria, disco). | `df -h` |

## 13.6 Ejercicios prácticos del Módulo 13

1. Ejecuta toda la batería de pruebas del apartado 13.2 y rellena la hoja de verificación.
2. Pide a un compañero que provoque un fallo oculto (detener un servicio, cambiar un permiso, borrar una línea de configuración) y encuéntralo siguiendo el método del apartado 13.1.
3. Provoca deliberadamente tres errores del catálogo y verifica que aparecen exactamente los mensajes descritos.
4. Escribe tu propio script `verificar-infraestructura` que automatice los pasos 13.1 a 13.10 y devuelva un resumen.
5. Documenta en una tabla tus propios errores encontrados durante el curso, su causa y su solución: es la base del manual de operaciones de tu servidor.
