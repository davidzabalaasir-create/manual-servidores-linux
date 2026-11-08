# Módulo 5. Integración de MariaDB con Apache y PHP

## Objetivos del módulo

- Entender cómo colaboran Apache, PHP-FPM y MariaDB para servir una página dinámica.
- Instalar PHP 8.3 con PHP-FPM y los módulos necesarios.
- Crear un *pool* de PHP-FPM independiente para cada web, con su propio usuario Linux.
- Conectar Apache con cada pool desde su VirtualHost.
- Programar una aplicación PHP que se conecta a su propia base de datos con PDO y consultas preparadas.
- Implementar las operaciones de inserción, lectura, actualización y eliminación (CRUD), con gestión de errores y protección de credenciales.

## 5.1 Cómo funciona una página dinámica

Una página estática (HTML) es un archivo que Apache entrega tal cual. Una página dinámica (PHP) es un **programa** que se ejecuta en cada visita y genera el HTML en ese momento, normalmente consultando una base de datos.

Hay dos formas de que Apache ejecute PHP:

| Método | Cómo funciona | Ventajas / inconvenientes |
|---|---|---|
| `mod_php` (libapache2-mod-php) | PHP se carga **dentro** de cada proceso de Apache. | Sencillo, pero obliga a usar el MPM `prefork` (más memoria), y todas las webs se ejecutan con el mismo usuario `www-data`. |
| **PHP-FPM** (FastCGI Process Manager) | PHP es un servicio aparte. Apache le pasa las peticiones `.php` por un socket. | Compatible con `mpm_event`, cada web puede tener su propio *pool* de procesos con su propio usuario y su propia configuración. **Es el método profesional y el que usaremos.** |

El recorrido completo de una petición:

```text title="Diagrama"
 Navegador: GET https://www.ejemplo1.com/index.php
      │
      ▼
 ┌─────────────────────────┐
 │ APACHE (www-data)       │ 1. TLS, elige el VirtualHost www.ejemplo1.com
 │ VirtualHost ejemplo1    │ 2. El archivo termina en .php → SetHandler proxy:unix:...web1.sock
 └───────────┬─────────────┘
             │ protocolo FastCGI por socket Unix /run/php/php8.3-fpm-web1.sock
             ▼
 ┌─────────────────────────┐
 │ PHP-FPM pool "web1"     │ 3. Un proceso PHP (usuario web1) ejecuta index.php
 │ usuario Linux: web1     │ 4. index.php lee ../config/config.php (fuera de public_html)
 └───────────┬─────────────┘
             │ PDO / protocolo MySQL por socket /run/mysqld/mysqld.sock
             ▼
 ┌─────────────────────────┐
 │ MARIADB                 │ 5. Autentica a web1_user, ejecuta la consulta en web1_db
 └───────────┬─────────────┘
             │ filas de resultado
             ▼
   PHP genera el HTML  →  Apache lo envía al navegador (6)
```

## 5.2 Instalación de PHP y PHP-FPM

### PASO 5.1 – Instalar PHP-FPM y los módulos necesarios

**Objetivo:** instalar el intérprete PHP como servicio FPM y las extensiones que usan las aplicaciones habituales.

**Comando:** 

```bash title="Terminal"
sudo apt install -y php-fpm php-mysql php-mbstring php-xml php-curl php-intl php-zip php-gd
```

**Análisis del comando:** en Ubuntu 24.04, los paquetes `php-*` son «metapaquetes» que instalan la versión del sistema (`php8.3-*`):

| Paquete | Para qué sirve |
|---|---|
| `php-fpm` | El servicio PHP-FPM (`php8.3-fpm`) y el intérprete de línea de comandos. |
| `php-mysql` | Las extensiones `pdo_mysql` y `mysqli` para hablar con MariaDB/MySQL. |
| `php-mbstring` | Manejo de textos multibyte (tildes, eñes, UTF-8). |
| `php-xml` | Procesado de XML (lo usan muchísimas aplicaciones). |
| `php-curl` | Peticiones HTTP desde PHP (APIs, pasarelas de pago). |
| `php-intl` | Internacionalización: formatos de fecha, moneda, ordenación por idioma. |
| `php-zip` y `php-gd` | Archivos comprimidos e imágenes (miniaturas, captchas). |

!!! warning "Atención"
    **No** instales el paquete `php` a secas ni `libapache2-mod-php`: arrastran `mod_php` y cambian Apache al MPM `prefork`, deshaciendo el diseño de este módulo. Si ves que se instala `libapache2-mod-php8.3`, cancela.

**Resultado esperado:** al final aparece un aviso como este:

```text title="Salida esperada"
NOTICE: Not enabling PHP 8.3 FPM by default.
NOTICE: To enable PHP 8.3 FPM in Apache2 do:
NOTICE: a2enmod proxy_fcgi setenvif
NOTICE: a2enconf php8.3-fpm
```

Es normal: haremos la integración nosotros, sitio por sitio.

**Comprobación:** 

```bash title="Terminal"
php -v
systemctl status php8.3-fpm --no-pager
php -m | grep -Ei 'pdo_mysql|mysqli|mbstring|intl'
sudo apachectl -M | grep -i php || echo "Correcto: mod_php no está cargado"
```

```text title="Salida esperada"
PHP 8.3.6 (cli) (built: ...) (NTS)
● php8.3-fpm.service - The PHP 8.3 FastCGI Process Manager
     Active: active (running) since ...
intl
mbstring
mysqli
pdo_mysql
Correcto: mod_php no está cargado
```

**Errores habituales:** si `php -v` muestra otra versión (por ejemplo 8.2 en Debian 12), usa esa versión en **todas** las rutas de este módulo (`/etc/php/8.2/...`, `php8.2-fpm`).

**Siguiente paso:** conocer la estructura de PHP-FPM.

## 5.3 Estructura de configuración de PHP-FPM

```text title="Diagrama"
/etc/php/8.3/
├── cli/php.ini              ← configuración de PHP en la línea de comandos
├── fpm/
│   ├── php.ini              ← configuración global de PHP para las webs
│   ├── php-fpm.conf         ← configuración del gestor de procesos
│   └── pool.d/
│       ├── www.conf         ← pool por defecto (usuario www-data) – lo desactivaremos
│       ├── web1.conf        ← pool de ejemplo1.com (usuario web1) – lo crearemos
│       └── web2.conf        ← pool de ejemplo2.com (usuario web2) – lo crearemos
└── mods-available/          ← extensiones (.ini)
/run/php/php8.3-fpm-web1.sock   ← socket del pool web1 (se crea al arrancar)
/var/log/php8.3-fpm.log         ← log del gestor FPM
```

Un **pool** es un grupo de procesos PHP con su propia configuración: con qué usuario se ejecutan, en qué socket escuchan, cuántos procesos pueden lanzar y qué opciones de PHP aplican. Con un pool por web conseguimos:

- **Aislamiento**: el código de ejemplo1.com se ejecuta como `web1` y no puede leer los archivos de `web2`.
- **Límites independientes**: si una web recibe muchísimas visitas, agota sus procesos, no los de la otra.
- **Configuración propia**: límites de memoria, rutas de sesión o remitente del correo distintos por web.

### PASO 5.2 – Crear la carpeta de logs de cada web

**Objetivo:** preparar un lugar donde PHP escriba los errores de cada aplicación.

**Comando:** 

```bash title="Terminal"
sudo install -d -o web1 -g web1 -m 700 /var/www/ejemplo1.com/logs
sudo install -d -o web2 -g web2 -m 700 /var/www/ejemplo2.com/logs
```

**Análisis del comando:** `install -d` crea una carpeta y, en la misma orden, fija propietario (`-o`), grupo (`-g`) y permisos (`-m`).

**Siguiente paso:** crear los pools.

### PASO 5.3 – Crear el pool de PHP-FPM de ejemplo1.com

**Objetivo:** definir el pool `web1`.

**Comando:** 

```bash title="Terminal"
sudo nano /etc/php/8.3/fpm/pool.d/web1.conf
```

**Archivo:** `/etc/php/8.3/fpm/pool.d/web1.conf` (nuevo, contenido completo):

```ini title="Archivo de configuración"
; Pool de PHP-FPM para ejemplo1.com
[web1]
; Usuario y grupo con los que se ejecuta el código PHP de esta web
user = web1
group = web1

; Socket por el que Apache enviará las peticiones
listen = /run/php/php8.3-fpm-web1.sock
; Apache (www-data) debe poder leer y escribir en el socket
listen.owner = www-data
listen.group = www-data
listen.mode = 0660

; Gestión de procesos: se crean bajo demanda y se cierran si están inactivos
pm = ondemand
pm.max_children = 10
pm.process_idle_timeout = 30s
pm.max_requests = 500

; Directorio de trabajo de los procesos
chdir = /

; --- Opciones de PHP fijadas para esta web (no modificables desde el código) ---
; PHP solo puede abrir archivos dentro de la carpeta de la web
php_admin_value[open_basedir] = /var/www/ejemplo1.com/:/usr/share/php/
; Temporales, subidas y sesiones dentro de la carpeta privada tmp/
php_admin_value[sys_temp_dir] = /var/www/ejemplo1.com/tmp
php_admin_value[upload_tmp_dir] = /var/www/ejemplo1.com/tmp
php_admin_value[session.save_path] = /var/www/ejemplo1.com/tmp
php_admin_value[session.gc_probability] = 1
; Errores: nunca se muestran al visitante, se guardan en el log de la web
php_admin_flag[display_errors] = off
php_admin_flag[log_errors] = on
php_admin_value[error_log] = /var/www/ejemplo1.com/logs/php-error.log
; Límites de recursos
php_admin_value[memory_limit] = 128M
php_admin_value[upload_max_filesize] = 16M
php_admin_value[post_max_size] = 16M
php_admin_value[max_execution_time] = 30
; Funciones peligrosas deshabilitadas
php_admin_value[disable_functions] = exec,passthru,shell_exec,system,proc_open,popen,pcntl_exec
; Remitente del correo enviado con mail() (se usa en el Módulo 11)
php_admin_value[sendmail_path] = /usr/sbin/sendmail -t -i -f noreply@ejemplo1.com
; Cookies de sesión seguras
php_admin_flag[session.cookie_httponly] = on
php_admin_flag[session.cookie_secure] = on
php_admin_flag[session.use_strict_mode] = on
```

Análisis de las directivas:

| Directiva | Significado |
|---|---|
| `[web1]` | Nombre del pool (aparece en los procesos y en los logs). |
| `user` / `group` | Identidad con la que se ejecuta el código PHP. |
| `listen` | Ruta del socket Unix. |
| `listen.owner/group/mode` | Propietario y permisos del socket: solo Apache (www-data) puede conectarse. |
| `pm = ondemand` | Los procesos se crean al llegar visitas y se cierran tras `process_idle_timeout`. Ideal para servidores pequeños con varias webs. Alternativas: `dynamic` (mantiene algunos procesos siempre listos) y `static`. |
| `pm.max_children` | Máximo de procesos simultáneos. Cada uno puede gastar hasta `memory_limit`; 10 × 128 MB es el peor caso. |
| `pm.max_requests` | Cada proceso se recicla tras 500 peticiones (evita fugas de memoria). |
| `php_admin_value[...]` | Fija una opción de PHP que el código de la web **no** puede cambiar con `ini_set()`. |
| `open_basedir` | Jaula: PHP no puede abrir archivos fuera de esas rutas. |
| `disable_functions` | Impide ejecutar comandos del sistema desde PHP, la vía favorita de los atacantes. |
| `session.cookie_secure` | La cookie de sesión solo viaja por HTTPS. |

**Siguiente paso:** el pool de ejemplo2.com.

### PASO 5.4 – Crear el pool de PHP-FPM de ejemplo2.com

**Archivo:** `/etc/php/8.3/fpm/pool.d/web2.conf` (nuevo, contenido completo):

```ini title="Archivo de configuración"
; Pool de PHP-FPM para ejemplo2.com
[web2]
user = web2
group = web2

listen = /run/php/php8.3-fpm-web2.sock
listen.owner = www-data
listen.group = www-data
listen.mode = 0660

pm = ondemand
pm.max_children = 10
pm.process_idle_timeout = 30s
pm.max_requests = 500

chdir = /

php_admin_value[open_basedir] = /var/www/ejemplo2.com/:/usr/share/php/
php_admin_value[sys_temp_dir] = /var/www/ejemplo2.com/tmp
php_admin_value[upload_tmp_dir] = /var/www/ejemplo2.com/tmp
php_admin_value[session.save_path] = /var/www/ejemplo2.com/tmp
php_admin_value[session.gc_probability] = 1
php_admin_flag[display_errors] = off
php_admin_flag[log_errors] = on
php_admin_value[error_log] = /var/www/ejemplo2.com/logs/php-error.log
php_admin_value[memory_limit] = 128M
php_admin_value[upload_max_filesize] = 16M
php_admin_value[post_max_size] = 16M
php_admin_value[max_execution_time] = 30
php_admin_value[disable_functions] = exec,passthru,shell_exec,system,proc_open,popen,pcntl_exec
php_admin_value[sendmail_path] = /usr/sbin/sendmail -t -i -f noreply@ejemplo2.com
php_admin_flag[session.cookie_httponly] = on
php_admin_flag[session.cookie_secure] = on
php_admin_flag[session.use_strict_mode] = on
```

### PASO 5.5 – Desactivar el pool por defecto y aplicar la configuración

**Objetivo:** dejar solo los pools que usamos y comprobar la configuración.

**Explicación:** el pool `www` ejecuta PHP como `www-data` y no lo necesitamos. Lo desactivamos renombrándolo (PHP-FPM solo carga los archivos que terminan en `.conf`), así podemos recuperarlo si hiciera falta.

**Comando:** 

```bash title="Terminal"
sudo mv /etc/php/8.3/fpm/pool.d/www.conf /etc/php/8.3/fpm/pool.d/www.conf.desactivado
sudo php-fpm8.3 -t
sudo systemctl restart php8.3-fpm
ls -l /run/php/
```

**Análisis del comando:** `php-fpm8.3 -t` comprueba la sintaxis de toda la configuración de FPM (equivale a `apachectl configtest`).

**Resultado esperado:** 

```text title="Salida esperada"
NOTICE: configuration file /etc/php/8.3/fpm/php-fpm.conf test is successful
srw-rw---- 1 www-data www-data 0 nov  8 23:40 php8.3-fpm-web1.sock
srw-rw---- 1 www-data www-data 0 nov  8 23:40 php8.3-fpm-web2.sock
-rw-r--r-- 1 root     root     5 nov  8 23:40 php8.3-fpm.pid
```

La `s` inicial indica que son sockets.

**Errores habituales:** 

- `ERROR: [pool web1] cannot get uid for user 'web1'`: el usuario no existe (vuelve al PASO 3.1).
- `ERROR: No pool defined`: se desactivaron todos los pools; revisa que `web1.conf` y `web2.conf` terminan en `.conf`.
- Consulta los detalles con `sudo journalctl -u php8.3-fpm -n 30 --no-pager` y en `/var/log/php8.3-fpm.log`.

**Siguiente paso:** conectar Apache con los pools.

## 5.4 Conexión de Apache con PHP-FPM

### PASO 5.6 – Activar los módulos de proxy FastCGI

**Comando:** 

```bash title="Terminal"
sudo a2enmod proxy_fcgi setenvif
sudo systemctl restart apache2
```

**Análisis del comando:** `proxy_fcgi` (que activa también `proxy`) permite a Apache hablar el protocolo FastCGI con PHP-FPM; `setenvif` lo requiere para pasar correctamente ciertas cabeceras (como la de autorización).

!!! tip "Consejo"
    No ejecutamos `a2enconf php8.3-fpm`: esa configuración activaría PHP para **todas** las webs a través de un pool común. Nosotros asignaremos un pool a cada VirtualHost.

### PASO 5.7 – VirtualHost definitivo de ejemplo1.com con PHP

**Objetivo:** indicar en el VirtualHost que los archivos `.php` se envíen al pool `web1`.

**Archivo:** `/etc/apache2/sites-available/ejemplo1.com.conf` (sustituye todo el contenido; es la versión definitiva):

```text title="Archivo de configuración"
# ---------------------------------------------------------------
# ejemplo1.com - HTTP: solo redirige a HTTPS
# ---------------------------------------------------------------
<VirtualHost *:80>
    ServerName www.ejemplo1.com
    ServerAlias ejemplo1.com
    ServerAdmin administracion@ejemplo1.com
    DocumentRoot /var/www/ejemplo1.com/public_html

    RewriteEngine On
    RewriteCond %{REQUEST_URI} !^/\.well-known/acme-challenge/
    RewriteRule ^ https://www.ejemplo1.com%{REQUEST_URI} [L,R=301]

    ErrorLog ${APACHE_LOG_DIR}/ejemplo1.com-error.log
    CustomLog ${APACHE_LOG_DIR}/ejemplo1.com-access.log combined
</VirtualHost>

# ---------------------------------------------------------------
# ejemplo1.com - HTTPS con PHP-FPM (pool web1)
# ---------------------------------------------------------------
<VirtualHost *:443>
    ServerName www.ejemplo1.com
    ServerAlias ejemplo1.com
    ServerAdmin administracion@ejemplo1.com
    DocumentRoot /var/www/ejemplo1.com/public_html

    SSLEngine on
    SSLCertificateFile    /etc/letsencrypt/live/ejemplo1.com/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/ejemplo1.com/privkey.pem
    Protocols h2 http/1.1

    RewriteEngine On
    RewriteCond %{HTTP_HOST} !^www\.ejemplo1\.com$ [NC]
    RewriteRule ^ https://www.ejemplo1.com%{REQUEST_URI} [L,R=301]

    Header always set Strict-Transport-Security "max-age=15552000"

    # Página inicial: primero index.php, después index.html
    DirectoryIndex index.php index.html

    # Los archivos .php se ejecutan en el pool web1 (usuario web1)
    <FilesMatch "\.php$">
        SetHandler "proxy:unix:/run/php/php8.3-fpm-web1.sock|fcgi://web1"
    </FilesMatch>

    <Directory /var/www/ejemplo1.com/public_html>
        Options -Indexes +FollowSymLinks
        AllowOverride None
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/ejemplo1.com-error.log
    CustomLog ${APACHE_LOG_DIR}/ejemplo1.com-access.log combined
</VirtualHost>
```

Análisis de las directivas nuevas:

| Directiva | Significado |
|---|---|
| `DirectoryIndex index.php index.html` | Si se pide una carpeta (`/`), Apache busca primero `index.php`. |
| `<FilesMatch "\.php$">` | Se aplica a los archivos cuyo nombre termina en `.php`. |
| `SetHandler "proxy:unix:RUTA\|fcgi://web1"` | Pasa la petición por FastCGI al socket indicado. `fcgi://web1` es un nombre interno; usar uno distinto por pool evita que Apache mezcle conexiones de pools diferentes. |

### PASO 5.8 – VirtualHost definitivo de ejemplo2.com con PHP

**Archivo:** `/etc/apache2/sites-available/ejemplo2.com.conf` (sustituye todo el contenido):

```text title="Archivo de configuración"
<VirtualHost *:80>
    ServerName www.ejemplo2.com
    ServerAlias ejemplo2.com
    ServerAdmin soporte@ejemplo2.com
    DocumentRoot /var/www/ejemplo2.com/public_html

    RewriteEngine On
    RewriteCond %{REQUEST_URI} !^/\.well-known/acme-challenge/
    RewriteRule ^ https://www.ejemplo2.com%{REQUEST_URI} [L,R=301]

    ErrorLog ${APACHE_LOG_DIR}/ejemplo2.com-error.log
    CustomLog ${APACHE_LOG_DIR}/ejemplo2.com-access.log combined
</VirtualHost>

<VirtualHost *:443>
    ServerName www.ejemplo2.com
    ServerAlias ejemplo2.com
    ServerAdmin soporte@ejemplo2.com
    DocumentRoot /var/www/ejemplo2.com/public_html

    SSLEngine on
    SSLCertificateFile    /etc/letsencrypt/live/ejemplo2.com/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/ejemplo2.com/privkey.pem
    Protocols h2 http/1.1

    RewriteEngine On
    RewriteCond %{HTTP_HOST} !^www\.ejemplo2\.com$ [NC]
    RewriteRule ^ https://www.ejemplo2.com%{REQUEST_URI} [L,R=301]

    Header always set Strict-Transport-Security "max-age=15552000"

    DirectoryIndex index.php index.html

    <FilesMatch "\.php$">
        SetHandler "proxy:unix:/run/php/php8.3-fpm-web2.sock|fcgi://web2"
    </FilesMatch>

    <Directory /var/www/ejemplo2.com/public_html>
        Options -Indexes +FollowSymLinks
        AllowOverride None
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/ejemplo2.com-error.log
    CustomLog ${APACHE_LOG_DIR}/ejemplo2.com-access.log combined
</VirtualHost>
```

Aplica los cambios:

```bash title="Terminal"
sudo apachectl configtest && sudo systemctl reload apache2
```

### PASO 5.9 – Comprobar que PHP se ejecuta con el usuario correcto

**Objetivo:** verificar que cada web ejecuta PHP en su propio pool.

**Comando:** 

```bash title="Terminal"
echo '<?php echo "PHP " . PHP_VERSION . " ejecutado por: " . get_current_user() . " / proceso: " . posix_getpwuid(posix_geteuid())["name"];' | sudo -u web1 tee /var/www/ejemplo1.com/public_html/quien.php > /dev/null
echo '<?php echo "PHP " . PHP_VERSION . " ejecutado por: " . get_current_user() . " / proceso: " . posix_getpwuid(posix_geteuid())["name"];' | sudo -u web2 tee /var/www/ejemplo2.com/public_html/quien.php > /dev/null
curl -s https://www.ejemplo1.com/quien.php; echo
curl -s https://www.ejemplo2.com/quien.php; echo
```

**Análisis del comando:** `posix_geteuid()` devuelve el usuario efectivo del proceso PHP; `posix_getpwuid()` lo traduce a nombre.

**Resultado esperado:** 

```text title="Salida esperada"
PHP 8.3.6 ejecutado por: web1 / proceso: web1
PHP 8.3.6 ejecutado por: web2 / proceso: web2
```

Comprobación adicional: borra los archivos de prueba (no dejes nunca scripts de diagnóstico publicados):

```bash title="Terminal"
sudo rm /var/www/ejemplo1.com/public_html/quien.php /var/www/ejemplo2.com/public_html/quien.php
```

**Errores habituales:** 

| Síntoma | Causa | Solución |
|---|---|---|
| El navegador descarga el archivo o muestra el código PHP. | El `<FilesMatch>` no se aplica (falta en el bloque 443, o falta `proxy_fcgi`). | Revisa el VirtualHost y `apachectl -M \| grep proxy_fcgi`. |
| `503 Service Unavailable`. | El socket no existe: PHP-FPM está parado o la ruta no coincide. | `ls /run/php/` y `systemctl status php8.3-fpm`. En el error.log verás `AH01079: failed to make connection to backend`. |
| `Permission denied: AH02454` en el log. | Apache no puede usar el socket. | Revisa `listen.owner/group/mode` del pool. |
| `File not found.` (texto plano). | PHP-FPM no encuentra o no puede leer el script. | Permisos: el usuario del pool debe poder leer el archivo. |
| `open_basedir restriction in effect`. | El código intenta abrir un archivo fuera de la jaula. | Es la protección funcionando; ajusta la ruta del código, no amplíes la jaula sin motivo. |

**Siguiente paso:** programar la aplicación.

## 5.5 Conexión de PHP con MariaDB

PHP tiene dos extensiones para MariaDB/MySQL: **mysqli** (específica de MySQL) y **PDO** (*PHP Data Objects*, una interfaz común para muchas bases de datos). Usaremos PDO porque su sintaxis de consultas preparadas es clara y porque el código se adapta fácilmente a otras bases de datos.

### Consultas preparadas y SQL injection

La **inyección SQL** es uno de los ataques web más frecuentes. Ocurre cuando el código mete directamente en la consulta lo que escribe el usuario:

```php title="PHP"
// ¡NUNCA HAGAS ESTO!
$email = $_GET['email'];
$sql = "SELECT * FROM contactos WHERE email = '$email'";
```

Si el atacante escribe como email `' OR '1'='1`, la consulta queda `... WHERE email = '' OR '1'='1'` y devuelve **todos** los registros. Con variantes más elaboradas puede leer otras tablas o borrar datos.

Las **consultas preparadas** separan el código SQL de los datos. Primero se envía la consulta con huecos (`:email`) y después, por separado, los valores. MariaDB nunca interpreta los valores como código:

```php title="PHP"
// Correcto: consulta preparada
$stmt = $pdo->prepare('SELECT * FROM contactos WHERE email = :email');
$stmt->execute(['email' => $_GET['email']]);
```

!!! warning "Atención"
    Regla sin excepciones: **todo dato que venga de fuera** (formularios, URL, cookies, cabeceras, APIs) va a la base de datos mediante consulta preparada, y **todo dato que se muestre en HTML** pasa por `htmlspecialchars()` para evitar ataques XSS (inyección de JavaScript).

### Estructura de archivos de la aplicación

```text title="Diagrama"
/var/www/ejemplo1.com/
├── config/
│   └── config.php        ← credenciales (600, solo web1). NO accesible por URL
├── app/
│   └── db.php            ← función de conexión reutilizable (fuera de public_html)
├── public_html/
│   ├── index.php         ← agenda de contactos: listar, crear, editar, borrar
│   └── estilo.css        ← apariencia
├── logs/php-error.log    ← errores de PHP de esta web
└── tmp/                  ← sesiones de PHP
```

### PASO 5.10 – Crear el archivo de credenciales protegido

**Objetivo:** guardar los datos de conexión fuera del DocumentRoot y legibles solo por `web1`.

**Comando:** 

```bash title="Terminal"
sudo -u web1 nano /var/www/ejemplo1.com/config/config.php
```

**Archivo:** `/var/www/ejemplo1.com/config/config.php` (nuevo, contenido completo):

```php title="PHP"
<?php
// Configuración privada de ejemplo1.com
// Este archivo está FUERA de public_html y solo lo puede leer el usuario web1.
return [
    'db' => [
        'host'    => 'localhost',   // 'localhost' = socket Unix /run/mysqld/mysqld.sock
        'name'    => 'web1_db',
        'user'    => 'web1_user',
        'pass'    => '<CLAVE_WEB1>',
        'charset' => 'utf8mb4',
    ],
    'app' => [
        'nombre'          => 'ejemplo1.com',
        'correo_contacto' => 'info@ejemplo1.com',
    ],
];
```

Protege el archivo:

```bash title="Terminal"
sudo chmod 600 /var/www/ejemplo1.com/config/config.php
sudo ls -l /var/www/ejemplo1.com/config/
```

```text title="Salida esperada"
-rw------- 1 web1 web1 512 nov  9 00:05 config.php
```

Análisis: el archivo devuelve un *array* con `return`, de modo que quien lo incluya recibe los datos en una variable sin crear variables globales. Con `host = localhost`, PDO se conecta por el socket Unix, que es más rápido que TCP y no pasa por la red.

**Comprobación:** ni Apache ni `web2` pueden leerlo:

```bash title="Terminal"
sudo -u www-data cat /var/www/ejemplo1.com/config/config.php
sudo -u web2 cat /var/www/ejemplo1.com/config/config.php
```

Ambos deben responder `Permission denied`.

### PASO 5.11 – Crear el archivo de conexión

**Objetivo:** centralizar la conexión a la base de datos en una función reutilizable.

**Comando:** 

```bash title="Terminal"
sudo install -d -o web1 -g web1 -m 700 /var/www/ejemplo1.com/app
sudo -u web1 nano /var/www/ejemplo1.com/app/db.php
```

**Archivo:** `/var/www/ejemplo1.com/app/db.php` (nuevo, contenido completo):

```php title="PHP"
<?php
declare(strict_types=1);

/**
 * Devuelve una conexión PDO a la base de datos de esta web.
 * La conexión se crea una sola vez por petición y se reutiliza.
 */
function obtenerConexion(): PDO
{
    static $pdo = null;
    if ($pdo instanceof PDO) {
        return $pdo;
    }

    $config = require __DIR__ . '/../config/config.php';
    $db = $config['db'];

    $dsn = sprintf('mysql:host=%s;dbname=%s;charset=%s', $db['host'], $db['name'], $db['charset']);

    $pdo = new PDO($dsn, $db['user'], $db['pass'], [
        // Los errores lanzan excepciones (no fallan en silencio)
        PDO::ATTR_ERRMODE            => PDO::ERRMODE_EXCEPTION,
        // Los resultados se devuelven como arrays asociativos
        PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,
        // Consultas preparadas reales en el servidor (no emuladas)
        PDO::ATTR_EMULATE_PREPARES   => false,
    ]);

    return $pdo;
}

/** Escapa texto para mostrarlo en HTML de forma segura (evita XSS). */
function e(?string $texto): string
{
    return htmlspecialchars($texto ?? '', ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8');
}
```

Análisis del código:

| Elemento | Significado |
|---|---|
| `declare(strict_types=1)` | PHP no convierte tipos automáticamente: detecta errores antes. |
| `static $pdo` | La variable conserva su valor entre llamadas: una sola conexión por petición. |
| `__DIR__` | La carpeta del archivo actual; así las rutas funcionan desde cualquier sitio. |
| DSN | Cadena de conexión: driver `mysql`, host, base de datos y juego de caracteres. |
| `ERRMODE_EXCEPTION` | Ante un error, PDO lanza una `PDOException` que podemos capturar. |
| `EMULATE_PREPARES => false` | Las consultas preparadas las hace MariaDB, no una simulación en PHP. |
| `htmlspecialchars` | Convierte `<`, `>`, `"`, `'` y `&` en entidades HTML. |

```bash title="Terminal"
sudo chmod 600 /var/www/ejemplo1.com/app/db.php
```

### PASO 5.12 – Página de prueba de conexión

**Objetivo:** verificar la conexión PHP → MariaDB antes de programar la aplicación completa.

**Comando:** 

```bash title="Terminal"
sudo -u web1 nano /var/www/ejemplo1.com/public_html/prueba-bd.php
```

**Archivo:** `/var/www/ejemplo1.com/public_html/prueba-bd.php` (temporal):

```php title="PHP"
<?php
declare(strict_types=1);
require __DIR__ . '/../app/db.php';

header('Content-Type: text/plain; charset=utf-8');
try {
    $pdo = obtenerConexion();
    $fila = $pdo->query('SELECT DATABASE() AS bd, CURRENT_USER() AS usuario, VERSION() AS version')->fetch();
    echo "Conexión correcta\n";
    echo "Base de datos: {$fila['bd']}\n";
    echo "Usuario:       {$fila['usuario']}\n";
    echo "Servidor:      {$fila['version']}\n";
    $total = $pdo->query('SELECT COUNT(*) FROM contactos')->fetchColumn();
    echo "Contactos:     {$total}\n";
} catch (PDOException $e) {
    error_log('Prueba BD: ' . $e->getMessage());
    http_response_code(500);
    echo "Error de conexión. Consulta el log de PHP de la web.\n";
}
```

**Comprobación:** 

```bash title="Terminal"
curl -s https://www.ejemplo1.com/prueba-bd.php
```

```text title="Salida esperada"
Conexión correcta
Base de datos: web1_db
Usuario:       web1_user@localhost
Servidor:      10.11.13-MariaDB-0ubuntu0.24.04.1
Contactos:     1
```

**Errores habituales (consulta siempre `sudo tail /var/www/ejemplo1.com/logs/php-error.log`):** 

| Mensaje en el log | Causa | Solución |
|---|---|---|
| `SQLSTATE[HY000] [1045] Access denied for user 'web1_user'@'localhost'` | Contraseña de `config.php` distinta a la de MariaDB. | Corrige `config.php` o cambia la clave con `ALTER USER` (PASO 4.11). |
| `SQLSTATE[HY000] [2002] No such file or directory` | MariaDB parado o socket en otra ruta. | `systemctl status mariadb`. |
| `SQLSTATE[HY000] [1049] Unknown database 'web1_db'` | Nombre de base incorrecto. | `SHOW DATABASES;` |
| `could not find driver` | Falta `php-mysql`. | `sudo apt install php-mysql && sudo systemctl restart php8.3-fpm`. |
| `Failed opening required '.../config/config.php'` + `Permission denied` | El archivo no es de `web1`. | `sudo chown web1:web1` y `chmod 600`. |

Cuando funcione, **bórrala**:

```bash title="Terminal"
sudo rm /var/www/ejemplo1.com/public_html/prueba-bd.php
```

## 5.6 Aplicación completa: agenda de contactos (CRUD)

**CRUD** son las cuatro operaciones básicas sobre datos: *Create* (crear, `INSERT`), *Read* (leer, `SELECT`), *Update* (actualizar, `UPDATE`) y *Delete* (borrar, `DELETE`). Nuestra aplicación las implementa todas con estas medidas de seguridad:

- Consultas preparadas en todas las operaciones.
- Escape HTML de toda la salida.
- Validación de datos en el servidor.
- Token **CSRF**: impide que otra web engañe al navegador del visitante para enviar formularios a la nuestra.
- Las modificaciones solo se aceptan por `POST`, y tras cada una se redirige (patrón *Post/Redirect/Get*) para que recargar la página no repita la acción.
- Los errores técnicos se registran en el log, nunca se muestran al visitante.

### PASO 5.13 – Crear la aplicación index.php

**Comando:** 

```bash title="Terminal"
sudo -u web1 nano /var/www/ejemplo1.com/public_html/index.php
```

**Archivo:** `/var/www/ejemplo1.com/public_html/index.php` (nuevo, contenido completo):

```php title="PHP"
<?php
declare(strict_types=1);

require __DIR__ . '/../app/db.php';
$config = require __DIR__ . '/../config/config.php';

session_start();
if (empty($_SESSION['csrf'])) {
    $_SESSION['csrf'] = bin2hex(random_bytes(32));
}

$errores = [];
$aviso = '';
$editar = null;
$contactos = [];
$datos = ['nombre' => '', 'email' => '', 'telefono' => '', 'notas' => ''];

/** Valida los datos del formulario y devuelve la lista de errores. */
function validar(array $d): array
{
    $err = [];
    if ($d['nombre'] === '' || mb_strlen($d['nombre']) > 100) {
        $err[] = 'El nombre es obligatorio y debe tener como máximo 100 caracteres.';
    }
    if (!filter_var($d['email'], FILTER_VALIDATE_EMAIL) || mb_strlen($d['email']) > 150) {
        $err[] = 'El email no es válido.';
    }
    if (mb_strlen($d['telefono']) > 30) {
        $err[] = 'El teléfono debe tener como máximo 30 caracteres.';
    }
    return $err;
}

try {
    $pdo = obtenerConexion();

    // ---------- Operaciones de escritura (solo por POST) ----------
    if ($_SERVER['REQUEST_METHOD'] === 'POST') {
        if (!hash_equals($_SESSION['csrf'], (string)($_POST['csrf'] ?? ''))) {
            http_response_code(400);
            exit('Petición no válida (token CSRF incorrecto). Recarga la página.');
        }

        $accion = (string)($_POST['accion'] ?? '');
        $id = filter_var($_POST['id'] ?? null, FILTER_VALIDATE_INT, ['options' => ['min_range' => 1]]);
        $datos = [
            'nombre'   => trim((string)($_POST['nombre'] ?? '')),
            'email'    => trim((string)($_POST['email'] ?? '')),
            'telefono' => trim((string)($_POST['telefono'] ?? '')),
            'notas'    => trim((string)($_POST['notas'] ?? '')),
        ];

        if ($accion === 'crear' || $accion === 'actualizar') {
            $errores = validar($datos);
        }

        if (!$errores) {
            switch ($accion) {
                case 'crear':      // CREATE
                    $stmt = $pdo->prepare(
                        'INSERT INTO contactos (nombre, email, telefono, notas)
                         VALUES (:nombre, :email, :telefono, :notas)'
                    );
                    $stmt->execute($datos);
                    header('Location: /?ok=creado');
                    exit;

                case 'actualizar': // UPDATE
                    if ($id === false) { throw new InvalidArgumentException('Id no válido'); }
                    $stmt = $pdo->prepare(
                        'UPDATE contactos
                            SET nombre = :nombre, email = :email, telefono = :telefono, notas = :notas
                          WHERE id = :id'
                    );
                    $stmt->execute($datos + ['id' => $id]);
                    header('Location: /?ok=actualizado');
                    exit;

                case 'eliminar':   // DELETE
                    if ($id === false) { throw new InvalidArgumentException('Id no válido'); }
                    $stmt = $pdo->prepare('DELETE FROM contactos WHERE id = :id');
                    $stmt->execute(['id' => $id]);
                    header('Location: /?ok=eliminado');
                    exit;

                default:
                    http_response_code(400);
                    exit('Acción desconocida.');
            }
        }
        if ($accion === 'actualizar' && $id !== false) {
            $editar = $datos + ['id' => $id];
        }
    }

    // ---------- Contacto a editar (READ de una fila) ----------
    if ($editar === null && isset($_GET['editar'])) {
        $idEditar = filter_var($_GET['editar'], FILTER_VALIDATE_INT, ['options' => ['min_range' => 1]]);
        if ($idEditar !== false) {
            $stmt = $pdo->prepare('SELECT id, nombre, email, telefono, notas FROM contactos WHERE id = :id');
            $stmt->execute(['id' => $idEditar]);
            $editar = $stmt->fetch() ?: null;
        }
    }

} catch (PDOException $ex) {
    // 1062 = clave duplicada (el email ya existe)
    if (($ex->errorInfo[1] ?? null) === 1062) {
        $errores[] = 'Ya existe un contacto con ese email.';
        if (isset($id) && $id !== false && ($accion ?? '') === 'actualizar') {
            $editar = $datos + ['id' => $id];
        }
    } else {
        error_log('Error de base de datos: ' . $ex->getMessage());
        $errores[] = 'Se ha producido un error interno. Inténtalo más tarde.';
    }
} catch (InvalidArgumentException $ex) {
    $errores[] = 'Datos no válidos.';
}

// ---------- Listado (READ de todas las filas) ----------
// Va en un bloque aparte para que el listado se muestre aunque falle una operación anterior.
try {
    $contactos = obtenerConexion()->query(
        'SELECT id, nombre, email, telefono, notas, creado_en FROM contactos ORDER BY creado_en DESC, id DESC'
    )->fetchAll();
} catch (PDOException $ex) {
    error_log('Error al leer contactos: ' . $ex->getMessage());
    $errores[] = 'No se ha podido cargar el listado de contactos.';
}

$mensajesOk = ['creado' => 'Contacto creado.', 'actualizado' => 'Contacto actualizado.', 'eliminado' => 'Contacto eliminado.'];
$aviso = $mensajesOk[$_GET['ok'] ?? ''] ?? '';
$form = $editar ?? $datos;
?>
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>Agenda · <?= e($config['app']['nombre']) ?></title>
    <link rel="stylesheet" href="/estilo.css">
</head>
<body>
<main>
    <h1>Agenda de contactos · <?= e($config['app']['nombre']) ?></h1>

    <?php if ($aviso): ?><p class="ok"><?= e($aviso) ?></p><?php endif; ?>
    <?php foreach ($errores as $err): ?><p class="error"><?= e($err) ?></p><?php endforeach; ?>

    <h2><?= $editar ? 'Editar contacto' : 'Nuevo contacto' ?></h2>
    <form method="post" action="/">
        <input type="hidden" name="csrf" value="<?= e($_SESSION['csrf']) ?>">
        <input type="hidden" name="accion" value="<?= $editar ? 'actualizar' : 'crear' ?>">
        <?php if ($editar): ?><input type="hidden" name="id" value="<?= (int)$editar['id'] ?>"><?php endif; ?>
        <label>Nombre <input name="nombre" required maxlength="100" value="<?= e($form['nombre']) ?>"></label>
        <label>Email <input name="email" type="email" required maxlength="150" value="<?= e($form['email']) ?>"></label>
        <label>Teléfono <input name="telefono" maxlength="30" value="<?= e($form['telefono']) ?>"></label>
        <label>Notas <textarea name="notas" rows="3"><?= e($form['notas']) ?></textarea></label>
        <button type="submit"><?= $editar ? 'Guardar cambios' : 'Añadir contacto' ?></button>
        <?php if ($editar): ?><a href="/">Cancelar</a><?php endif; ?>
    </form>

    <h2>Contactos (<?= count($contactos) ?>)</h2>
    <table>
        <thead><tr><th>Nombre</th><th>Email</th><th>Teléfono</th><th>Notas</th><th>Alta</th><th></th></tr></thead>
        <tbody>
        <?php foreach ($contactos as $c): ?>
            <tr>
                <td><?= e($c['nombre']) ?></td>
                <td><?= e($c['email']) ?></td>
                <td><?= e($c['telefono']) ?></td>
                <td><?= e($c['notas']) ?></td>
                <td><?= e($c['creado_en']) ?></td>
                <td class="acciones">
                    <a href="/?editar=<?= (int)$c['id'] ?>">Editar</a>
                    <form method="post" action="/" onsubmit="return confirm('¿Eliminar este contacto?');">
                        <input type="hidden" name="csrf" value="<?= e($_SESSION['csrf']) ?>">
                        <input type="hidden" name="accion" value="eliminar">
                        <input type="hidden" name="id" value="<?= (int)$c['id'] ?>">
                        <button type="submit" class="peligro">Eliminar</button>
                    </form>
                </td>
            </tr>
        <?php endforeach; ?>
        </tbody>
    </table>
</main>
</body>
</html>
```

Análisis del código por bloques:

| Bloque | Qué hace |
|---|---|
| `session_start()` y `$_SESSION['csrf']` | Crea una sesión (guardada en `tmp/`) con un token aleatorio de 64 caracteres. Cada formulario lo envía y el servidor lo compara con `hash_equals()` (comparación resistente a ataques de tiempo). |
| `validar()` | Comprueba longitudes y formato del email con `filter_var`. Validar en el servidor es obligatorio: los controles del navegador (`required`, `type="email"`) se pueden saltar. |
| `INSERT ... VALUES (:nombre, ...)` + `execute($datos)` | **Create** con consulta preparada. Las claves del array coinciden con los marcadores. |
| `UPDATE ... WHERE id = :id` | **Update**. El `WHERE` limita el cambio a un único contacto. |
| `DELETE ... WHERE id = :id` | **Delete**. Llega por `POST` con token CSRF, nunca por un enlace `GET`. |
| `SELECT ... WHERE id = :id` | **Read** de una fila para rellenar el formulario de edición. |
| `SELECT ... ORDER BY creado_en DESC` | **Read** de todas las filas. No tiene datos externos, así que `query()` es seguro. |
| `header('Location: /?ok=creado')` | Patrón Post/Redirect/Get. |
| `catch (PDOException $ex)` | El código 1062 (duplicado) se traduce a un mensaje amable; cualquier otro error se registra con `error_log()` y el usuario ve un texto genérico. |
| `(int)$c['id']` y `e(...)` | Toda salida se escapa o se fuerza a número. |

### PASO 5.14 – Añadir la hoja de estilos

**Archivo:** `/var/www/ejemplo1.com/public_html/estilo.css` (nuevo, contenido completo):

```css title="CSS"
body { font-family: system-ui, sans-serif; margin: 0; background: #f4f6f9; color: #1f2933; }
main { max-width: 980px; margin: 2rem auto; background: #fff; padding: 2rem; border-radius: 8px; box-shadow: 0 2px 8px rgba(0,0,0,.08); }
h1 { margin-top: 0; color: #1f3864; }
label { display: block; margin: .6rem 0; font-weight: 600; }
input, textarea { display: block; width: 100%; padding: .5rem; margin-top: .25rem; border: 1px solid #c3ccd6; border-radius: 4px; box-sizing: border-box; font: inherit; }
button { background: #2e75b6; color: #fff; border: 0; padding: .55rem 1rem; border-radius: 4px; cursor: pointer; }
button.peligro { background: #c0392b; padding: .3rem .6rem; }
table { width: 100%; border-collapse: collapse; margin-top: 1rem; font-size: .92rem; }
th, td { text-align: left; padding: .5rem; border-bottom: 1px solid #e3e8ee; vertical-align: top; }
th { background: #1f3864; color: #fff; }
.acciones { white-space: nowrap; }
.acciones form { display: inline; }
.ok { background: #eaf5ea; border-left: 4px solid #538135; padding: .6rem; }
.error { background: #fdf0e6; border-left: 4px solid #c55a11; padding: .6rem; }
```

Fija los permisos de los archivos públicos y elimina la página de bienvenida estática (ya no se usa, porque `index.php` tiene prioridad):

```bash title="Terminal"
sudo chmod 640 /var/www/ejemplo1.com/public_html/index.php /var/www/ejemplo1.com/public_html/estilo.css
sudo rm -f /var/www/ejemplo1.com/public_html/index.html
sudo ls -l /var/www/ejemplo1.com/public_html/
```

### PASO 5.15 – Probar la aplicación desde el navegador

**Objetivo:** verificar las cuatro operaciones CRUD.

**Comprobación:** abre `https://www.ejemplo1.com` y realiza esta secuencia:

1. Debe aparecer la agenda con el contacto «Ana Pérez» creado en el Módulo 4 (**Read**).
2. Añade un contacto nuevo (**Create**). Debe aparecer el aviso «Contacto creado.»
3. Intenta añadir otro con el mismo email. Debe aparecer «Ya existe un contacto con ese email.»
4. Pulsa «Editar», cambia el teléfono y guarda (**Update**).
5. Pulsa «Eliminar» y confirma (**Delete**).

Comprueba en la base de datos que los cambios son reales:

```bash title="Terminal"
sudo mariadb -e "SELECT id, nombre, email, telefono FROM web1_db.contactos;"
```

Prueba también la protección frente a inyección SQL: en el campo email del formulario de edición la validación rechazará `' OR '1'='1`; y aunque un atacante saltara la validación, la consulta preparada trataría ese texto como un valor literal.

**Errores habituales:** si la página aparece en blanco o con «error interno», mira `sudo tail -n 20 /var/www/ejemplo1.com/logs/php-error.log`. Un error de sintaxis aparecerá como `PHP Parse error: syntax error, unexpected ... in /var/www/ejemplo1.com/public_html/index.php on line N`.

### PASO 5.16 – Desplegar la misma aplicación en ejemplo2.com

**Objetivo:** tener la aplicación en la segunda web, conectada a **su** base de datos.

**Comando:** copiamos el código y creamos una configuración propia:

```bash title="Terminal"
sudo install -d -o web2 -g web2 -m 700 /var/www/ejemplo2.com/app
sudo install -o web2 -g web2 -m 600 /var/www/ejemplo1.com/app/db.php /var/www/ejemplo2.com/app/db.php
sudo install -o web2 -g www-data -m 640 /var/www/ejemplo1.com/public_html/index.php /var/www/ejemplo2.com/public_html/index.php
sudo install -o web2 -g www-data -m 640 /var/www/ejemplo1.com/public_html/estilo.css /var/www/ejemplo2.com/public_html/estilo.css
sudo rm -f /var/www/ejemplo2.com/public_html/index.html
sudo -u web2 nano /var/www/ejemplo2.com/config/config.php
```

**Análisis del comando:** `install` copia el archivo y fija propietario, grupo y permisos en una sola orden (un `cp` normal dejaría a root como propietario).

**Archivo:** `/var/www/ejemplo2.com/config/config.php` (nuevo, contenido completo):

```php title="PHP"
<?php
// Configuración privada de ejemplo2.com
return [
    'db' => [
        'host'    => 'localhost',
        'name'    => 'web2_db',
        'user'    => 'web2_user',
        'pass'    => '<CLAVE_WEB2>',
        'charset' => 'utf8mb4',
    ],
    'app' => [
        'nombre'          => 'ejemplo2.com',
        'correo_contacto' => 'soporte@ejemplo2.com',
    ],
];
```

```bash title="Terminal"
sudo chmod 600 /var/www/ejemplo2.com/config/config.php
```

**Comprobación:** abre `https://www.ejemplo2.com`. Debe mostrar «Agenda de contactos · ejemplo2.com» con el contacto «Luis Gómez». Los contactos que añadas en una web **no** aparecen en la otra: cada una tiene su propia base de datos, su propio usuario de base de datos y su propio usuario Linux.

## 5.7 Protección de credenciales: resumen

| Medida | Dónde se aplica |
|---|---|
| Credenciales fuera del DocumentRoot | `config/config.php` |
| Archivo legible solo por el usuario de la web | `chmod 600`, propietario `web1` |
| Un usuario de base de datos por web, con privilegios mínimos | `GRANT SELECT, INSERT, UPDATE, DELETE` |
| Usuario de base de datos limitado a `localhost` | `'web1_user'@'localhost'` |
| MariaDB no escucha en Internet | `bind-address = 127.0.0.1` |
| Errores técnicos nunca visibles | `display_errors = off`, `error_log()` |
| Jaula de archivos de PHP | `open_basedir` |
| No guardar el archivo de configuración en repositorios públicos (Git) | Añade `config/` a `.gitignore` |

## 5.8 Ejercicios prácticos del Módulo 5

1. Añade a la agenda un buscador por nombre usando `SELECT ... WHERE nombre LIKE :q` con el valor `'%' . $busqueda . '%'` pasado como parámetro.
2. Escribe en `/var/www/ejemplo1.com/public_html/intrusion.php` un script que intente leer `/var/www/ejemplo2.com/config/config.php`. Ejecútalo desde el navegador, observa el error de `open_basedir` en el log y borra el script.
3. Cambia `pm.max_children` del pool `web2` a 2, abre muchas pestañas a la vez y busca en `/var/log/php8.3-fpm.log` el aviso `server reached pm.max_children setting`.
4. Rompe a propósito la contraseña en `config.php` de ejemplo2 y documenta qué ve el visitante y qué aparece en el log. Restáurala.
5. Reescribe la consulta de borrado usando `mysqli` con `prepare()`, `bind_param()` y `execute()` para comparar ambas extensiones.
