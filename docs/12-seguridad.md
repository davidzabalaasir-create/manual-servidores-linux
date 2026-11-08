# Módulo 12. Seguridad y mantenimiento

## Objetivos del módulo

- Dejar el cortafuegos UFW con solo los puertos imprescindibles.
- Repasar y reforzar la seguridad de SSH, MariaDB, Apache, Postfix y Dovecot.
- Proteger el servidor contra ataques de fuerza bruta con Fail2ban.
- Automatizar las actualizaciones de seguridad y las copias de seguridad.
- Monitorizar los servicios, revisar los logs y saber recuperarse de los fallos más habituales.

## 12.1 El modelo de seguridad por capas

Ninguna medida es infalible. La seguridad profesional se construye en capas: si una falla, la siguiente sigue protegiendo.

```text title="Diagrama"
  Internet
     │
  [1] Cortafuegos UFW ............ solo 22, 25, 80, 443, 465, 587, 993, 995
     │
  [2] Fail2ban ................... bloquea IP que fallan contraseñas repetidamente
     │
  [3] Servicios endurecidos ...... SSH solo con clave, TLS obligatorio, sin relay abierto
     │
  [4] Aislamiento ................ un usuario Linux, un pool PHP y un usuario BD por web
     │
  [5] Mínimo privilegio .......... MariaDB solo local, permisos 640/750/600, open_basedir
     │
  [6] Actualizaciones ............ parches de seguridad automáticos
     │
  [7] Copias de seguridad ........ si todo lo demás falla, se puede reconstruir
     │
  [8] Monitorización y logs ...... para enterarte antes que tus usuarios
```

## 12.2 Configuración definitiva de UFW

### PASO 12.1 – Revisar y ordenar las reglas del cortafuegos

**Objetivo:** comprobar que solo están abiertos los puertos necesarios.

**Comando:** 

```bash title="Terminal"
sudo ufw status numbered
```

Resultado esperado (IPv4; cada regla aparece también con `(v6)`):

```text title="Salida esperada"
Status: active
     To                         Action      From
     --                         ------      ----
[ 1] OpenSSH                    ALLOW IN    Anywhere
[ 2] Apache Full                ALLOW IN    Anywhere
[ 3] 25/tcp                     ALLOW IN    Anywhere      # SMTP recepcion
[ 4] 587/tcp                    ALLOW IN    Anywhere      # SMTP submission
[ 5] 465/tcp                    ALLOW IN    Anywhere      # SMTPS
[ 6] 993/tcp                    ALLOW IN    Anywhere      # IMAPS
[ 7] 995/tcp                    ALLOW IN    Anywhere      # POP3S
```

Tabla de referencia de puertos del servidor:

| Puerto | Servicio | Abierto en UFW | Escucha en | Motivo |
|---|---|---|---|---|
| 22/tcp | SSH | Sí | Todas | Administración (solo con clave). |
| 25/tcp | Postfix smtpd | Sí | Todas | Recepción de correo desde Internet. |
| 80/tcp | Apache | Sí | Todas | Redirección a HTTPS y validación de Let's Encrypt. |
| 443/tcp | Apache | Sí | Todas | Webs HTTPS. |
| 465/tcp | Postfix smtps | Sí | Todas | Envío autenticado (TLS implícito). |
| 587/tcp | Postfix submission | Sí | Todas | Envío autenticado (STARTTLS). |
| 993/tcp | Dovecot IMAPS | Sí | Todas | Lectura de correo. |
| 995/tcp | Dovecot POP3S | Sí (si usas POP3) | Todas | Lectura de correo. |
| 3306/tcp | MariaDB | **No** | 127.0.0.1 | Solo uso interno. |
| 8891/tcp | OpenDKIM | **No** | 127.0.0.1 | Solo uso interno. |
| 110, 143 | POP3/IMAP sin TLS | **No** | (deshabilitados) | Inseguros. |

Si no usas POP3, cierra el 995 en Dovecot (pon `port = 0` también en `pop3s`) y en UFW:

```bash title="Terminal"
sudo ufw delete allow 995/tcp
```

!!! tip "Consejo"
    **Restringir SSH a tu IP.** Si administras siempre desde una IP fija (la de tu oficina o una VPN), puedes limitar SSH a ella: `sudo ufw allow from 198.51.100.7 to any port 22 proto tcp` y después `sudo ufw delete allow OpenSSH`. Asegúrate antes de que tienes acceso a la consola web del proveedor por si cambia tu IP.

Comprueba también qué escucha realmente el servidor (lo que no aparezca aquí no es accesible aunque el cortafuegos lo permita):

```bash title="Terminal"
sudo ss -tlnp | awk 'NR==1 || /LISTEN/' | sort -k4
```

## 12.3 Protección de SSH

Ya en el Módulo 1 desactivamos el acceso de root y por contraseña. Repasemos el estado y añadamos Fail2ban más adelante.

### PASO 12.2 – Verificar el endurecimiento de SSH

**Comando:** 

```bash title="Terminal"
sudo sshd -T | grep -Ei '^(permitrootlogin|passwordauthentication|kbdinteractiveauthentication|pubkeyauthentication|maxauthtries|allowusers|x11forwarding)'
```

**Resultado esperado:** 

```text title="Salida esperada"
permitrootlogin no
pubkeyauthentication yes
passwordauthentication no
kbdinteractiveauthentication no
maxauthtries 3
x11forwarding no
allowusers sysadmin
```

Buenas prácticas adicionales:

- Protege tu clave privada con frase de paso y guarda una copia en lugar seguro.
- Si varias personas administran el servidor, cada una con su propio usuario y su propia clave (nunca compartidas). Así sabrás quién hizo qué (`/var/log/auth.log`) y podrás revocar a una persona sin afectar al resto.
- Revisa periódicamente `~/.ssh/authorized_keys` y elimina las claves que ya no se usen.
- Cambiar el puerto 22 reduce el ruido en los logs, pero no es una medida de seguridad real. En Ubuntu 24.04 SSH se activa por socket (`ssh.socket`): si cambias el puerto debes hacerlo con un *drop-in* de `ssh.socket` (`ListenStream=`), no solo en `sshd_config`.

## 12.4 Protección de MariaDB

| Medida | Comprobación |
|---|---|
| Solo escucha en local | `sudo ss -tlnp \| grep 3306` → `127.0.0.1:3306` |
| Puerto cerrado en UFW | `sudo ufw status \| grep 3306` → sin resultados |
| root solo por unix_socket | `mariadb -u root` sin sudo → `Access denied` |
| Sin usuarios anónimos ni base `test` | `sudo mariadb -e "SELECT user,host FROM mysql.user;"` |
| Sin usuarios con host `%` | En la consulta anterior no debe haber `%` |
| Privilegios mínimos por aplicación | `SHOW GRANTS FOR 'web1_user'@'localhost';` |
| Copias diarias | PASO 12.9 |

!!! warning "Atención"
    Si algún día necesitas administrar MariaDB con una herramienta gráfica desde tu ordenador, **no** abras el puerto 3306. Usa un túnel SSH: `ssh -L 3307:127.0.0.1:3306 sysadmin@mail.ejemplo1.com` y conecta tu herramienta a `127.0.0.1:3307`. El tráfico viaja cifrado por SSH y MariaDB sigue siendo invisible desde Internet.

## 12.5 Seguridad de Apache

### PASO 12.3 – Ocultar información y añadir cabeceras de seguridad

**Objetivo:** que Apache no revele su versión ni la del sistema, y que los navegadores apliquen protecciones adicionales.

**Comando:** 

```bash title="Terminal"
sudo nano /etc/apache2/conf-available/security.conf
```

**Archivo:** `/etc/apache2/conf-available/security.conf`. Localiza y deja así estas directivas (están en el archivo; cambia su valor y conserva el resto):

```text title="Archivo de configuración"
ServerTokens Prod
ServerSignature Off
TraceEnable Off
```

Después crea un archivo propio con las cabeceras de seguridad:

**Archivo:** `/etc/apache2/conf-available/cabeceras-seguridad.conf` (nuevo, contenido completo):

```text title="Archivo de configuración"
# Cabeceras de seguridad para todas las webs
<IfModule mod_headers.c>
    # Impide que el navegador "adivine" tipos de archivo
    Header always set X-Content-Type-Options "nosniff"
    # Impide que otras webs muestren las nuestras dentro de un marco (clickjacking)
    Header always set X-Frame-Options "SAMEORIGIN"
    # Limita la información de procedencia enviada a otras webs
    Header always set Referrer-Policy "strict-origin-when-cross-origin"
    # Desactiva funciones del navegador que nuestras webs no usan
    Header always set Permissions-Policy "geolocation=(), microphone=(), camera=()"
</IfModule>

# Bloquea el acceso a archivos ocultos y de copia que puedan quedar en las webs
<FilesMatch "(^\.(?!well-known)|\.(bak|old|orig|swp|sql|ini|log)$|~$)">
    Require all denied
</FilesMatch>
```

Análisis:

| Directiva | Efecto |
|---|---|
| `ServerTokens Prod` | La cabecera `Server` muestra solo `Apache`, sin versión ni sistema. |
| `ServerSignature Off` | Las páginas de error no muestran la versión. |
| `TraceEnable Off` | Desactiva el método HTTP TRACE (útil en algunos ataques). |
| `<FilesMatch ...>` | Deniega archivos que empiezan por punto (`.env`, `.git`...) salvo `.well-known`, y copias como `index.php.bak` o volcados `.sql` olvidados. |

**Comando:** 

```bash title="Terminal"
sudo a2enconf security cabeceras-seguridad
sudo apachectl configtest && sudo systemctl reload apache2
curl -sI https://www.ejemplo1.com | grep -Ei '^(server|x-content|x-frame|referrer|strict)'
```

**Resultado esperado:** 

```text title="Salida esperada"
server: Apache
strict-transport-security: max-age=15552000
x-content-type-options: nosniff
x-frame-options: SAMEORIGIN
referrer-policy: strict-origin-when-cross-origin
```

Más medidas para Apache:

- Desactiva los módulos que no uses: `sudo apachectl -M` y `sudo a2dismod autoindex status` (el módulo `status` expone información del servidor; `autoindex` genera listados de carpetas).
- Mantén `Options -Indexes` y `AllowOverride None` en todos los VirtualHosts.
- No dejes scripts de prueba (`phpinfo()`, `prueba-bd.php`) en producción.
- Mantén actualizadas las aplicaciones web (WordPress, plugins...): son la vía de entrada más frecuente.

## 12.6 Seguridad de Postfix y Dovecot

Lista de comprobación de lo ya configurado:

| Medida | Dónde | Cómo comprobarla |
|---|---|---|
| Sin relay abierto | `smtpd_relay_restrictions` | Conversación manual del PASO 6.9 → `Relay access denied`. |
| Autenticación solo con TLS y solo en 587/465 | `master.cf`, `smtpd_tls_auth_only` | PASO 10.7. |
| Remitente ligado al usuario | `reject_sender_login_mismatch` | PASO 8.2. |
| Direcciones inexistentes rechazadas en el acto | `virtual_mailbox_maps` | `noexiste@ejemplo1.com` → 550. |
| Sin `VRFY` | `disable_vrfy_command` | `nc mail.ejemplo1.com 25` → `VRFY info` → `502`. |
| IMAP/POP3 solo cifrados | `10-master.conf` | `ss -tlnp` no muestra 143 ni 110. |
| Contraseñas con hash SHA512-CRYPT | `/etc/dovecot/users` | `sudo cut -d: -f2 /etc/dovecot/users \| cut -c1-14`. |
| Buzones inaccesibles para otros usuarios | `/var/mail/vhosts` (vmail, 700) | `sudo -u web1 ls /var/mail/vhosts` → `Permission denied`. |

### PASO 12.4 – Limitar la velocidad de los clientes SMTP

**Objetivo:** dificultar que una cuenta robada o un atacante envíe miles de correos o pruebe contraseñas sin control.

**Archivo:** `/etc/postfix/main.cf`. Conserva todo y añade al final:

```text title="Archivo de configuración"
# --- Límites de conexión y envío (Módulo 12) ------------------------------
smtpd_client_connection_rate_limit = 30
smtpd_client_message_rate_limit = 100
smtpd_client_recipient_rate_limit = 200
anvil_rate_time_unit = 60s
smtpd_error_sleep_time = 5s
smtpd_soft_error_limit = 5
smtpd_hard_error_limit = 10
```

Análisis: cada cliente (IP) puede abrir como máximo 30 conexiones, enviar 100 mensajes y 200 destinatarios por minuto (`anvil_rate_time_unit`). Tras 5 errores, Postfix espera 5 segundos entre respuestas; tras 10, corta la conexión. Los clientes de `mynetworks` (el propio servidor) están exentos.

```bash title="Terminal"
sudo postfix check && sudo systemctl reload postfix
```

## 12.7 Protección contra fuerza bruta con Fail2ban

**Fail2ban** lee los logs, detecta patrones de fallos de autenticación (contraseñas incorrectas en SSH, SMTP, IMAP...) y, cuando una IP supera un número de intentos en un tiempo dado, la bloquea en el cortafuegos durante un periodo.

```text title="Diagrama"
 /var/log/auth.log, /var/log/mail.log, journal
          │ (lee continuamente)
          ▼
 FAIL2BAN ── filtro (expresiones regulares) ── ¿5 fallos de la IP X en 10 min?
          │ sí
          ▼
 Regla de cortafuegos: DROP desde IP X durante 1 hora  →  el atacante deja de llegar
```

Conceptos: un **jail** (cárcel) combina un **filtro** (qué buscar en qué log) con una **acción** (cómo bloquear) y unos parámetros: `maxretry` (intentos), `findtime` (ventana de tiempo) y `bantime` (duración del bloqueo).

### PASO 12.5 – Instalar y configurar Fail2ban

**Comando:** 

```bash title="Terminal"
sudo apt install -y fail2ban
```

!!! warning "Atención"
    Nunca edites `/etc/fail2ban/jail.conf`: se sobrescribe al actualizar. Las configuraciones propias van en `/etc/fail2ban/jail.local` o en `/etc/fail2ban/jail.d/*.local`.

**Archivo:** `/etc/fail2ban/jail.local` (nuevo, contenido completo):

```ini title="Archivo de configuración"
[DEFAULT]
# IP que nunca se bloquean (el propio servidor; añade tu IP fija si la tienes)
ignoreip = 127.0.0.1/8 ::1
# Bloqueo de 1 hora tras 5 fallos en 10 minutos
bantime  = 1h
findtime = 10m
maxretry = 5

# --- SSH ---
[sshd]
enabled  = true
backend  = systemd
maxretry = 3

# --- Postfix: clientes que provocan rechazos repetidos ---
[postfix]
enabled  = true
mode     = more
logpath  = /var/log/mail.log
backend  = auto

# --- Postfix: contraseñas SMTP incorrectas (puertos 587/465) ---
[postfix-sasl]
enabled  = true
logpath  = /var/log/mail.log
backend  = auto

# --- Dovecot: contraseñas IMAP/POP3 incorrectas ---
[dovecot]
enabled  = true
logpath  = /var/log/mail.log
backend  = auto

# --- Reincidentes: quien es bloqueado 3 veces en un día, 1 semana fuera ---
[recidive]
enabled  = true
logpath  = /var/log/fail2ban.log
bantime  = 1w
findtime = 1d
maxretry = 3
```

Análisis:

| Jail | Qué vigila |
|---|---|
| `sshd` | Fallos de inicio de sesión SSH (lee el journal de systemd). |
| `postfix` (`mode = more`) | Clientes SMTP rechazados repetidamente (relay denegado, destinatarios inexistentes, HELO inválido...). |
| `postfix-sasl` | `SASL LOGIN authentication failed` en los puertos de envío. |
| `dovecot` | `auth failed` en IMAP/POP3. |
| `recidive` | El propio log de Fail2ban: castiga a los reincidentes con un bloqueo largo. |

**Comando:** 

```bash title="Terminal"
sudo fail2ban-client -t
sudo systemctl enable --now fail2ban
sudo systemctl restart fail2ban
sudo fail2ban-client status
```

**Resultado esperado:** 

```text title="Salida esperada"
OK: configuration test is successful
Status
|- Number of jail:      5
`- Jail list:   dovecot, postfix, postfix-sasl, recidive, sshd
```

**Comprobación:** consulta un jail concreto:

```bash title="Terminal"
sudo fail2ban-client status postfix-sasl
```

```text title="Salida esperada"
Status for the jail: postfix-sasl
|- Filter
|  |- Currently failed: 1
|  |- Total failed:     14
|  `- File list:        /var/log/mail.log
`- Actions
   |- Currently banned: 2
   |- Total banned:     6
   `- Banned IP list:   192.0.2.45 198.51.100.200
```

Operaciones habituales:

| Tarea | Comando |
|---|---|
| Ver todos los jails | `sudo fail2ban-client status` |
| Desbloquear una IP | `sudo fail2ban-client set sshd unbanip 198.51.100.7` |
| Bloquear manualmente | `sudo fail2ban-client set sshd banip 192.0.2.45` |
| Probar un filtro contra un log | `sudo fail2ban-regex /var/log/mail.log /etc/fail2ban/filter.d/dovecot.conf` |
| Ver el log de Fail2ban | `sudo tail -f /var/log/fail2ban.log` |

**Errores habituales:** 

- `Have not found any log file for ... jail`: el archivo indicado en `logpath` no existe. Comprueba que `rsyslog` está instalado y activo (`systemctl status rsyslog`); en Debian 12, que no trae rsyslog por defecto, usa `backend = systemd` también en los jails de correo.
- Fail2ban no arranca en Ubuntu 24.04 recién instalado con un error de Python (`ModuleNotFoundError: No module named 'asynchat'`): instala las actualizaciones pendientes (`sudo apt update && sudo apt full-upgrade`), que incluyen la versión corregida del paquete.
- Te bloqueas a ti mismo: entra por la consola web del proveedor y usa `unbanip`; añade tu IP a `ignoreip`.

## 12.8 Actualizaciones automáticas de seguridad

### PASO 12.6 – Configurar unattended-upgrades

**Objetivo:** que los parches de seguridad se instalen solos cada día.

**Comando:** 

```bash title="Terminal"
sudo apt install -y unattended-upgrades apt-listchanges
sudo dpkg-reconfigure -plow unattended-upgrades
```

Responde **Yes** a «Automatically download and install stable updates?». Esto crea `/etc/apt/apt.conf.d/20auto-upgrades`.

Para recibir un aviso por correo cuando se instalen actualizaciones, edita:

**Archivo:** `/etc/apt/apt.conf.d/50unattended-upgrades`. Busca estas líneas (están comentadas con `//`), quita las barras y deja estos valores:

```text title="Archivo de configuración"
Unattended-Upgrade::Mail "root";
Unattended-Upgrade::MailReport "on-change";
Unattended-Upgrade::Remove-Unused-Dependencies "true";
Unattended-Upgrade::Automatic-Reboot "false";
```

Análisis: `Mail "root"` envía el informe a root (que es alias de `administracion@ejemplo1.com`); `on-change` solo cuando se instala algo; `Automatic-Reboot "false"` evita reinicios sorpresa: tú decides cuándo reiniciar (el archivo `/var/run/reboot-required` te avisa de que hace falta).

**Comprobación:** 

```bash title="Terminal"
cat /etc/apt/apt.conf.d/20auto-upgrades
sudo unattended-upgrade --dry-run --debug 2>&1 | tail -5
```

```text title="Salida esperada"
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
```

!!! tip "Consejo"
    Las actualizaciones de versión mayor (por ejemplo, de Ubuntu 24.04 a 26.04) **nunca** son automáticas. Requieren planificación, copia de seguridad completa y revisión de los cambios (en este caso, el paso a Dovecot 2.4).

## 12.9 Copias de seguridad automáticas

Regla **3-2-1**: al menos 3 copias de los datos, en 2 soportes distintos, y 1 fuera del servidor (en otro centro de datos). Una copia que solo está en el mismo disco no te protege de un fallo del disco, de un borrado del proveedor ni de un ataque de *ransomware*.

Qué hay que copiar:

| Qué | Ruta | Por qué |
|---|---|---|
| Bases de datos | (volcado con `mariadb-dump`) | Los archivos de `/var/lib/mysql` no se pueden copiar en caliente de forma fiable. |
| Webs | `/var/www/` | Código, configuración y archivos subidos. |
| Buzones | `/var/mail/vhosts/` | Todo el correo. |
| Configuración | `/etc/` | Apache, PHP, Postfix, Dovecot, OpenDKIM, usuarios de correo... |
| Certificados y cuenta ACME | `/etc/letsencrypt/` (incluido en `/etc`) | Evita reemitir y conserva la cuenta. |
| Claves DKIM | `/etc/opendkim/` (incluido en `/etc`) | Si las pierdes, tendrás que publicar claves nuevas en el DNS. |
| Scripts propios | `/usr/local/sbin/` | Tus herramientas de administración. |

### PASO 12.7 – Crear el script de copia de seguridad

**Archivo:** `/usr/local/sbin/backup-servidor` (nuevo, contenido completo):

```bash title="Terminal"
#!/bin/bash
# =====================================================================
#  backup-servidor - Copia diaria de bases de datos, webs, correo y configuración
#  Se ejecuta desde cron. Conserva los últimos DIAS días.
# =====================================================================
set -euo pipefail

DESTINO=/var/backups/servidor
DIAS=7
FECHA=$(date +%F)
DIR="$DESTINO/$FECHA"
LOG=/var/log/backup-servidor.log

exec >>"$LOG" 2>&1
echo "=== $(date '+%F %T') Inicio de la copia ==="

umask 077
mkdir -p "$DIR"

# 1. Bases de datos: un archivo por base (excepto las internas)
for BD in $(mariadb -N -e "SHOW DATABASES" | grep -Ev '^(information_schema|performance_schema|sys)$'); do
    mariadb-dump --single-transaction --routines --triggers --events --databases "$BD" | gzip > "$DIR/bd_${BD}.sql.gz"
    echo "  Base de datos $BD copiada"
done

# 2. Archivos: webs, buzones, configuración y scripts
tar -czf "$DIR/www.tar.gz"     -C / var/www
tar -czf "$DIR/correo.tar.gz"  -C / var/mail/vhosts
tar -czf "$DIR/etc.tar.gz"     -C / etc
tar -czf "$DIR/scripts.tar.gz" -C / usr/local/sbin
echo "  Archivos copiados"

# 3. Lista de paquetes instalados (para reconstruir el servidor)
dpkg --get-selections > "$DIR/paquetes.txt"

# 4. Suma de verificación para detectar copias corruptas
( cd "$DIR" && sha256sum ./* > SHA256SUMS )

# 5. Borrar copias antiguas
find "$DESTINO" -mindepth 1 -maxdepth 1 -type d -mtime +"$DIAS" -exec rm -rf {} +

echo "  Tamaño de la copia: $(du -sh "$DIR" | cut -f1)"
echo "=== $(date '+%F %T') Copia terminada correctamente ==="
```

Análisis:

| Línea | Qué hace |
|---|---|
| `exec >>"$LOG" 2>&1` | Todo lo que el script escriba (incluidos los errores) va al log. |
| `umask 077` | Los archivos creados solo los puede leer root. |
| `mariadb -N -e "SHOW DATABASES"` | Lista las bases sin cabecera (`-N`); así las nuevas se copian automáticamente. |
| `tar -czf archivo -C / var/www` | Comprime la carpeta guardando rutas relativas (`var/www/...`), lo que facilita restaurar. |
| `sha256sum` | Huellas para verificar la integridad de la copia antes de restaurarla. |
| `find ... -mtime +7 -exec rm -rf` | Elimina las carpetas de más de 7 días. |

**Comando:** 

```bash title="Terminal"
sudo chmod 700 /usr/local/sbin/backup-servidor
sudo /usr/local/sbin/backup-servidor
sudo tail -n 15 /var/log/backup-servidor.log
sudo ls -lh /var/backups/servidor/$(date +%F)/
```

**Resultado esperado:** 

```text title="Salida esperada"
=== 2026-11-09 04:30:00 Inicio de la copia ===
  Base de datos mysql copiada
  Base de datos web1_db copiada
  Base de datos web2_db copiada
  Archivos copiados
  Tamaño de la copia: 14M
=== 2026-11-09 04:30:09 Copia terminada correctamente ===
```

### PASO 12.8 – Programar la copia diaria con cron

**Archivo:** `/etc/cron.d/backup-servidor` (nuevo, contenido completo):

```text title="Tarea programada (cron)"
# Copia de seguridad diaria a las 03:30
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
MAILTO=root
30 3 * * * root /usr/local/sbin/backup-servidor || echo "La copia de seguridad ha fallado. Revisa /var/log/backup-servidor.log"
```

Formato de una línea de cron:

```text title="Diagrama"
 30   3    *    *    *    root  comando
 │    │    │    │    │     │
 │    │    │    │    │     └─ usuario que lo ejecuta (solo en /etc/cron.d)
 │    │    │    │    └─────── día de la semana (0-7, 0 y 7 = domingo)
 │    │    │    └──────────── mes (1-12)
 │    │    └───────────────── día del mes (1-31)
 │    └────────────────────── hora (0-23)
 └─────────────────────────── minuto (0-59)
```

Si el script falla, el `|| echo ...` produce una salida y cron la envía por correo a `MAILTO` (root → `administracion@ejemplo1.com`).

```bash title="Terminal"
sudo chmod 644 /etc/cron.d/backup-servidor
```

### PASO 12.9 – Sacar las copias del servidor

**Objetivo:** cumplir la parte «1 copia fuera» de la regla 3-2-1.

**Explicación:** la forma más sencilla es que **otro** equipo (un segundo servidor o un NAS) descargue las copias por SSH cada día con `rsync`. Así, si alguien compromete tu servidor, no tiene credenciales para borrar las copias externas. Ejemplo, ejecutado **en el equipo de copias**, con un usuario que tenga una clave autorizada en el servidor:

```bash title="Terminal"
rsync -az --delete -e ssh sysadmin@mail.ejemplo1.com:/var/backups/servidor/ /srv/copias/mail.ejemplo1.com/
```

Para que `sysadmin` pueda leer las copias sin ser root, cambia el grupo de la carpeta de destino del script y su `umask` (por ejemplo, `umask 027` y `chgrp -R sysadmin "$DESTINO"`), o usa un usuario dedicado de solo lectura. Otra opción muy extendida es `rclone`, que envía las copias cifradas a un almacenamiento de objetos.

!!! warning "Atención"
    Prueba la restauración. Al menos una vez al mes, restaura en una máquina de pruebas una base de datos, una web y un buzón. Comprueba antes las sumas con `sha256sum -c SHA256SUMS`.

## 12.10 Monitorización de servicios

### PASO 12.10 – Script de estado del servidor

**Objetivo:** tener un único comando que revise todo y avise por correo si algo va mal.

**Archivo:** `/usr/local/sbin/estado-servidor` (nuevo, contenido completo):

```bash title="Terminal"
#!/bin/bash
# =====================================================================
#  estado-servidor - Revisión rápida de la salud del servidor
#  Uso: sudo estado-servidor            (muestra el informe)
#       sudo estado-servidor --avisar   (solo envía correo a root si hay problemas)
# =====================================================================
SERVICIOS="apache2 php8.3-fpm mariadb postfix@- dovecot opendkim fail2ban ssh cron"
PUERTOS="22 25 80 443 465 587 993"
LIMITE_DISCO=85
DIAS_CERT=20
PROBLEMAS=()

informe() { [ "${1:-}" = "--avisar" ] || echo "$2"; }

for S in $SERVICIOS; do
    if systemctl is-active --quiet "$S"; then informe "$1" "[OK]    servicio $S activo"
    else PROBLEMAS+=("Servicio $S detenido"); informe "$1" "[FALLO] servicio $S detenido"; fi
done

for P in $PUERTOS; do
    if ss -tln | awk '{print $4}' | grep -qE "[:.]$P$"; then informe "$1" "[OK]    puerto $P escuchando"
    else PROBLEMAS+=("Puerto $P sin escuchar"); informe "$1" "[FALLO] puerto $P sin escuchar"; fi
done

while read -r USO MONTAJE; do
    USO=${USO%\%}
    if [ "$USO" -ge "$LIMITE_DISCO" ]; then PROBLEMAS+=("Disco $MONTAJE al $USO%"); informe "$1" "[FALLO] disco $MONTAJE al $USO%"
    else informe "$1" "[OK]    disco $MONTAJE al $USO%"; fi
done < <(df -P -x tmpfs -x devtmpfs -x squashfs -x overlay | awk 'NR>1 {print $5, $6}')

for C in /etc/letsencrypt/live/*/cert.pem; do
    [ -e "$C" ] || continue
    N=$(basename "$(dirname "$C")")
    if openssl x509 -checkend $((DIAS_CERT*86400)) -noout -in "$C" >/dev/null; then informe "$1" "[OK]    certificado $N vigente más de $DIAS_CERT días"
    else PROBLEMAS+=("Certificado $N caduca en menos de $DIAS_CERT días"); informe "$1" "[FALLO] certificado $N caduca pronto"; fi
done

COLA=$(postqueue -j 2>/dev/null | wc -l)
if [ "$COLA" -gt 50 ]; then PROBLEMAS+=("Cola de correo con $COLA mensajes"); informe "$1" "[FALLO] cola de correo: $COLA mensajes"
else informe "$1" "[OK]    cola de correo: $COLA mensajes"; fi

[ -f /var/run/reboot-required ] && informe "$1" "[AVISO] el sistema necesita reiniciarse"

if [ "${#PROBLEMAS[@]}" -gt 0 ] && [ "${1:-}" = "--avisar" ]; then
    printf '%s\n' "${PROBLEMAS[@]}" | mail -s "[$(hostname)] ${#PROBLEMAS[@]} problema(s) detectado(s)" root
fi
[ "${#PROBLEMAS[@]}" -eq 0 ]
```

**Comando:** 

```bash title="Terminal"
sudo chmod 700 /usr/local/sbin/estado-servidor
sudo estado-servidor
```

**Resultado esperado:** 

```text title="Salida esperada"
[OK]    servicio apache2 activo
[OK]    servicio php8.3-fpm activo
[OK]    servicio mariadb activo
[OK]    servicio postfix@- activo
[OK]    servicio dovecot activo
[OK]    servicio opendkim activo
[OK]    servicio fail2ban activo
[OK]    servicio ssh activo
[OK]    servicio cron activo
[OK]    puerto 22 escuchando
...
[OK]    disco / al 18%
[OK]    certificado ejemplo1.com vigente más de 20 días
[OK]    cola de correo: 0 mensajes
```

Prográmalo cada 15 minutos en modo aviso:

**Archivo:** `/etc/cron.d/estado-servidor` (nuevo, contenido completo):

```text title="Tarea programada (cron)"
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
*/15 * * * * root /usr/local/sbin/estado-servidor --avisar >/dev/null 2>&1
```

!!! warning "Atención"
    Un servidor no puede avisar de que él mismo está caído o sin red. Complementa este script con un servicio de monitorización **externo** (hay opciones gratuitas de *uptime monitoring*) que compruebe cada pocos minutos `https://www.ejemplo1.com`, `https://www.ejemplo2.com` y el puerto 25 desde fuera.

Para que systemd reinicie automáticamente un servicio si se cae, puedes añadir un *drop-in* (ejemplo con Dovecot):

```bash title="Terminal"
sudo mkdir -p /etc/systemd/system/dovecot.service.d
printf '[Service]\nRestart=on-failure\nRestartSec=5s\n' | sudo tee /etc/systemd/system/dovecot.service.d/reinicio.conf
sudo systemctl daemon-reload
```

## 12.11 Revisión de logs

| Log | Contenido | Comando típico |
|---|---|---|
| `/var/log/auth.log` | SSH, sudo, inicios de sesión. | `sudo grep 'Failed\|Accepted' /var/log/auth.log \| tail` |
| `/var/log/mail.log` | Postfix, Dovecot, OpenDKIM. | `sudo grep -E 'reject\|warning\|error' /var/log/mail.log \| tail -30` |
| `/var/log/apache2/*-error.log` | Errores de cada web. | `sudo tail -n 50 /var/log/apache2/ejemplo1.com-error.log` |
| `/var/log/apache2/*-access.log` | Visitas de cada web. | `sudo awk '{print $9}' /var/log/apache2/ejemplo1.com-access.log \| sort \| uniq -c` |
| `/var/www/*/logs/php-error.log` | Errores PHP de cada web. | `sudo tail /var/www/ejemplo1.com/logs/php-error.log` |
| `/var/log/mysql/error.log` | Errores de MariaDB. | `sudo tail /var/log/mysql/error.log` |
| `/var/log/fail2ban.log` | Bloqueos. | `sudo grep Ban /var/log/fail2ban.log \| tail` |
| `/var/log/php8.3-fpm.log` | Gestor PHP-FPM. | `sudo tail /var/log/php8.3-fpm.log` |
| journal de systemd | Todo, por servicio. | `sudo journalctl -u dovecot --since today` |

Comandos de journal especialmente útiles:

```bash title="Terminal"
# Errores de todo el sistema desde el último arranque
sudo journalctl -p err -b
# Seguir en directo varios servicios a la vez
sudo journalctl -f -u apache2 -u php8.3-fpm -u postfix@- -u dovecot
# Espacio que ocupa el journal y limpiarlo a 500 MB
journalctl --disk-usage
sudo journalctl --vacuum-size=500M
```

Rutina de revisión recomendada:

| Frecuencia | Tarea |
|---|---|
| Diaria (automática) | Copia de seguridad, actualizaciones de seguridad, `estado-servidor --avisar`. |
| Semanal | Revisar `mail.log` (rechazos, colas), `fail2ban-client status`, errores de Apache y PHP, informes DMARC. |
| Mensual | Prueba de restauración, revisar usuarios y claves SSH, espacio en disco, `certbot certificates`, listas negras de la IP. |
| Trimestral | Revisar versiones de las aplicaciones web, rotar contraseñas sensibles, revisar reglas de UFW. |

## 12.12 Recuperación ante fallos

| Situación | Procedimiento |
|---|---|
| **Un servicio no arranca** | 1. `systemctl status servicio` y `journalctl -u servicio -n 50`. 2. Prueba la configuración (`apachectl configtest`, `postfix check`, `doveconf -n`, `php-fpm8.3 -t`, `sshd -t`). 3. Si cambiaste algo, restaura la copia `.original`/`.bak` del archivo. |
| **Disco lleno** | `df -h` para ver la partición; `sudo du -xh / --max-depth=2 \| sort -h \| tail` para localizar qué ocupa. Causas típicas: logs, copias antiguas, cola de correo, journal. Nunca borres archivos de `/var/lib/mysql` a mano. |
| **Se ha borrado una web** | `sudo tar -xzf /var/backups/servidor/FECHA/www.tar.gz -C /tmp/restaura var/www/ejemplo1.com` y copia lo necesario con `rsync -a` a su sitio; revisa propietarios (`web1:www-data`). |
| **Se ha dañado una base de datos** | PASO 4.13 con el volcado `bd_web1_db.sql.gz` del día elegido. |
| **Un usuario ha borrado correos** | Extrae su carpeta de `correo.tar.gz` en `/tmp`, copia los archivos de `cur/` a su Maildir, ajusta propietario `vmail:vmail` y ejecuta `sudo doveadm force-resync -u usuario '*'`. |
| **Contraseña de correo robada** (envía spam) | `sudo correo-clave usuario`; vacía la cola de spam (`mailq`, `postsuper -d`); revisa listas negras; avisa al usuario de que revise su equipo. |
| **Servidor comprometido** | No intentes «limpiarlo». Aísla el servidor, conserva evidencias, monta uno nuevo desde cero (Módulo 14) y restaura **solo datos** (bases de datos, contenidos, buzones) de copias anteriores al incidente, revisándolos. Cambia todas las contraseñas y claves (incluidas las DKIM). |
| **Reconstrucción completa** | Instala Ubuntu, sigue los módulos 1-12 o restaura `/etc` de forma selectiva, reinstala los paquetes (`sudo dpkg --set-selections < paquetes.txt && sudo apt-get dselect-upgrade`), restaura bases de datos, `/var/www` y `/var/mail/vhosts`, y comprueba todo con el Módulo 13. |

## 12.13 Ejercicios prácticos del Módulo 12

1. Desde otra IP, falla tres veces la contraseña SSH (o la de IMAP) y comprueba con `fail2ban-client status` que la IP queda bloqueada. Desbloquéala.
2. Ejecuta `sudo estado-servidor` con Dovecot detenido. ¿Qué muestra? ¿Qué correo llega a root con `--avisar`?
3. Restaura en `/tmp/prueba` el buzón de `soporte@ejemplo2.com` de la copia de hoy y comprueba que los mensajes están completos.
4. Calcula cuánto espacio ocuparán 7 días de copias y decide si te conviene una partición o disco aparte.
5. Usa `curl -sI` para comprobar que ninguna de tus webs revela la versión de Apache ni de PHP (busca la cabecera `X-Powered-By`; si aparece, añade `php_admin_flag[expose_php] = off` en cada pool).
