# Módulo 14. Proyecto final

## Objetivos del proyecto

Construir **desde cero**, en un servidor nuevo y sin consultar soluciones salvo este manual, una infraestructura completa con dos dominios independientes. Al terminar, deberás poder:

- Acceder a `https://www.ejemplo1.com` y `https://www.ejemplo2.com`, cada una con su aplicación PHP y su base de datos.
- Usar desde un cliente de correo las cuentas de ambos dominios, enviando y recibiendo correo de Internet con SPF, DKIM y DMARC correctos.
- Demostrar que la infraestructura es segura, que se recupera sola tras un reinicio y que tiene copias de seguridad.

!!! tip "Consejo"
    Para el proyecto, sustituye `ejemplo1.com` y `ejemplo2.com` por dominios reales tuyos (o por los nombres de tu laboratorio) y la IP `203.0.113.10` por la de tu servidor. Una forma cómoda de hacerlo es preparar una tabla con tus valores (apartado «Los datos del proyecto guía» de la introducción) antes de empezar.

## 14.1 Requisitos de partida

| Requisito | Comprobado |
|---|---|
| Servidor con Ubuntu Server 24.04 LTS recién instalado, 2 GB de RAM o más. | |
| IP pública fija; puertos 25, 80, 443, 465, 587, 993 y 995 sin bloquear por el proveedor. | |
| Posibilidad de configurar el PTR de la IP. | |
| Dos dominios con acceso a su panel DNS. | |
| Acceso SSH o consola web del proveedor. | |
| Un ordenador con cliente SSH y un cliente de correo (Thunderbird). | |

## 14.2 Plan de construcción

Sigue las fases en orden. Cada fase termina con una **verificación obligatoria**: no pases a la siguiente sin superarla.

### Fase 1 – Sistema base (Módulo 1)

| Tarea | Referencia |
|---|---|
| Primera conexión y actualización completa | PASOS 1.1 y 1.2 |
| Hostname `mail.ejemplo1.com` y `/etc/hosts` | PASO 1.3 |
| Zona horaria y NTP | PASO 1.4 |
| Usuario `sysadmin` con sudo | PASO 1.5 |
| Claves SSH y endurecimiento de SSH | PASOS 1.6 y 1.7 |
| Herramientas básicas y UFW con SSH | PASOS 1.8 y 1.9 |

Verificación: `hostname -f` devuelve `mail.ejemplo1.com`; entras como `sysadmin` con clave; `ssh root@IP` es rechazado; `ufw status` activo.

### Fase 2 – DNS inicial (Módulos 3 y 9)

| Tarea | Referencia |
|---|---|
| Registros A (y AAAA si procede) para `@`, `www` y `mail` de ambos dominios | PASO 3.10 |
| PTR de la IP → `mail.ejemplo1.com` | PASO 9.2 |
| Registros MX de ambos dominios | PASO 9.1 |
| SPF de dominio y de `mail` | PASO 9.3 |
| DMARC en modo `p=none` | PASO 9.11 |

Hazlo pronto: así, mientras trabajas en las fases siguientes, el DNS tiene tiempo de propagarse. Verificación: `dig` devuelve los valores correctos desde `1.1.1.1` y `8.8.8.8`; `dig -x IP` devuelve `mail.ejemplo1.com.`

### Fase 3 – Servidor web y VirtualHosts (Módulos 2 y 3)

| Tarea | Referencia |
|---|---|
| Instalar Apache, abrir 80/443, ServerName global, módulos | PASOS 2.1 a 2.6 |
| Usuarios `web1`/`web2`, carpetas y permisos | PASOS 3.1 a 3.3 |
| Sitio comodín `000-default` | PASO 3.4 |
| VirtualHosts HTTP y habilitación | PASOS 3.5 a 3.7 |
| Certificados de las webs y del correo | PASOS 3.11 y 6.2 |
| VirtualHosts HTTPS y comodín `default-ssl` | PASOS 3.12 a 3.14 |

Verificación: PASOS 13.1 y 13.2 (con las páginas HTML de prueba).

### Fase 4 – Base de datos (Módulo 4)

| Tarea | Referencia |
|---|---|
| Instalar MariaDB, comprobar `bind-address`, `mariadb-secure-installation` | PASOS 4.1 a 4.3 |
| Contraseñas, bases `web1_db`/`web2_db`, usuarios y permisos | PASOS 4.5 a 4.8 |
| Tablas `contactos` (y `mensajes`, PASO 11.1) | PASO 4.9 |
| Prueba de aislamiento | PASO 4.10 |
| Copia y restauración manual | PASOS 4.12 y 4.13 |

Verificación: `web1_user` lee `web1_db` y recibe `Access denied` en `web2_db`.

### Fase 5 – PHP y aplicaciones (Módulos 5 y 11)

| Tarea | Referencia |
|---|---|
| Instalar PHP-FPM y extensiones | PASO 5.1 |
| Carpetas de logs, pools `web1` y `web2`, desactivar `www` | PASOS 5.2 a 5.5 |
| `proxy_fcgi` y VirtualHosts definitivos | PASOS 5.6 a 5.8 |
| Comprobar usuario de ejecución | PASO 5.9 |
| `config.php`, `db.php`, aplicación CRUD y estilos | PASOS 5.10 a 5.14 |
| Despliegue en ejemplo2.com | PASO 5.16 |

Verificación: PASOS 13.3, 13.4 y 5.15 (las cuatro operaciones CRUD en ambas webs).

### Fase 6 – Correo (Módulos 6, 7 y 8)

| Tarea | Referencia |
|---|---|
| Comprobar el puerto 25 saliente | PASO 6.1 |
| Instalar Postfix, `main.cf`, alias y puerto 25 | PASOS 6.3 a 6.6 |
| Pruebas de envío y recepción locales | PASOS 6.7 a 6.9 |
| Instalar Dovecot, usuario `vmail` y primera cuenta | PASOS 7.1 a 7.3 |
| Autenticación, buzones, servicios, TLS y LMTP | PASOS 7.4 a 7.11 |
| Integración con Postfix y puertos 587/465 | PASOS 7.12 a 7.15 |
| Tabla de remitentes y restricción en 587/465 | PASOS 8.1 y 8.2 |
| Scripts `correo-alta`, `correo-clave` y `correo-baja` | PASOS 8.3 a 8.5 |
| Cuentas, alias y redirección de root | PASOS 8.6 a 8.9 |

Verificación: PASOS 13.5, 13.6 y 13.7, y configuración de Thunderbird (PASO 8.11) para `info@ejemplo1.com` y `soporte@ejemplo2.com`.

### Fase 7 – Autenticación del correo (Módulo 9)

| Tarea | Referencia |
|---|---|
| Instalar OpenDKIM y generar claves | PASOS 9.4 y 9.5 |
| Tablas, `opendkim.conf` y socket | PASOS 9.6 a 9.8 |
| Milter en Postfix | PASO 9.9 |
| Publicar DKIM en el DNS | PASO 9.10 |

Verificación: PASO 13.9 y «Mostrar original» en Gmail con SPF, DKIM y DMARC en **PASS** para ambos dominios (PASO 9.12).

### Fase 8 – TLS, integración y seguridad (Módulos 10, 11 y 12)

| Tarea | Referencia |
|---|---|
| Temporizador, simulación y *deploy hook* de Certbot | PASOS 10.1 a 10.3 |
| Formulario de contacto en ambas webs | PASOS 11.1 y 11.2 |
| UFW definitivo, cabeceras de Apache, límites de Postfix | PASOS 12.1 a 12.4 |
| Fail2ban | PASO 12.5 |
| Actualizaciones automáticas | PASO 12.6 |
| Copias de seguridad y cron | PASOS 12.7 a 12.9 |
| Script de estado y monitorización | PASO 12.10 |

Verificación: PASOS 13.8, 13.10 y la hoja de verificación completa del apartado 13.3.

### Fase 9 (opcional) – Servidor FTP cifrado (Módulo 15)

| Tarea | Referencia |
|---|---|
| Registros DNS `ftp` y certificado `ftp.ejemplo1.com` | PASOS 15.1 y 15.2 |
| Instalar vsftpd, contraseñas y jaula chroot con ACL | PASOS 15.3 a 15.5 |
| PAM y lista de usuarios autorizados | PASOS 15.6 y 15.7 |
| `vsftpd.conf` y puertos en UFW | PASOS 15.8 y 15.9 |
| Renovación, Fail2ban y monitorización | PASOS 15.12 a 15.14 |

Verificación: PASO 15.10 (cifrado obligatorio, usuario no autorizado rechazado, jaula y subida con permisos correctos) y conexión desde FileZilla (PASO 15.11).

## 14.3 Pruebas de aceptación

El proyecto se considera superado cuando se cumplen **todas** estas pruebas, realizadas desde fuera del servidor salvo indicación:

| # | Prueba | Resultado esperado |
|---|---|---|
| 1 | Abrir `http://ejemplo1.com` | Redirige a `https://www.ejemplo1.com` con candado válido y muestra la agenda de ejemplo1. |
| 2 | Abrir `http://ejemplo2.com` | Redirige a `https://www.ejemplo2.com` y muestra la agenda de ejemplo2. |
| 3 | Crear un contacto en la web 1 | Aparece en la web 1 y **no** en la web 2. |
| 4 | Editar y borrar ese contacto | Los cambios se reflejan en `web1_db`. |
| 5 | Enviar el formulario de contacto de la web 2 | El mensaje se guarda en `web2_db.mensajes` y llega al buzón de `soporte@ejemplo2.com`. |
| 6 | Desde Gmail, escribir a `info@ejemplo1.com` y `soporte@ejemplo2.com` | Ambos llegan a sus buzones en Thunderbird. |
| 7 | Desde Thunderbird, responder a Gmail con cada cuenta | Llegan a la bandeja de entrada de Gmail con SPF, DKIM y DMARC en PASS. |
| 8 | Escribir a `postmaster@ejemplo2.com` | Llega al buzón de `soporte@ejemplo2.com`. |
| 9 | Escribir a `noexiste@ejemplo1.com` | Gmail informa de que la dirección no existe (rechazo 550). |
| 10 | Intentar relay sin autenticación por el puerto 25 | `Relay access denied`. |
| 11 | Autenticarse como `info@` y enviar como `administracion@` | `Sender address rejected: not owned by user`. |
| 12 | `nc -vz IP 3306`, `nc -vz IP 143`, `nc -vz IP 110` | Las tres conexiones fallan. |
| 13 | 4 intentos de contraseña erróneos en IMAP desde una IP de prueba | La IP queda bloqueada por Fail2ban. |
| 14 | Reiniciar el servidor | Todo vuelve a funcionar sin intervención; `estado-servidor` todo en `[OK]`. |
| 15 | Puntuación en mail-tester.com | 9/10 o superior desde ambos dominios. |
| 16 | Borrar `web2_db` y restaurarla desde la copia automática de la noche | La web 2 vuelve a mostrar sus datos. |
| 17 (Fase 9) | Subir por FTPS con FileZilla un archivo a `public_html` de ejemplo2.com como `web2` | Se ve en `https://www.ejemplo2.com`; `web2` no puede ver la carpeta de ejemplo1.com y el FTP sin cifrar es rechazado. |

## 14.4 Entregables del proyecto

1. **Ficha de la infraestructura**: la tabla del apartado 11.1 con tus valores reales.
2. **Registro de DNS**: exportación o captura de los registros de ambos dominios.
3. **Configuraciones**: copia (sin contraseñas ni claves privadas) de los archivos que has creado o modificado. Puedes obtener las configuraciones efectivas con `postconf -n`, `sudo doveconf -n`, `sudo apachectl -S` y `sudo opendkim -n` (comprobación de la configuración de OpenDKIM).
4. **Evidencias**: salida de la batería de pruebas del Módulo 13, cabeceras `Authentication-Results` de Gmail y resultado de mail-tester.
5. **Manual de operaciones**: cómo crear una cuenta de correo, cómo añadir una tercera web y cómo restaurar una copia, escritos por ti con tus palabras.
6. **Registro de incidencias**: los errores que encontraste, su causa y su solución.

## 14.5 Retos de ampliación

Cuando domines el proyecto, amplíalo:

1. **Tercer dominio completo** (`ejemplo3.com`): usuario `web3`, carpeta, pool, VirtualHost, certificado, base de datos, dominio de correo en `virtual_domains`, clave DKIM y registros DNS. Escribe un script `sitio-alta` que automatice la parte web.
2. **Webmail**: instala Roundcube en un VirtualHost propio (`webmail.ejemplo1.com`) con su propia base de datos, conectado por IMAP a `127.0.0.1:993` y por SMTP a `127.0.0.1:587`.
3. **Usuarios de correo en MariaDB**: sustituye `/etc/dovecot/users` y las tablas `hash:` de Postfix por consultas a una base de datos `mailserver` (drivers `sql` en Dovecot y `mysql:` en Postfix, paquete `postfix-mysql`).
4. **Antispam**: añade Rspamd como milter para filtrar el correo entrante y mover el spam a la carpeta Junk con Sieve (`dovecot-sieve`).
5. **Cuotas de buzón** con el plugin `quota` de Dovecot y aviso al usuario al 90 %.
6. **MTA-STS y TLS-RPT**: publica la política que obliga a los servidores remitentes a usar TLS válido al entregarte correo.
7. **Copias externas cifradas** con `restic` o `borg` hacia otro servidor.
8. **Migración a Dovecot 2.4**: en una máquina de pruebas con Debian 13, traduce la configuración de los módulos 7 y 8 a la nueva sintaxis.

## 14.6 Archivo main.cf completo de referencia

Así queda `/etc/postfix/main.cf` al terminar todos los módulos. Úsalo para comparar con el tuyo:

```text title="Archivo de configuración"
# =====================================================================
#  /etc/postfix/main.cf  -  mail.ejemplo1.com  (versión final)
# =====================================================================
compatibility_level = 3.6
smtpd_banner = $myhostname ESMTP
biff = no
append_dot_mydomain = no
readme_directory = no

# --- Identidad del servidor ---
myhostname = mail.ejemplo1.com
mydomain = ejemplo1.com
myorigin = $mydomain
mydestination = $myhostname, localhost.$mydomain, localhost
mynetworks = 127.0.0.0/8 [::ffff:127.0.0.0]/104 [::1]/128
inet_interfaces = all
inet_protocols = all
relayhost =

# --- Alias y límites ---
alias_maps = hash:/etc/aliases
alias_database = hash:/etc/aliases
recipient_delimiter = +
mailbox_size_limit = 0
message_size_limit = 52428800

# --- TLS entrante ---
smtpd_tls_cert_file = /etc/letsencrypt/live/mail.ejemplo1.com/fullchain.pem
smtpd_tls_key_file = /etc/letsencrypt/live/mail.ejemplo1.com/privkey.pem
smtpd_tls_security_level = may
smtpd_tls_auth_only = yes
smtpd_tls_protocols = >=TLSv1.2
smtpd_tls_mandatory_protocols = >=TLSv1.2
smtpd_tls_loglevel = 1
smtpd_tls_session_cache_database = btree:${data_directory}/smtpd_scache

# --- TLS saliente ---
smtp_tls_security_level = may
smtp_tls_CApath = /etc/ssl/certs
smtp_tls_loglevel = 1
smtp_tls_session_cache_database = btree:${data_directory}/smtp_scache

# --- Restricciones ---
smtpd_helo_required = yes
disable_vrfy_command = yes
smtpd_helo_restrictions =
    permit_mynetworks,
    permit_sasl_authenticated,
    reject_invalid_helo_hostname,
    reject_non_fqdn_helo_hostname
smtpd_sender_restrictions =
    permit_mynetworks,
    permit_sasl_authenticated,
    reject_non_fqdn_sender,
    reject_unknown_sender_domain
smtpd_relay_restrictions =
    permit_mynetworks,
    permit_sasl_authenticated,
    defer_unauth_destination
smtpd_recipient_restrictions =
    permit_mynetworks,
    permit_sasl_authenticated,
    reject_unauth_destination,
    reject_non_fqdn_recipient,
    reject_unknown_recipient_domain

# --- Buzones virtuales (Dovecot) ---
virtual_mailbox_domains = hash:/etc/postfix/virtual_domains
virtual_mailbox_maps = hash:/etc/postfix/vmailbox
virtual_alias_maps = hash:/etc/postfix/virtual
virtual_transport = lmtp:unix:private/dovecot-lmtp

# --- SASL (Dovecot) ---
smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth
smtpd_sasl_auth_enable = no
smtpd_sasl_security_options = noanonymous, noplaintext
smtpd_sasl_tls_security_options = noanonymous

# --- Remitentes autorizados ---
smtpd_sender_login_maps = hash:/etc/postfix/sender_login

# --- DKIM (OpenDKIM) ---
milter_default_action = accept
milter_protocol = 6
smtpd_milters = inet:localhost:8891
non_smtpd_milters = $smtpd_milters

# --- Límites ---
smtpd_client_connection_rate_limit = 30
smtpd_client_message_rate_limit = 100
smtpd_client_recipient_rate_limit = 200
anvil_rate_time_unit = 60s
smtpd_error_sleep_time = 5s
smtpd_soft_error_limit = 5
smtpd_hard_error_limit = 10
```

## 14.7 Conclusión

Si has llegado hasta aquí y superado las pruebas de aceptación, has construido y entiendes una infraestructura que muchas empresas pequeñas y medianas pagan por tener: varias webs aisladas con sus bases de datos, un servidor de correo con autenticación moderna, cifrado en todos los servicios, protección contra ataques, copias de seguridad y monitorización.

Lo más importante no son los comandos, que siempre podrás consultar, sino el método: **entender cada pieza, cambiar una cosa cada vez, verificar siempre y leer los logs**. Con ese método podrás administrar este servidor, ampliarlo y diagnosticar cualquier problema nuevo que aparezca.
