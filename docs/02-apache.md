# Módulo 2. Instalación y configuración de Apache

## Objetivos del módulo

- Instalar Apache HTTP Server y dejarlo arrancando automáticamente.
- Conocer la estructura de directorios y los archivos de configuración de Apache en Ubuntu.
- Entender el directorio raíz de las webs, los propietarios y los permisos.
- Crear una página de prueba y verla desde el navegador.
- Activar y desactivar módulos, sitios y configuraciones con las herramientas `a2*`.
- Comprobar errores de configuración y leer los logs de acceso y de errores.

## 2.1 Qué es Apache y cómo funciona

**Apache HTTP Server** es un servidor web: un programa que escucha en los puertos 80 (HTTP) y 443 (HTTPS), recibe peticiones de los navegadores y responde con archivos (HTML, imágenes, CSS) o con el resultado de ejecutar un programa (por ejemplo, PHP).

Una petición HTTP tiene este aspecto (es texto plano):

```text
GET /index.html HTTP/1.1
Host: www.ejemplo1.com
User-Agent: Mozilla/5.0 ...
```

Y la respuesta de Apache:

```text
HTTP/1.1 200 OK
Server: Apache
Content-Type: text/html; charset=UTF-8

<html> ... contenido de la página ... </html>
```

La cabecera **Host** es la clave del Virtual Hosting (Módulo 3): gracias a ella, Apache sabe a qué web quiere ir el visitante aunque todas compartan la misma IP.

Apache atiende muchas peticiones a la vez gracias a un **MPM** (*Multi-Processing Module*). En Ubuntu 24.04 se usa por defecto **mpm_event**, que gestiona miles de conexiones con pocos procesos. Es el MPM recomendado cuando PHP se ejecuta aparte con PHP-FPM, como haremos en el Módulo 5.

Los códigos de respuesta más habituales:

| Código | Significado | Causa típica |
|---|---|---|
| 200 | OK | Todo correcto. |
| 301 / 302 | Redirección permanente / temporal | Paso de HTTP a HTTPS, de `ejemplo1.com` a `www`. |
| 403 | Prohibido | Permisos de archivo o configuración `Require` que deniega. |
| 404 | No encontrado | El archivo no existe en el DocumentRoot. |
| 500 | Error interno | Error en el código PHP o en un `.htaccess`. |
| 502 / 503 | Puerta de enlace / servicio no disponible | PHP-FPM detenido o socket incorrecto. |

### PASO 2.1 – Instalar Apache

**Objetivo:** instalar el servidor web.

**Explicación:** el paquete `apache2` instala el programa, los módulos básicos, las herramientas de administración y una configuración por defecto que ya sirve una página de bienvenida.

**Comando:** 

```bash title="Terminal"
sudo apt update
sudo apt install -y apache2
```

**Análisis del comando:** `apt install` instala el paquete y todas sus dependencias (`apache2-bin`, `apache2-data`, `apache2-utils`...). `-y` acepta sin preguntar.

**Resultado esperado:** termina con líneas como `Setting up apache2 (2.4.58-1ubuntu8...)` y `Created symlink ... apache2.service`. Esa última indica que el servicio ya está registrado para arrancar con el sistema.

**Comprobación:** 

```bash title="Terminal"
apache2 -v
```

```text title="Salida esperada"
Server version: Apache/2.4.58 (Ubuntu)
Server built:   2025-xx-xx...
```

**Errores habituales:** `Unable to locate package apache2` indica que no ejecutaste `apt update` o que no hay red.

**Siguiente paso:** abrir el cortafuegos y gestionar el servicio.

### PASO 2.2 – Permitir el tráfico web en el cortafuegos

**Objetivo:** permitir conexiones a los puertos 80 y 443.

**Comando:** 

```bash title="Terminal"
sudo ufw app list
sudo ufw allow "Apache Full"
sudo ufw status
```

**Análisis del comando:** Apache instala tres perfiles de UFW: `Apache` (solo 80), `Apache Secure` (solo 443) y `Apache Full` (80 y 443). Usamos el completo porque el puerto 80 es necesario para redirigir a HTTPS y para que Let's Encrypt valide el dominio.

**Resultado esperado:** en `ufw status` aparecerá la línea `Apache Full  ALLOW  Anywhere`, además de `OpenSSH`.

!!! warning "Atención"
    Si tu proveedor tiene además un **cortafuegos externo** (en AWS «Security Groups», en Oracle «Security Lists», en Hetzner «Firewalls»...), también debes abrir allí los puertos. UFW solo controla el cortafuegos del propio servidor.

**Siguiente paso:** aprender a gestionar el servicio.

## 2.2 Gestión del servicio con systemctl

`systemctl` es la herramienta con la que hablas con systemd. Funciona igual para todos los servicios del manual (`apache2`, `mariadb`, `postfix`, `dovecot`, `php8.3-fpm`, `opendkim`, `fail2ban`, `ssh`).

| Comando | Qué hace |
|---|---|
| `sudo systemctl start apache2` | Arranca el servicio. |
| `sudo systemctl stop apache2` | Lo detiene. |
| `sudo systemctl restart apache2` | Lo detiene y lo arranca (corta las conexiones en curso). |
| `sudo systemctl reload apache2` | Relee la configuración sin cortar conexiones. **Úsalo tras cambiar configuración.** |
| `systemctl status apache2` | Muestra si está activo, desde cuándo y las últimas líneas del log. |
| `sudo systemctl enable apache2` | Hace que arranque automáticamente al iniciar el sistema. |
| `sudo systemctl disable apache2` | Quita el arranque automático. |
| `systemctl is-active apache2` | Responde solo `active` o `inactive` (útil en scripts). |
| `systemctl is-enabled apache2` | Responde `enabled` o `disabled`. |
| `journalctl -u apache2 -n 50` | Muestra las últimas 50 líneas del registro de systemd para ese servicio. |

### PASO 2.3 – Comprobar que Apache está activo y habilitado

**Objetivo:** verificar que Apache funciona y arrancará tras cada reinicio.

**Comando:** 

```bash title="Terminal"
sudo systemctl enable --now apache2
systemctl status apache2 --no-pager
```

**Análisis del comando:** `enable --now` habilita el arranque automático y, a la vez, arranca el servicio si no lo estaba. `--no-pager` muestra la salida completa sin entrar en el visor paginado (si entras en él, sales con la tecla `q`).

**Resultado esperado:** 

```text title="Salida esperada"
● apache2.service - The Apache HTTP Server
     Loaded: loaded (/usr/lib/systemd/system/apache2.service; enabled; preset: enabled)
     Active: active (running) since Sun 2026-11-08 21:52:04 CET; 3min ago
       Docs: https://httpd.apache.org/docs/2.4/
   Main PID: 2741 (apache2)
      Tasks: 55 (limit: 2275)
     Memory: 5.3M
```

Las palabras clave son **enabled** (arranca solo) y **active (running)** (está funcionando).

**Comprobación:** abre en tu navegador `http://203.0.113.10` (tu IP). Debe aparecer la página **«Apache2 Default Page – It works!»**. Desde el propio servidor también puedes comprobarlo:

```bash title="Terminal"
curl -I http://localhost
```

```text title="Salida esperada"
HTTP/1.1 200 OK
Date: Sun, 08 Nov 2026 20:55:41 GMT
Server: Apache/2.4.58 (Ubuntu)
Content-Type: text/html
```

**Errores habituales:** 

- El navegador no carga nada: revisa UFW (PASO 2.2) y el cortafuegos del proveedor.
- `Active: failed`: mira el motivo con `journalctl -u apache2 -n 30 --no-pager`. Suele ser que otro programa (por ejemplo nginx) ocupa el puerto 80; compruébalo con `sudo ss -tlnp | grep ':80'`.

**Siguiente paso:** conocer dónde vive la configuración.

## 2.3 Estructura de directorios de Apache en Ubuntu

Ubuntu (y Debian) organizan Apache de forma modular, distinta de otras distribuciones como Rocky Linux o Fedora (que usan `/etc/httpd/` y el servicio `httpd`). Todo lo que hagas en este manual corresponde a la organización de Debian/Ubuntu.

```text title="Diagrama"
/etc/apache2/
├── apache2.conf          ← archivo principal: incluye todos los demás
├── envvars               ← variables: usuario (www-data), rutas de logs
├── ports.conf            ← en qué puertos escucha (Listen 80, Listen 443)
├── conf-available/       ← fragmentos de configuración global disponibles
├── conf-enabled/         ← enlaces a los fragmentos activos  (a2enconf/a2disconf)
├── mods-available/       ← todos los módulos instalados (.load y .conf)
├── mods-enabled/         ← enlaces a los módulos activos      (a2enmod/a2dismod)
├── sites-available/      ← un archivo por cada sitio web (VirtualHost)
└── sites-enabled/        ← enlaces a los sitios activos       (a2ensite/a2dissite)

/var/www/html/            ← DocumentRoot del sitio por defecto
/var/log/apache2/         ← access.log, error.log, other_vhosts_access.log
/usr/lib/systemd/system/apache2.service  ← unidad de systemd
```

La idea clave del par **available / enabled**: en las carpetas `*-available` están todos los elementos que **podrías** usar; en `*-enabled` hay **enlaces simbólicos** a los que **estás** usando. Apache solo lee las carpetas `*-enabled`. Las herramientas `a2en*` crean esos enlaces y las `a2dis*` los borran. Así puedes activar o desactivar un sitio sin borrar su configuración.

### Archivos principales

| Archivo | Contenido | ¿Lo modificaremos? |
|---|---|---|
| `/etc/apache2/apache2.conf` | Configuración global: tiempos de espera, usuario, permisos de directorios, inclusión del resto de archivos. | Casi nunca. |
| `/etc/apache2/ports.conf` | Directivas `Listen`. | No (ya escucha en 80 y en 443 cuando se activa SSL). |
| `/etc/apache2/envvars` | Define `APACHE_RUN_USER=www-data` y `APACHE_RUN_GROUP=www-data`. | No. |
| `/etc/apache2/conf-available/security.conf` | Cabeceras e información que Apache revela. | Sí (Módulo 12). |
| `/etc/apache2/sites-available/000-default.conf` | VirtualHost por defecto. | Lo usaremos como «sitio comodín». |

Echa un vistazo a cómo `apache2.conf` incluye el resto:

```bash title="Terminal"
grep -E '^(Include|IncludeOptional)' /etc/apache2/apache2.conf
```

```text title="Salida esperada"
IncludeOptional mods-enabled/*.load
IncludeOptional mods-enabled/*.conf
Include ports.conf
IncludeOptional conf-enabled/*.conf
IncludeOptional sites-enabled/*.conf
```

Y la sección que controla qué carpetas puede servir Apache:

```bash title="Terminal"
sed -n '/<Directory \/>/,/<\/Directory>/p;/<Directory \/var\/www\/>/,/<\/Directory>/p' /etc/apache2/apache2.conf
```

```text title="Salida esperada"
<Directory />
        Options FollowSymLinks
        AllowOverride None
        Require all denied
</Directory>
<Directory /var/www/>
        Options Indexes FollowSymLinks
        AllowOverride None
        Require all granted
</Directory>
```

Esto significa: por defecto Apache **deniega** el acceso a todo el sistema de archivos (`/`) y solo **permite** servir lo que esté bajo `/var/www/`. Por eso todas nuestras webs vivirán en `/var/www/`.

!!! tip "Consejo"
    La palabra `Indexes` permite que Apache muestre un listado de archivos si una carpeta no tiene `index.html`. Eso puede revelar archivos que no quieres publicar. En nuestros VirtualHosts lo desactivaremos con `Options -Indexes`.

## 2.4 El directorio raíz de las webs, propietarios y permisos

El **DocumentRoot** es la carpeta cuyo contenido se publica. Si el DocumentRoot es `/var/www/html`, la URL `http://servidor/fotos/a.jpg` corresponde al archivo `/var/www/html/fotos/a.jpg`.

Apache se ejecuta con el usuario **www-data**. Para servir un archivo estático, `www-data` necesita:

- Permiso `x` en **todas** las carpetas de la ruta (`/`, `/var`, `/var/www`, `/var/www/html`).
- Permiso `r` sobre el propio archivo.

Lo que **no** debe tener `www-data`: permiso de escritura sobre el código de la web. Si un atacante consiguiera ejecutar algo a través de Apache, no podría modificar tus páginas.

### PASO 2.4 – Crear una página de prueba

**Objetivo:** publicar una página propia en el sitio por defecto y entender la relación entre archivo y URL.

**Comando:** 

```bash title="Terminal"
ls -l /var/www/html/
echo '<h1>Servidor mail.ejemplo1.com funcionando</h1>' | sudo tee /var/www/html/prueba.html
sudo chmod 644 /var/www/html/prueba.html
ls -l /var/www/html/
```

**Análisis del comando:** 

- `echo '...'` escribe el texto en la salida.
- `| sudo tee archivo` recibe ese texto y lo escribe en el archivo **con privilegios de root** (un simple `sudo echo ... > archivo` no funcionaría, porque la redirección `>` la hace tu shell sin privilegios).
- `chmod 644`: el propietario (root) lee y escribe; el resto, solo lee. `www-data` entra en «el resto», así que puede leerlo.

**Resultado esperado:** 

```text title="Salida esperada"
-rw-r--r-- 1 root root 10671 nov  8 21:52 index.html
-rw-r--r-- 1 root root    49 nov  8 22:01 prueba.html
```

**Comprobación:** abre `http://203.0.113.10/prueba.html` o, desde el servidor:

```bash title="Terminal"
curl http://localhost/prueba.html
```

```text title="Salida esperada"
<h1>Servidor mail.ejemplo1.com funcionando</h1>
```

Ahora provoca un error 403 a propósito para entender los permisos:

```bash title="Terminal"
sudo chmod 600 /var/www/html/prueba.html
curl -I http://localhost/prueba.html
sudo chmod 644 /var/www/html/prueba.html
```

La respuesta intermedia será `HTTP/1.1 403 Forbidden`: `www-data` ya no podía leer el archivo. Tras restaurar `644`, vuelve a funcionar.

**Errores habituales:** un 404 significa que el archivo no está en la ruta que crees; comprueba mayúsculas y minúsculas (Linux las distingue: `Prueba.html` ≠ `prueba.html`).

**Siguiente paso:** aprender a comprobar la configuración.

## 2.5 Comprobación de errores de configuración: apachectl

Antes de recargar Apache tras un cambio, **comprueba siempre la sintaxis**. Si recargas con un error, Apache puede negarse a arrancar y dejar todas tus webs caídas.

| Comando | Qué hace |
|---|---|
| `sudo apachectl configtest` | Comprueba la sintaxis. Debe responder `Syntax OK`. |
| `sudo apachectl -S` | Muestra los VirtualHosts cargados, sus puertos y archivos. Fundamental en el Módulo 3. |
| `sudo apachectl -M` | Lista los módulos cargados. |
| `sudo apachectl -t -D DUMP_INCLUDES` | Muestra qué archivos se incluyen y en qué orden. |

### PASO 2.5 – Comprobar la configuración y corregir el aviso de ServerName

**Objetivo:** obtener un `Syntax OK` limpio, sin avisos.

**Comando:** 

```bash title="Terminal"
sudo apachectl configtest
```

**Resultado esperado (en una instalación nueva):** 

```text title="Salida esperada"
AH00558: apache2: Could not reliably determine the server's fully qualified domain name, using 127.0.1.1. Set the 'ServerName' directive globally to suppress this message
Syntax OK
```

**Explicación:** no es un error, sino un aviso: Apache no tiene un nombre global configurado. Lo corregimos creando un pequeño fragmento de configuración global.

**Comando:** 

```bash title="Terminal"
echo 'ServerName mail.ejemplo1.com' | sudo tee /etc/apache2/conf-available/servername.conf
sudo a2enconf servername
sudo apachectl configtest
sudo systemctl reload apache2
```

**Análisis del comando:** creamos `/etc/apache2/conf-available/servername.conf` con una sola directiva y lo activamos con `a2enconf` (que crea el enlace en `conf-enabled/`). Después comprobamos y recargamos.

**Resultado esperado:** `configtest` responde solo `Syntax OK`.

**Comprobación:** `ls -l /etc/apache2/conf-enabled/servername.conf` debe mostrar un enlace (`->`) a `../conf-available/servername.conf`.

**Siguiente paso:** gestionar módulos.

## 2.6 Activación y desactivación de módulos, sitios y configuraciones

Las herramientas `a2*` funcionan todas igual: crean o eliminan enlaces y te recuerdan que recargues Apache.

| Herramienta | Actúa sobre | Ejemplo |
|---|---|---|
| `a2enmod` / `a2dismod` | Módulos (`mods-available`) | `sudo a2enmod rewrite` |
| `a2ensite` / `a2dissite` | Sitios (`sites-available`) | `sudo a2ensite ejemplo1.com` |
| `a2enconf` / `a2disconf` | Configuraciones (`conf-available`) | `sudo a2enconf servername` |

Algunos módulos que usaremos:

| Módulo | Para qué |
|---|---|
| `ssl` | HTTPS. Lo activará Certbot. |
| `rewrite` | Reescritura de URL y redirecciones avanzadas. |
| `headers` | Añadir o modificar cabeceras HTTP (seguridad). |
| `proxy` y `proxy_fcgi` | Pasar las peticiones PHP a PHP-FPM. |
| `setenvif` | Variables según la petición (lo requiere la configuración de PHP-FPM). |
| `http2` | Protocolo HTTP/2, más eficiente. |

### PASO 2.6 – Activar los módulos básicos

**Objetivo:** activar los módulos que necesitaremos en los módulos siguientes y practicar el ciclo activar → comprobar → recargar.

**Comando:** 

```bash title="Terminal"
sudo a2enmod rewrite headers http2
sudo apachectl configtest
sudo systemctl restart apache2
sudo apachectl -M | grep -E 'rewrite|headers|http2|mpm'
```

**Análisis del comando:** activamos tres módulos de una vez. Tras activar **módulos** se recomienda `restart` (no basta `reload` para algunos). El último comando filtra la lista de módulos cargados.

**Resultado esperado:** 

```text title="Salida esperada"
 mpm_event_module (shared)
 headers_module (shared)
 http2_module (shared)
 rewrite_module (shared)
```

Ahora practica desactivar uno y volver a activarlo:

```bash title="Terminal"
sudo a2dismod http2
sudo systemctl restart apache2
sudo apachectl -M | grep http2 || echo "http2 desactivado"
sudo a2enmod http2
sudo systemctl restart apache2
```

**Errores habituales:** `ERROR: Module xyz does not exist!` significa que el módulo no está instalado o que el nombre está mal escrito (lista los disponibles con `ls /etc/apache2/mods-available/`).

**Siguiente paso:** entender los logs.

## 2.7 Logs de acceso y de errores

Los logs son la primera herramienta de diagnóstico de un administrador. Apache escribe en `/var/log/apache2/`:

| Archivo | Contenido |
|---|---|
| `access.log` | Una línea por petición atendida por el sitio por defecto. |
| `error.log` | Errores y avisos de Apache y de los módulos. |
| `other_vhosts_access.log` | Accesos a VirtualHosts que no definen su propio log. |
| `ejemplo1.com-access.log` (lo crearemos) | Accesos al sitio ejemplo1.com. |
| `ejemplo1.com-error.log` (lo crearemos) | Errores del sitio ejemplo1.com. |

Una línea de `access.log` (formato *combined*):

```text title="Salida esperada"
198.51.100.7 - - [08/Nov/2026:22:10:31 +0100] "GET /prueba.html HTTP/1.1" 200 289 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64)"
```

| Campo | Valor | Significado |
|---|---|---|
| IP del cliente | 198.51.100.7 | Quién hizo la petición. |
| Fecha | [08/Nov/2026:22:10:31 +0100] | Cuándo. |
| Petición | "GET /prueba.html HTTP/1.1" | Método, ruta y protocolo. |
| Código | 200 | Resultado. |
| Tamaño | 289 | Bytes enviados. |
| Referer | "-" | Página de la que venía el visitante. |
| User-Agent | "Mozilla/5.0 ..." | Navegador o robot. |

### PASO 2.7 – Ver los logs en tiempo real

**Objetivo:** aprender a observar lo que ocurre mientras haces pruebas.

**Comando:** 

```bash title="Terminal"
sudo tail -f /var/log/apache2/access.log /var/log/apache2/error.log
```

**Análisis del comando:** `tail` muestra el final de un archivo; `-f` (*follow*) se queda esperando y muestra cada línea nueva al instante. Con dos archivos, los muestra con un encabezado `==> archivo <==`.

**Comprobación:** con el comando en marcha, recarga `http://TU_IP/prueba.html` y luego visita `http://TU_IP/no-existe`. Verás aparecer una línea con `200` y otra con `404`. Pulsa **Ctrl+C** para salir.

Otros comandos útiles con logs:

```bash title="Terminal"
# Las 20 últimas líneas de errores
sudo tail -n 20 /var/log/apache2/error.log
# Cuántas peticiones ha hecho cada IP (top 10)
sudo awk '{print $1}' /var/log/apache2/access.log | sort | uniq -c | sort -rn | head
# Todas las peticiones que dieron error 404
sudo grep '" 404 ' /var/log/apache2/access.log
```

!!! tip "Consejo"
    Los logs rotan automáticamente cada día gracias a **logrotate** (`/etc/logrotate.d/apache2`): el de ayer pasa a llamarse `access.log.1`, los anteriores se comprimen (`access.log.2.gz`) y se conservan 14 días. Para leer uno comprimido usa `zcat` o `zgrep`.

**Siguiente paso:** Módulo 3, alojar varias webs con VirtualHosts.

## 2.8 Ejercicios prácticos del Módulo 2

1. Detén Apache con `systemctl stop`, comprueba desde el navegador que la web no responde, consulta `systemctl status` y vuelve a arrancarlo.
2. Introduce a propósito una errata en `/etc/apache2/conf-available/servername.conf` (por ejemplo `ServerNam`), ejecuta `apachectl configtest`, lee el mensaje de error y corrígelo. ¿Qué habría pasado si hubieras hecho `restart` sin comprobar?
3. Crea una carpeta `/var/www/html/sinindice` con un archivo `a.txt` y visítala. ¿Qué ves? ¿Qué directiva lo provoca?
4. Usa `awk` sobre `access.log` para averiguar qué rutas se piden más a menudo (pista: el campo 7).
5. Busca en `error.log` líneas con la palabra `AH` y averigua qué significan sus códigos en la documentación oficial de Apache.
