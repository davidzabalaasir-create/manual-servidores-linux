# Módulo 15. Servidor FTP con vsftpd (complementario)

## Objetivos del módulo

- Entender cómo funciona FTP, sus modos activo y pasivo y sus riesgos de seguridad.
- Decidir con criterio entre FTP cifrado (FTPS) y SFTP.
- Instalar y configurar **vsftpd** para que cada web tenga su propio acceso FTP cifrado.
- Encerrar a cada usuario en la carpeta de su web (*chroot*) sin debilitar la seguridad.
- Obtener un certificado para `ftp.ejemplo1.com` y `ftp.ejemplo2.com` e integrarlo en la renovación automática.
- Proteger el servicio con UFW y Fail2ban, y probarlo desde la línea de comandos y desde FileZilla.

!!! tip "Consejo"
    Este módulo amplía la infraestructura del proyecto final. Requiere haber completado los módulos 1 a 5 (usuarios `web1`/`web2` y carpetas de las webs), 10 (Certbot) y 12 (UFW y Fail2ban).

## 15.1 Qué es FTP y para qué lo queremos

**FTP** (*File Transfer Protocol*) es uno de los protocolos más antiguos de Internet (1971). Sirve para subir y descargar archivos entre un cliente y un servidor. En un alojamiento web se usa para que la persona responsable de cada web pueda **subir sus archivos** (HTML, imágenes, una aplicación PHP) sin tener acceso de administrador al servidor.

Nuestro objetivo:

| Usuario FTP | Accede a | Lo que ve como «/» | Puede escribir en |
|---|---|---|---|
| `web1` | ejemplo1.com | `/var/www/ejemplo1.com` | `public_html`, `app`, `config`, `tmp`, `logs` |
| `web2` | ejemplo2.com | `/var/www/ejemplo2.com` | Las mismas carpetas de su web |
| `sysadmin`, `root` y cualquier otro | — | Acceso denegado | — |

Así, los archivos subidos pertenecen automáticamente al usuario de la web (`web1`), que es el mismo con el que se ejecuta su PHP. No hay que corregir propietarios después de cada subida.

### Cómo funciona FTP: dos conexiones

A diferencia de HTTP, FTP usa **dos** conexiones TCP:

- La **conexión de control** (puerto 21): por ella viajan los comandos (`USER`, `PASS`, `LIST`, `STOR`...) y las respuestas.
- La **conexión de datos**: se abre una nueva para cada listado de carpeta y cada archivo transferido.

La conexión de datos puede abrirse de dos formas:

```text title="Diagrama"
 MODO ACTIVO (el servidor llama al cliente) – problemático con NAT y cortafuegos
   Cliente:5001 ──── control ────► Servidor:21     "PORT 192.168.1.5,19,137"
   Cliente:5001 ◄─── datos ─────── Servidor:20     el servidor se conecta al cliente

 MODO PASIVO (el cliente llama al servidor) – el que usaremos
   Cliente:5001 ──── control ────► Servidor:21     "PASV"
                                                     "227 Entering Passive Mode (...,156,64)"
   Cliente:5002 ──── datos ──────► Servidor:40000  el servidor abre un puerto del rango pasivo
```

En **modo pasivo** es el cliente quien abre las dos conexiones, lo que funciona detrás de los routers domésticos. A cambio, el servidor necesita un **rango de puertos** abierto en el cortafuegos para las conexiones de datos (usaremos 40000–40100).

### El problema de seguridad de FTP

El FTP clásico envía **todo en texto plano**, incluidos el usuario y la contraseña. Cualquiera que pueda observar el tráfico (una red Wi-Fi pública, un equipo comprometido en el camino) las lee directamente. Por eso **nunca** debe usarse FTP sin cifrar en un servidor público.

Hay dos alternativas seguras:

| Característica | FTPS (FTP sobre TLS) | SFTP (SSH File Transfer Protocol) |
|---|---|---|
| Qué es | FTP de siempre con cifrado TLS (como HTTPS). | Un protocolo distinto que funciona dentro de SSH. |
| Servidor | vsftpd | OpenSSH (ya instalado) |
| Puertos | 21 + rango pasivo (40000–40100) | Solo el 22 |
| Certificado | Sí (Let's Encrypt) | No (usa las claves del servidor SSH) |
| Compatibilidad | Cualquier cliente FTP con «FTP explícito sobre TLS» | FileZilla, WinSCP, Cyberduck, `sftp`, `scp`... |
| Complejidad | Mayor (dos conexiones, rango pasivo, NAT) | Menor |

!!! tip "Consejo"
    **Recomendación profesional:** si puedes elegir, SFTP es más sencillo de proteger porque solo usa el puerto 22. FTPS sigue siendo muy útil cuando los usuarios o las herramientas (editores web, aplicaciones antiguas, algunos servicios de despliegue) solo hablan FTP. En este módulo montamos **FTPS con vsftpd** y en el apartado 15.10 verás cómo ofrecer SFTP con la misma separación por webs.

### vsftpd

**vsftpd** (*Very Secure FTP Daemon*) es el servidor FTP de referencia en Linux: pequeño, rápido y diseñado con la seguridad como prioridad (separación de privilegios, *chroot*, procesos sin permisos). Ubuntu 24.04 incluye vsftpd 3.0.5.

## 15.2 Preparar el DNS y el certificado

### PASO 15.1 – Crear los registros DNS de FTP

**Objetivo:** que los usuarios se conecten a un nombre propio de su dominio.

Registros (en el panel DNS de cada dominio):

| Dominio | Tipo | Nombre | Valor |
|---|---|---|---|
| ejemplo1.com | A | ftp | 203.0.113.10 |
| ejemplo2.com | A | ftp | 203.0.113.10 |

**Comprobación:** 

```bash title="Terminal"
dig +short A ftp.ejemplo1.com
dig +short A ftp.ejemplo2.com
```

Ambos deben devolver `203.0.113.10`.

### PASO 15.2 – Obtener el certificado de ftp.ejemplo1.com y ftp.ejemplo2.com

**Objetivo:** tener un certificado válido para los dos nombres FTP.

**Explicación:** igual que con el correo (PASO 6.2), validamos a través del sitio comodín de Apache. Los nombres `ftp.*` no están en ningún VirtualHost, así que Apache los atiende con `000-default.conf`, cuyo DocumentRoot es `/var/www/html`.

**Comando:** 

```bash title="Terminal"
sudo certbot certonly --webroot -w /var/www/html -d ftp.ejemplo1.com -d ftp.ejemplo2.com --email administracion@ejemplo1.com --agree-tos --no-eff-email
```

**Resultado esperado:** 

```text title="Salida esperada"
Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/ftp.ejemplo1.com/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/ftp.ejemplo1.com/privkey.pem
```

**Errores habituales:** los mismos del PASO 3.11 (DNS aún no propagado, puerto 80 cerrado).

!!! tip "Consejo"
    **Laboratorio sin dominio real:** usa en el PASO 15.6 el certificado autofirmado (`/etc/ssl/certs/ssl-cert-snakeoil.pem` y `/etc/ssl/private/ssl-cert-snakeoil.key`). Los clientes FTP te pedirán aceptar el certificado la primera vez.

## 15.3 Instalación de vsftpd

### PASO 15.3 – Instalar vsftpd

**Comando:** 

```bash title="Terminal"
sudo apt install -y vsftpd acl
vsftpd -v
systemctl status vsftpd --no-pager
```

**Análisis del comando:** `vsftpd` es el servidor; `acl` aporta los comandos `setfacl` y `getfacl`, que necesitaremos en el PASO 15.5.

**Resultado esperado:** 

```text title="Salida esperada"
vsftpd: version 3.0.5
● vsftpd.service - vsftpd FTP server
     Loaded: loaded (/usr/lib/systemd/system/vsftpd.service; enabled; preset: enabled)
     Active: active (running) since ...
```

!!! warning "Atención"
    Tras la instalación, vsftpd arranca con una configuración por defecto que **todavía no cifra**. El puerto 21 aún está cerrado en UFW, así que no es accesible desde fuera; no lo abras hasta terminar el PASO 15.7.

Archivos de vsftpd en Ubuntu:

| Archivo | Contenido |
|---|---|
| `/etc/vsftpd.conf` | Configuración principal. |
| `/etc/pam.d/vsftpd` | Cómo se autentican los usuarios (PAM). |
| `/etc/ftpusers` | Usuarios a los que PAM **prohíbe** el FTP (root, daemon...). |
| `/etc/vsftpd.userlist` | (Lo crearemos) usuarios a los que vsftpd **permite** el FTP. |
| `/var/log/vsftpd.log` | Registro de conexiones y transferencias. |

## 15.4 Usuarios FTP y jaula chroot

### Por qué usamos los usuarios de las webs

Podríamos crear usuarios FTP aparte (`ftp_web1`...), pero entonces los archivos subidos tendrían un propietario distinto del que ejecuta PHP y habría que ajustar permisos constantemente. Usar directamente `web1` y `web2` mantiene el diseño del Módulo 3: **un usuario por web, dueño de todos sus archivos**.

Recuerda que estos usuarios tienen como shell `/usr/sbin/nologin` y que SSH solo admite a `sysadmin` (`AllowUsers`, PASO 1.7). Darles contraseña les permitirá **solo** usar FTP: no podrán iniciar sesión en el servidor.

### PASO 15.4 – Asignar contraseña a los usuarios FTP

**Objetivo:** que `web1` y `web2` puedan autenticarse en FTP.

**Comando:** 

```bash title="Terminal"
sudo passwd web1
sudo passwd web2
```

Elige contraseñas largas y distintas (puedes generarlas con `openssl rand -base64 18`).

**Comprobación:** el shell sigue siendo `nologin` y SSH sigue cerrado para ellos:

```bash title="Terminal"
getent passwd web1
sudo sshd -T | grep -i allowusers
```

```text title="Salida esperada"
web1:x:998:998::/var/www/ejemplo1.com:/usr/sbin/nologin
allowusers sysadmin
```

### PASO 15.5 – Preparar la carpeta de cada web como jaula chroot

**Objetivo:** que cada usuario quede encerrado en la carpeta de su web y que vsftpd acepte esa carpeta como raíz.

**Explicación:** con `chroot_local_user=YES`, vsftpd encierra a cada usuario en su carpeta personal (*home*), que para `web1` es `/var/www/ejemplo1.com`. Por seguridad, vsftpd **se niega** a usar como raíz de la jaula una carpeta en la que el propio usuario puede escribir, y da este error al conectar:

```text title="Salida esperada"
500 OOPS: vsftpd: refusing to run with writable root inside chroot()
```

El motivo es que un usuario que puede escribir en la raíz de la jaula podría crear allí carpetas como `etc` o `lib` y engañar a programas del sistema que se ejecuten dentro de ella. Existe la opción `allow_writeable_chroot=YES` para saltarse la comprobación, pero **no la usaremos**: hacemos que la raíz no sea escribible y mantenemos la protección.

Ahora `/var/www/ejemplo1.com` pertenece a `web1` (PASO 3.2). La cambiamos a `root`, conservando el acceso de Apache (grupo `www-data`) y dando a `web1` permiso de **lectura y paso** (sin escritura) mediante una **ACL** (*Access Control List*, lista de control de acceso): un permiso adicional para un usuario concreto, más allá de propietario, grupo y otros.

**Comando:** 

```bash title="Terminal"
sudo chown root:www-data /var/www/ejemplo1.com /var/www/ejemplo2.com
sudo chmod 750 /var/www/ejemplo1.com /var/www/ejemplo2.com
sudo setfacl -m u:web1:rx /var/www/ejemplo1.com
sudo setfacl -m u:web2:rx /var/www/ejemplo2.com
getfacl /var/www/ejemplo1.com
```

**Análisis del comando:** 

| Orden | Efecto |
|---|---|
| `chown root:www-data` | El dueño pasa a ser root (nadie más puede escribir en la raíz); el grupo sigue siendo `www-data`. |
| `chmod 750` | root: todo; `www-data`: entrar y listar (Apache sigue llegando a `public_html`); otros (por ejemplo `web2`): nada. |
| `setfacl -m u:web1:rx` | Añade (`-m`, *modify*) una entrada para el usuario `web1` con lectura y paso, sin escritura. |
| `getfacl` | Muestra todos los permisos, incluidas las ACL. |

**Resultado esperado:** 

```text title="Salida esperada"
# file: var/www/ejemplo1.com
# owner: root
# group: www-data
user::rwx
user:web1:r-x
group::r-x
mask::r-x
other::---
```

Las subcarpetas **no cambian**: `public_html` sigue siendo de `web1` (con setgid y grupo `www-data`), y `config`, `app`, `tmp` y `logs` siguen siendo de `web1` con permisos `700`. Por tanto `web1` puede escribir en todas ellas, pero no crear archivos sueltos en la raíz de su web (no lo necesita).

**Comprobación:** la web y PHP deben seguir funcionando exactamente igual:

```bash title="Terminal"
sudo -u www-data ls /var/www/ejemplo1.com/public_html > /dev/null && echo "Apache llega a public_html"
sudo -u web1 ls /var/www/ejemplo1.com && echo "web1 puede listar su carpeta"
sudo -u web1 touch /var/www/ejemplo1.com/prueba 2>&1 | head -1
sudo -u web2 ls /var/www/ejemplo1.com 2>&1 | head -1
curl -s -o /dev/null -w "%{http_code}\n" https://www.ejemplo1.com/
```

```text title="Salida esperada"
Apache llega a public_html
app  config  logs  public_html  tmp
web1 puede listar su carpeta
touch: cannot touch '/var/www/ejemplo1.com/prueba': Permission denied
ls: cannot open directory '/var/www/ejemplo1.com': Permission denied
200
```

**Errores habituales:** `setfacl: command not found` (falta el paquete `acl`); si la web da 403 o 503 tras el cambio, revisa con `namei -l /var/www/ejemplo1.com/public_html/index.php` y con `getfacl` que el grupo es `www-data` y la ACL de `web1` existe.

### PASO 15.6 – Permitir el inicio de sesión a usuarios con shell nologin

**Objetivo:** que PAM no rechace a `web1` y `web2` por tener `/usr/sbin/nologin` como shell.

**Explicación:** el archivo PAM de vsftpd en Ubuntu termina con la línea `auth required pam_shells.so`, que solo deja entrar a usuarios cuyo shell esté en `/etc/shells`. `nologin` no está (y no debe añadirse, porque otros servicios usan esa lista). Desactivamos esa comprobación **solo para vsftpd**; el control de quién puede usar FTP lo haremos con una lista explícita (PASO 15.7).

**Comando:** 

```bash title="Terminal"
sudo cp /etc/pam.d/vsftpd /etc/pam.d/vsftpd.original
sudo sed -i 's/^auth\s\+required\s\+pam_shells.so/# auth required pam_shells.so  (desactivado: los usuarios FTP usan nologin)/' /etc/pam.d/vsftpd
cat /etc/pam.d/vsftpd
```

**Resultado esperado:** 

```text title="Salida esperada"
# Standard behaviour for ftpd(8).
auth	required	pam_listfile.so item=user sense=deny file=/etc/ftpusers onerr=succeed

# Note: vsftpd handles anonymous logins on its own. Do not enable pam_ftp.so.

# Standard pam includes
@include common-account
@include common-session
@include common-auth
# auth required pam_shells.so  (desactivado: los usuarios FTP usan nologin)
```

La primera línea se mantiene: los usuarios de `/etc/ftpusers` (root y las cuentas del sistema) siguen sin poder usar FTP.

### PASO 15.7 – Crear la lista de usuarios autorizados

**Objetivo:** que **solo** `web1` y `web2` puedan conectarse por FTP.

**Archivo:** `/etc/vsftpd.userlist` (nuevo, contenido completo):

```text title="Archivo de configuración"
web1
web2
```

```bash title="Terminal"
sudo chmod 644 /etc/vsftpd.userlist
```

Con la configuración del paso siguiente (`userlist_deny=NO`), esta lista funciona como «lista blanca»: cualquier otro usuario (incluido `sysadmin`) es rechazado **antes** de pedir la contraseña, así que su contraseña nunca viaja por FTP.

## 15.5 Configuración de vsftpd

### PASO 15.8 – Escribir la configuración vsftpd.conf

**Objetivo:** configurar vsftpd con usuarios locales enjaulados, TLS obligatorio y modo pasivo.

**Comando:** 

```bash title="Terminal"
sudo cp /etc/vsftpd.conf /etc/vsftpd.conf.original
sudo nano /etc/vsftpd.conf
```

**Archivo:** `/etc/vsftpd.conf` (sustituye todo el contenido):

```text title="Archivo de configuración"
# =====================================================================
#  /etc/vsftpd.conf - FTPS para ejemplo1.com y ejemplo2.com
# =====================================================================

# --- Escucha ------------------------------------------------------------
# Un único proceso que escucha en IPv6 y también en IPv4
listen=NO
listen_ipv6=YES

# --- Usuarios -----------------------------------------------------------
# Sin acceso anónimo
anonymous_enable=NO
# Usuarios locales de Linux (web1, web2) con permiso de escritura
local_enable=YES
write_enable=YES
# Archivos nuevos 640 y carpetas nuevas 750
local_umask=027
# Solo pueden entrar los usuarios de /etc/vsftpd.userlist
userlist_enable=YES
userlist_deny=NO
userlist_file=/etc/vsftpd.userlist
pam_service_name=vsftpd

# --- Jaula chroot -------------------------------------------------------
# Cada usuario queda encerrado en su carpeta personal (/var/www/ejemploN.com)
chroot_local_user=YES
# La raíz de la jaula NO debe ser escribible (ver PASO 15.5)
allow_writeable_chroot=NO
secure_chroot_dir=/var/run/vsftpd/empty
# Los listados muestran "ftp" en lugar de los nombres reales de usuario y grupo
hide_ids=YES

# --- Cifrado TLS obligatorio (FTPS explícito) ---------------------------
ssl_enable=YES
rsa_cert_file=/etc/letsencrypt/live/ftp.ejemplo1.com/fullchain.pem
rsa_private_key_file=/etc/letsencrypt/live/ftp.ejemplo1.com/privkey.pem
allow_anon_ssl=NO
force_local_logins_ssl=YES
force_local_data_ssl=YES
ssl_sslv2=NO
ssl_sslv3=NO
ssl_ciphers=HIGH
# Algunos clientes (FileZilla) fallan si se exige reutilizar la sesión TLS
require_ssl_reuse=NO

# --- Modo pasivo ----------------------------------------------------------
pasv_enable=YES
pasv_min_port=40000
pasv_max_port=40100
# Si el servidor está detrás de NAT (IP privada en la nube), descomenta y
# pon la IP pública:
#pasv_address=203.0.113.10

# --- Modo activo ----------------------------------------------------------
connect_from_port_20=YES

# --- Registro --------------------------------------------------------------
xferlog_enable=YES
xferlog_std_format=NO
vsftpd_log_file=/var/log/vsftpd.log
log_ftp_protocol=NO
use_localtime=YES

# --- Límites y cortesía ------------------------------------------------------
ftpd_banner=Servidor FTP. Solo se admiten conexiones cifradas.
idle_session_timeout=600
data_connection_timeout=120
max_clients=50
max_per_ip=5
dirmessage_enable=NO
```

Análisis de las directivas principales:

| Directiva | Significado |
|---|---|
| `listen=NO` + `listen_ipv6=YES` | Un solo socket IPv6 que acepta también IPv4. Si activaras las dos, vsftpd fallaría con `could not bind listening IPv4 socket`. |
| `anonymous_enable=NO` | Nadie puede entrar como `anonymous`. |
| `local_enable` / `write_enable` | Los usuarios locales pueden entrar y subir, modificar y borrar archivos. |
| `local_umask=027` | Permisos de lo que se sube: archivos `640` y carpetas `750`, iguales a los del Módulo 3. Gracias al bit setgid de `public_html`, el grupo será `www-data`. |
| `userlist_enable=YES` + `userlist_deny=NO` | La lista `/etc/vsftpd.userlist` es una lista de **permitidos**. |
| `chroot_local_user=YES` | Cada usuario ve su carpeta personal como `/` y no puede salir de ella. |
| `allow_writeable_chroot=NO` | Mantiene la comprobación de seguridad del PASO 15.5. |
| `hide_ids=YES` | Los listados no revelan nombres de usuarios del sistema. |
| `ssl_enable=YES` | Activa TLS. |
| `force_local_logins_ssl` / `force_local_data_ssl` | Exige cifrado para la contraseña **y** para los datos. Un cliente sin TLS recibe `530 Non-anonymous sessions must use encryption.` |
| `ssl_sslv2=NO`, `ssl_sslv3=NO` | Desactiva los protocolos obsoletos. |
| `ssl_ciphers=HIGH` | Solo cifrados robustos. |
| `require_ssl_reuse=NO` | Por defecto vsftpd exige que la conexión de datos reutilice la sesión TLS de la de control; es más seguro, pero muchos clientes fallan con `425 Unable to build data connection`. |
| `pasv_min_port` / `pasv_max_port` | Rango de puertos de datos en modo pasivo (101 puertos: suficiente para 50 transferencias simultáneas). |
| `pasv_address` | Solo si el servidor no tiene la IP pública en su interfaz (NAT de algunos proveedores de nube). |
| `xferlog_std_format=NO` | Registro en el formato propio de vsftpd en `/var/log/vsftpd.log`, que es el que entiende Fail2ban. |
| `max_clients` / `max_per_ip` | Máximo de conexiones totales y por IP. |

Comprueba y aplica:

```bash title="Terminal"
sudo systemctl restart vsftpd
systemctl status vsftpd --no-pager
sudo ss -tlnp | grep vsftpd
```

**Resultado esperado:** 

```text title="Salida esperada"
● vsftpd.service - vsftpd FTP server
     Active: active (running) since ...
LISTEN 0  32  *:21  *:*  users:(("vsftpd",pid=10240,fd=3))
```

`*:21` significa que escucha en IPv4 e IPv6.

**Errores habituales:** 

| Mensaje | Causa | Solución |
|---|---|---|
| `500 OOPS: could not bind listening IPv4 socket` | `listen=YES` y `listen_ipv6=YES` a la vez. | Deja solo `listen_ipv6=YES`. |
| `500 OOPS: SSL: cannot load RSA certificate` | Ruta del certificado incorrecta. | `sudo ls -l /etc/letsencrypt/live/ftp.ejemplo1.com/`. |
| `500 OOPS: unrecognised variable in config file: ...` | Errata en un nombre de directiva. | Corrige la línea indicada. |
| El servicio no arranca sin mensaje claro | Error de sintaxis (vsftpd no admite espacios alrededor del `=`). | `sudo journalctl -u vsftpd -n 20 --no-pager`; escribe `opcion=valor` sin espacios. |

### PASO 15.9 – Abrir los puertos en el cortafuegos

**Comando:** 

```bash title="Terminal"
sudo ufw allow 21/tcp comment 'FTP control (FTPS explicito)'
sudo ufw allow 40000:40100/tcp comment 'FTP pasivo'
sudo ufw status numbered
```

Análisis: `40000:40100/tcp` abre el rango completo de puertos pasivos. Si tu proveedor tiene un cortafuegos externo, ábrelos también allí.

!!! warning "Atención"
    No abras el puerto 20: en modo pasivo no se usa, y en modo activo es el **servidor** quien sale desde él (UFW permite todo el tráfico saliente). Tampoco abras el 990 (FTPS implícito): no lo hemos configurado.

## 15.6 Pruebas de funcionamiento

### PASO 15.10 – Probar el acceso con curl y lftp

**Objetivo:** verificar el cifrado, la autenticación, la jaula y la subida de archivos. Haz estas pruebas desde **otro equipo** (tu ordenador con Linux, macOS o WSL en Windows).

Comprobar el certificado del puerto 21 con STARTTLS:

```bash title="Terminal"
echo | openssl s_client -connect ftp.ejemplo1.com:21 -starttls ftp 2>/dev/null | openssl x509 -noout -subject -ext subjectAltName
```

```text title="Salida esperada"
subject=CN = ftp.ejemplo1.com
X509v3 Subject Alternative Name:
    DNS:ftp.ejemplo1.com, DNS:ftp.ejemplo2.com
```

Listar la carpeta con conexión cifrada (curl pedirá la contraseña):

```bash title="Terminal"
curl --ssl-reqd -u web1 ftp://ftp.ejemplo1.com/
```

Análisis: `--ssl-reqd` exige FTPS explícito (`AUTH TLS` sobre el puerto 21) y aborta si el servidor no lo admite. Como `-u web1` no lleva contraseña, curl la pide sin que quede en el historial.

```text title="Salida esperada"
drwx------    2 ftp      ftp          4096 Nov 09 10:12 app
drwx------    2 ftp      ftp          4096 Nov 09 10:12 config
drwx------    2 ftp      ftp          4096 Nov 09 10:12 logs
drwxr-s---    2 ftp      ftp          4096 Nov 09 10:12 public_html
drwx------    2 ftp      ftp          4096 Nov 09 10:12 tmp
```

Subir un archivo a la web:

```bash title="Terminal"
echo '<h1>Subido por FTPS</h1>' > subida.html
curl --ssl-reqd -u web1 -T subida.html ftp://ftp.ejemplo1.com/public_html/
curl -s https://www.ejemplo1.com/subida.html
```

```text title="Salida esperada"
<h1>Subido por FTPS</h1>
```

Comprobaciones de seguridad (todas deben **fallar**):

```bash title="Terminal"
# 1. Sin cifrado
curl -u web1 ftp://ftp.ejemplo1.com/
# 2. Usuario no autorizado
curl --ssl-reqd -u sysadmin ftp://ftp.ejemplo1.com/
# 3. Salir de la jaula
curl --ssl-reqd -u web1 ftp://ftp.ejemplo1.com/../../etc/
```

```text title="Salida esperada"
curl: (67) Access denied: 530           ← 530 Non-anonymous sessions must use encryption.
curl: (67) Access denied: 530           ← 530 Permission denied. (sin llegar a pedir la contraseña)
                                          ← muestra de nuevo /var/www/ejemplo1.com: "/" es la raíz de la jaula
```

Con **lftp**, un cliente FTP de terminal muy completo, puedes trabajar de forma interactiva:

```bash title="Terminal"
lftp -u web1 -e "set ftp:ssl-force true; set ssl:verify-certificate yes" ftp.ejemplo1.com
```

Dentro de lftp: `ls`, `cd public_html`, `put archivo`, `get archivo`, `mirror -R carpeta_local carpeta_remota` (sube una carpeta entera) y `bye` para salir.

Comprobación en el servidor: propietario y permisos de lo subido, y registro:

```bash title="Terminal"
sudo ls -l /var/www/ejemplo1.com/public_html/subida.html
sudo tail -n 5 /var/log/vsftpd.log
```

```text title="Salida esperada"
-rw-r----- 1 web1 www-data 25 nov  9 10:20 /var/www/ejemplo1.com/public_html/subida.html
Mon Nov  9 10:20:01 2026 [pid 10301] [web1] OK LOGIN: Client "::ffff:198.51.100.7"
Mon Nov  9 10:20:02 2026 [pid 10303] [web1] OK UPLOAD: Client "::ffff:198.51.100.7", "/public_html/subida.html", 25 bytes, 0.41Kbyte/sec
```

El archivo pertenece a `web1`, su grupo es `www-data` y sus permisos `640`: exactamente lo que necesita Apache para servirlo. Bórralo al terminar:

```bash title="Terminal"
sudo rm /var/www/ejemplo1.com/public_html/subida.html
```

**Errores habituales:** 

| Síntoma | Causa | Solución |
|---|---|---|
| `530 Login incorrect.` | Contraseña errónea, o PAM rechaza el shell. | `sudo passwd web1`; revisa el PASO 15.6. En `/var/log/auth.log` verás el motivo. |
| `500 OOPS: vsftpd: refusing to run with writable root inside chroot()` | La raíz de la web es escribible por el usuario. | PASO 15.5. |
| El login funciona pero el listado se queda colgado o da `425` | Puertos pasivos cerrados, o `pasv_address` necesario por NAT. | UFW y cortafuegos del proveedor (40000–40100); configura `pasv_address`. |
| `425 Unable to build data connection: ... SSL` | El cliente no reutiliza la sesión TLS. | `require_ssl_reuse=NO`. |
| `553 Could not create file.` | El usuario no puede escribir en esa carpeta (por ejemplo, en la raíz de la jaula). | Sube a `public_html/`. |
| Error de certificado en el cliente | Conectaste con la IP o con `mail.*` en lugar de `ftp.*`. | Usa `ftp.ejemplo1.com` o `ftp.ejemplo2.com`. |

### PASO 15.11 – Configurar FileZilla

**Objetivo:** dar acceso a la persona responsable de cada web con un cliente gráfico.

En FileZilla: «Archivo» → «Gestor de sitios» → «Nuevo sitio» y rellena:

| Campo | Valor para ejemplo1.com | Valor para ejemplo2.com |
|---|---|---|
| Protocolo | FTP – Protocolo de transferencia de archivos | Igual |
| Servidor | `ftp.ejemplo1.com` | `ftp.ejemplo2.com` |
| Puerto | 21 (o vacío) | 21 (o vacío) |
| Cifrado | **Requiere FTP explícito sobre TLS** | Igual |
| Modo de acceso | Normal | Normal |
| Usuario | `web1` | `web2` |
| Contraseña | La del PASO 15.4 | La del PASO 15.4 |

En la pestaña «Opciones de transferencia», el modo debe ser **Pasivo** (es el predeterminado).

**Comprobación:** al conectar, FileZilla muestra el certificado; comprueba que está emitido para `ftp.ejemplo1.com`, acéptalo y verás las carpetas de la web. Arrastra un archivo a `public_html` y ábrelo en el navegador.

!!! warning "Atención"
    Explica a los usuarios que **nunca** elijan «Usar FTP plano (inseguro)». Con nuestra configuración el servidor lo rechazaría, pero el cliente habría enviado ya el nombre de usuario sin cifrar.

## 15.7 Integración con la infraestructura

### PASO 15.12 – Recargar vsftpd cuando se renueve su certificado

**Objetivo:** incluir vsftpd en el script de renovación del Módulo 10.

**Archivo:** `/etc/letsencrypt/renewal-hooks/deploy/recargar-servicios.sh` (sustituye todo el contenido):

```bash title="Terminal"
#!/bin/bash
# Se ejecuta tras cada renovación correcta de un certificado.
# Certbot define RENEWED_LINEAGE, por ejemplo /etc/letsencrypt/live/mail.ejemplo1.com
set -u

logger -t certbot-deploy "Certificado renovado: ${RENEWED_LINEAGE}"

case "${RENEWED_LINEAGE}" in
    */mail.ejemplo1.com)
        # Certificado del correo: Postfix y Dovecot
        systemctl reload postfix
        systemctl reload dovecot
        logger -t certbot-deploy "Postfix y Dovecot recargados"
        ;;
    */ftp.ejemplo1.com)
        # Certificado del FTP: vsftpd lo carga al arrancar
        systemctl restart vsftpd
        logger -t certbot-deploy "vsftpd reiniciado"
        ;;
    *)
        # Certificados de las webs: Apache
        if apachectl configtest >/dev/null 2>&1; then
            systemctl reload apache2
            logger -t certbot-deploy "Apache recargado"
        else
            logger -t certbot-deploy "ERROR: configuración de Apache inválida, no se recarga"
        fi
        ;;
esac
```

Análisis: el nuevo caso reinicia vsftpd cuando se renueva el certificado `ftp.ejemplo1.com`. El reinicio corta las transferencias en curso, pero Certbot renueva de madrugada y solo una vez cada dos meses.

Prueba el script:

```bash title="Terminal"
sudo RENEWED_LINEAGE=/etc/letsencrypt/live/ftp.ejemplo1.com /etc/letsencrypt/renewal-hooks/deploy/recargar-servicios.sh
sudo journalctl -t certbot-deploy -n 3 --no-pager
```

### PASO 15.13 – Proteger el FTP con Fail2ban

**Objetivo:** bloquear las IP que prueban contraseñas en el FTP.

**Archivo:** `/etc/fail2ban/jail.local`. Conserva todo lo del PASO 12.5 y **añade al final**:

```ini title="Archivo de configuración"
# --- vsftpd: contraseñas FTP incorrectas ---
[vsftpd]
enabled  = true
logpath  = /var/log/vsftpd.log
backend  = auto
maxretry = 4
```

**Comando:** 

```bash title="Terminal"
sudo fail2ban-client -t && sudo systemctl restart fail2ban
sudo fail2ban-client status vsftpd
```

Análisis: el filtro `vsftpd` incluido en Fail2ban busca las líneas `FAIL LOGIN` de `/var/log/vsftpd.log`. Al bloquear una IP, se cierran a la vez el puerto 21 y el rango pasivo, porque la acción bloquea todo el tráfico de esa IP hacia los puertos indicados en el jail (que por defecto incluye `ftp`, `ftp-data` y `ftps`) y, sin el 21, el resto queda inservible.

**Comprobación:** falla la contraseña cuatro veces desde una IP de pruebas y comprueba que aparece en `Banned IP list`. Desbloquéala con `sudo fail2ban-client set vsftpd unbanip IP`.

### PASO 15.14 – Añadir vsftpd a la monitorización

**Objetivo:** que el script de estado del Módulo 12 vigile también el FTP.

**Archivo:** `/usr/local/sbin/estado-servidor`. Modifica solo estas dos líneas del principio (el resto se conserva):

```bash title="Terminal"
SERVICIOS="apache2 php8.3-fpm mariadb postfix@- dovecot opendkim fail2ban ssh cron vsftpd"
PUERTOS="21 22 25 80 443 465 587 993"
```

**Comprobación:** `sudo estado-servidor | grep -E 'vsftpd|puerto 21'` debe mostrar dos líneas `[OK]`.

!!! tip "Consejo"
    Las copias de seguridad no necesitan cambios: el script del PASO 12.7 ya incluye `/var/www` (los archivos subidos) y `/etc` (la configuración de vsftpd y PAM).

## 15.8 Gestión del día a día

| Tarea | Cómo |
|---|---|
| Cambiar la contraseña FTP de una web | `sudo passwd web1` |
| Quitar temporalmente el FTP a una web | Borra su línea de `/etc/vsftpd.userlist` (efecto inmediato en nuevas conexiones). |
| Bloquear la cuenta sin tocar la lista | `sudo passwd -l web1` (y `sudo passwd -u web1` para desbloquearla). |
| Dar FTP a una nueva web (`web3`) | Crea la web (Anexo D), haz los PASOS 15.4 y 15.5 para `web3`, añádela a `vsftpd.userlist` y amplía el certificado con `ftp.ejemplo3.com`. |
| Ver quién está conectado | `ps -eo user,cmd \| grep '[v]sftpd'` |
| Ver las subidas de hoy | `sudo grep "$(date '+%a %b %e')" /var/log/vsftpd.log \| grep UPLOAD` |
| Ver los fallos de acceso | `sudo grep 'FAIL LOGIN' /var/log/vsftpd.log \| tail` |
| Desactivar el FTP por completo | `sudo systemctl disable --now vsftpd` y cierra los puertos con `sudo ufw delete allow 21/tcp` y `sudo ufw delete allow 40000:40100/tcp`. |

## 15.9 Diagrama de la integración

```text title="Diagrama"
 FileZilla / lftp (persona responsable de ejemplo1.com)
     │ 1. TCP 21 · AUTH TLS · certificado ftp.ejemplo1.com
     │ 2. USER web1 → ¿está en /etc/vsftpd.userlist? → PASS (cifrada)
     ▼
 ┌──────────────────────────────── SERVIDOR ──────────────────────────────────┐
 │  UFW: 21 y 40000-40100 permitidos · Fail2ban vigila /var/log/vsftpd.log    │
 │                                                                            │
 │  VSFTPD ── PAM (/etc/pam.d/vsftpd): /etc/ftpusers + contraseña de web1     │
 │     │ 3. chroot en /var/www/ejemplo1.com (raíz root:www-data + ACL web1)   │
 │     │ 4. PASV → conexión de datos cifrada por un puerto 40000-40100        │
 │     ▼                                                                      │
 │  /var/www/ejemplo1.com/public_html/nuevo.php  (web1:www-data, 640)         │
 │     │                                                                      │
 │     ├──► APACHE lo sirve (grupo www-data)                                  │
 │     └──► PHP-FPM pool web1 lo ejecuta (usuario web1)                       │
 └────────────────────────────────────────────────────────────────────────────┘
```

## 15.10 Alternativa: SFTP con OpenSSH

Si prefieres no abrir el puerto 21 ni el rango pasivo, OpenSSH puede ofrecer SFTP con la misma separación por webs. Esquema de la configuración (sustituye a vsftpd; no uses ambos sin necesidad):

**Archivo:** `/etc/ssh/sshd_config.d/10-sftp-webs.conf` (nuevo):

```text title="Archivo de configuración"
# Los usuarios del grupo sftpwebs solo pueden usar SFTP, encerrados en su web
Match Group sftpwebs
    ChrootDirectory %h
    ForceCommand internal-sftp -u 0027
    PasswordAuthentication yes
    AllowTcpForwarding no
    X11Forwarding no
```

**Comandos:** 

```bash title="Terminal"
sudo groupadd sftpwebs
sudo usermod -aG sftpwebs web1
sudo usermod -aG sftpwebs web2
sudo sed -i 's/^AllowUsers .*/AllowUsers sysadmin web1 web2/' /etc/ssh/sshd_config.d/00-endurecimiento.conf
sudo sshd -t && sudo systemctl restart ssh
```

Análisis:

| Elemento | Significado |
|---|---|
| `Match Group sftpwebs` | Lo que sigue solo se aplica a los miembros de ese grupo. |
| `ChrootDirectory %h` | Encierra al usuario en su carpeta personal. OpenSSH exige que esa carpeta **y todas las superiores** pertenezcan a root y no sean escribibles por nadie más: el PASO 15.5 ya lo cumple. |
| `ForceCommand internal-sftp -u 0027` | Solo SFTP (nunca un terminal), con archivos nuevos `640`. |
| `PasswordAuthentication yes` | Permite contraseña solo a estos usuarios (lo ideal es usar también claves). |
| `AllowUsers` | Hay que añadir `web1` y `web2`, porque en el PASO 1.7 limitamos SSH a `sysadmin`. |

Los clientes se conectan con protocolo **SFTP**, puerto 22 y servidor `ftp.ejemplo1.com` (o cualquier nombre del servidor). Fail2ban ya protege el puerto 22 con el jail `sshd`.

## 15.11 Ejercicios prácticos del Módulo 15

1. Conecta con FileZilla como `web2` y comprueba que no puedes ver ni entrar en `/var/www/ejemplo1.com`. ¿Qué ves como carpeta raíz?
2. Sube con `lftp mirror -R` una carpeta con varios archivos HTML a `public_html` de ejemplo2.com y verifica sus permisos en el servidor.
3. Cambia temporalmente `allow_writeable_chroot` y la propiedad de `/var/www/ejemplo1.com` para provocar el error `refusing to run with writable root inside chroot()`. Explica por qué existe esa protección y restaura la configuración.
4. Comenta el rango pasivo en UFW y observa qué hace FileZilla al listar una carpeta. Restaura la regla.
5. Configura la alternativa SFTP del apartado 15.10 en una máquina de pruebas y compara ambos métodos: puertos abiertos, configuración necesaria y experiencia de uso.
