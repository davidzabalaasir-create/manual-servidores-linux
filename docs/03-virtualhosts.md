# Módulo 3. Configuración de Virtual Hosting

## Objetivos del módulo

- Entender qué es un VirtualHost y cómo un solo servidor aloja muchos dominios.
- Crear para cada dominio un usuario Linux, una carpeta y un archivo de configuración independientes.
- Configurar `ServerName`, `ServerAlias`, `DocumentRoot`, permisos y logs propios.
- Habilitar los sitios, probarlos antes y después de configurar el DNS y resolver conflictos.
- Proteger cada dominio con HTTPS mediante un certificado de Let's Encrypt.

## 3.1 Qué es un VirtualHost

Un **VirtualHost** (anfitrión virtual) es un bloque de configuración que le dice a Apache: «cuando llegue una petición para **este** nombre, sírvela desde **esta** carpeta, con **estas** opciones y guarda los logs **aquí**».

Apache admite dos tipos:

- **Por IP**: cada web tiene una IP distinta. Hoy apenas se usa porque las IPv4 son escasas.
- **Por nombre** (*name-based*): todas las webs comparten IP y Apache las distingue por la cabecera `Host` de la petición. Es lo que usaremos.

Cómo elige Apache el VirtualHost:

```text title="Diagrama"
 Petición:  GET / HTTP/1.1   Host: www.ejemplo2.com      (llega a 203.0.113.10:443)
                     │
                     ▼
   1. Apache busca los VirtualHost definidos para la IP y el puerto (*:443)
   2. Compara el Host con ServerName y ServerAlias de cada uno, por orden:
        ┌────────────────────────┬────────────────────────────────────┐
        │ default-ssl (comodín)  │ ServerName mail.ejemplo1.com        │ ✗
        │ ejemplo1.com           │ www.ejemplo1.com / ejemplo1.com     │ ✗
        │ ejemplo2.com           │ www.ejemplo2.com / ejemplo2.com     │ ✓  ← este
        └────────────────────────┴────────────────────────────────────┘
   3. Si ninguno coincide, usa el PRIMERO de la lista (el «vhost por defecto»)
   4. Sirve /var/www/ejemplo2.com/public_html/index.php
```

En HTTPS existe un detalle adicional: la cabecera `Host` va cifrada, así que Apache no puede leerla hasta después de elegir el certificado. Para resolverlo, el navegador envía el nombre en claro al iniciar la conexión TLS mediante una extensión llamada **SNI** (*Server Name Indication*). Todos los navegadores actuales la usan.

!!! warning "Atención"
    El punto 3 es la fuente de confusión número uno: si un VirtualHost tiene un error en `ServerName` o el visitante llega con un nombre desconocido (o directamente con la IP), verá **el primer VirtualHost** cargado. Por eso dejaremos el sitio por defecto (`000-default`) como «comodín» neutro en primer lugar, para que nadie vea la web de ejemplo1 por accidente al visitar ejemplo2 mal configurado.

## 3.2 Diseño de carpetas y usuarios

Cada web tendrá su propio usuario de sistema. PHP-FPM (Módulo 5) ejecutará el código de cada web con ese usuario, de modo que una web no pueda leer los archivos de la otra.

```text title="Diagrama"
/var/www/
├── html/                         ← sitio comodín (000-default), propietario root
├── ejemplo1.com/                 ← propietario web1, grupo www-data, 750
│   ├── public_html/              ← DocumentRoot  (web1:www-data, 2750)
│   │   ├── index.html / index.php
│   │   └── .well-known/acme-challenge/   ← validación de Let's Encrypt (temporal)
│   ├── config/                   ← credenciales, FUERA del DocumentRoot (web1:web1, 700)
│   └── tmp/                      ← temporales y sesiones de PHP (web1:web1, 700)
└── ejemplo2.com/                 ← misma estructura con el usuario web2
```

¿Por qué `config/` está **fuera** de `public_html/`? Porque todo lo que está dentro de `public_html` puede pedirse por URL. Si un fallo de configuración hiciera que Apache sirviera el PHP como texto, las contraseñas de la base de datos quedarían a la vista. Fuera del DocumentRoot, eso es imposible.

### PASO 3.1 – Crear los usuarios de sistema de cada web

**Objetivo:** crear `web1` y `web2`, usuarios sin contraseña ni acceso por terminal, que serán dueños de los archivos de cada web.

**Comando:** 

```bash title="Terminal"
sudo useradd --system --user-group --home-dir /var/www/ejemplo1.com --shell /usr/sbin/nologin web1
sudo useradd --system --user-group --home-dir /var/www/ejemplo2.com --shell /usr/sbin/nologin web2
```

**Análisis del comando:** 

- `useradd`: herramienta de bajo nivel para crear usuarios (no hace preguntas, ideal para cuentas de servicio).
- `--system`: cuenta de sistema (UID bajo, sin caducidad de contraseña, no aparece en la pantalla de inicio).
- `--user-group`: crea también un grupo con el mismo nombre.
- `--home-dir`: carpeta personal del usuario (la crearemos en el paso siguiente).
- `--shell /usr/sbin/nologin`: impide iniciar sesión con este usuario. Si alguien lo intentara, recibiría «This account is currently not available».

**Comprobación:** 

```bash title="Terminal"
id web1
id web2
```

```text title="Salida esperada"
uid=998(web1) gid=998(web1) groups=998(web1)
uid=997(web2) gid=997(web2) groups=997(web2)
```

(Los números pueden ser distintos.)

**Errores habituales:** `useradd: user 'web1' already exists`: ya lo habías creado; continúa.

**Siguiente paso:** crear la estructura de carpetas.

### PASO 3.2 – Crear las carpetas y asignar propietarios y permisos

**Objetivo:** crear la estructura de cada web con los permisos correctos.

**Comando (para ejemplo1.com):** 

```bash title="Terminal"
sudo mkdir -p /var/www/ejemplo1.com/{public_html,config,tmp}
sudo chown web1:www-data /var/www/ejemplo1.com /var/www/ejemplo1.com/public_html
sudo chown web1:web1 /var/www/ejemplo1.com/config /var/www/ejemplo1.com/tmp
sudo chmod 750 /var/www/ejemplo1.com
sudo chmod 2750 /var/www/ejemplo1.com/public_html
sudo chmod 700 /var/www/ejemplo1.com/config /var/www/ejemplo1.com/tmp
```

Repite lo mismo para ejemplo2.com con el usuario `web2`:

```bash title="Terminal"
sudo mkdir -p /var/www/ejemplo2.com/{public_html,config,tmp}
sudo chown web2:www-data /var/www/ejemplo2.com /var/www/ejemplo2.com/public_html
sudo chown web2:web2 /var/www/ejemplo2.com/config /var/www/ejemplo2.com/tmp
sudo chmod 750 /var/www/ejemplo2.com
sudo chmod 2750 /var/www/ejemplo2.com/public_html
sudo chmod 700 /var/www/ejemplo2.com/config /var/www/ejemplo2.com/tmp
```

**Análisis del comando:** 

- `mkdir -p`: crea carpetas, incluidas las intermedias, sin error si ya existen. Las llaves `{a,b,c}` son una abreviatura del shell que genera tres rutas.
- `chown web1:www-data`: el dueño es `web1` (podrá escribir) y el grupo `www-data` (Apache podrá entrar y leer).
- `chmod 750` en la carpeta del sitio: dueño todo, grupo (Apache) entrar y listar, el resto nada. Así `web2` no puede entrar en la carpeta de `web1`.
- `chmod 2750` en `public_html`: el `2` inicial activa el bit **setgid**: todo archivo nuevo creado dentro heredará el grupo `www-data` automáticamente, en vez del grupo de quien lo cree.
- `chmod 700` en `config` y `tmp`: solo `web1` puede entrar. Apache no los necesita.

**Comprobación:** 

```bash title="Terminal"
sudo ls -la /var/www/ejemplo1.com/
```

```text title="Salida esperada"
drwxr-x---  5 web1 www-data 4096 nov  8 22:20 .
drwxr-xr-x  5 root root     4096 nov  8 22:20 ..
drwx------  2 web1 web1     4096 nov  8 22:20 config
drwxr-s---  2 web1 www-data 4096 nov  8 22:20 public_html
drwx------  2 web1 web1     4096 nov  8 22:20 tmp
```

La `s` en `drwxr-s---` es el bit setgid.

**Siguiente paso:** crear una página de bienvenida distinta para cada dominio.

### PASO 3.3 – Crear una página de prueba en cada sitio

**Objetivo:** tener contenido distinto en cada web para comprobar que cada dominio carga la suya.

**Comando:** creamos el archivo con el usuario `web1` (para que sea su dueño) usando `sudo -u`:

```bash title="Terminal"
sudo -u web1 tee /var/www/ejemplo1.com/public_html/index.html > /dev/null <<'EOF'
<!DOCTYPE html>
<html lang="es">
<head><meta charset="utf-8"><title>ejemplo1.com</title></head>
<body style="font-family: sans-serif; background: #e8f1fb;">
  <h1>Bienvenido a ejemplo1.com</h1>
  <p>Este contenido se sirve desde /var/www/ejemplo1.com/public_html</p>
</body>
</html>
EOF
```

```bash title="Terminal"
sudo -u web2 tee /var/www/ejemplo2.com/public_html/index.html > /dev/null <<'EOF'
<!DOCTYPE html>
<html lang="es">
<head><meta charset="utf-8"><title>ejemplo2.com</title></head>
<body style="font-family: sans-serif; background: #eaf5ea;">
  <h1>Bienvenido a ejemplo2.com</h1>
  <p>Este contenido se sirve desde /var/www/ejemplo2.com/public_html</p>
</body>
</html>
EOF
```

**Análisis del comando:** 

- `sudo -u web1 tee archivo`: escribe el archivo como el usuario `web1`.
- `<<'EOF' ... EOF`: un *here-document*: todo lo que hay entre las dos marcas se envía como entrada al comando. Las comillas en `'EOF'` impiden que el shell interprete caracteres especiales como `$`.
- `> /dev/null`: descarta la copia que `tee` muestra por pantalla.

Ajustamos permisos de los archivos (640: dueño lee y escribe, grupo www-data lee):

```bash title="Terminal"
sudo chmod 640 /var/www/ejemplo1.com/public_html/index.html /var/www/ejemplo2.com/public_html/index.html
sudo ls -l /var/www/ejemplo1.com/public_html/
```

```text title="Salida esperada"
-rw-r----- 1 web1 www-data 287 nov  8 22:25 index.html
```

Fíjate en que el grupo es `www-data` gracias al bit setgid.

**Siguiente paso:** preparar el sitio comodín.

### PASO 3.4 – Preparar el sitio por defecto como comodín

**Objetivo:** que quien visite el servidor por IP o con un nombre desconocido vea una página neutra.

**Comando:** 

```bash title="Terminal"
echo '<!DOCTYPE html><html><head><meta charset="utf-8"><title>Servidor</title></head><body><p>Servidor en funcionamiento.</p></body></html>' | sudo tee /var/www/html/index.html > /dev/null
```

**Archivo:** `/etc/apache2/sites-available/000-default.conf`. Sustituye todo su contenido por:

```text title="Archivo de configuración"
# Sitio comodín: atiende peticiones por IP o con nombres no configurados.
# Debe ser el PRIMER VirtualHost en *:80 (por eso se llama 000-...).
# También sirve los desafíos de Let's Encrypt para mail.ejemplo1.com y mail.ejemplo2.com.
<VirtualHost *:80>
    ServerName mail.ejemplo1.com
    ServerAlias mail.ejemplo2.com
    ServerAdmin administracion@ejemplo1.com
    DocumentRoot /var/www/html

    <Directory /var/www/html>
        Options -Indexes
        AllowOverride None
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

Análisis: `${APACHE_LOG_DIR}` es una variable definida en `/etc/apache2/envvars` que vale `/var/log/apache2`. `ServerName mail.ejemplo1.com` y su alias permitirán, en el Módulo 6, obtener el certificado del servidor de correo usando esta misma carpeta.

```bash title="Terminal"
sudo apachectl configtest && sudo systemctl reload apache2
```

El operador `&&` ejecuta la recarga **solo si** la comprobación termina bien. Acostúmbrate a esta combinación.

**Siguiente paso:** crear el VirtualHost de cada dominio.

## 3.3 Los archivos de configuración de los VirtualHosts

### PASO 3.5 – Crear el VirtualHost de ejemplo1.com (HTTP)

**Objetivo:** decirle a Apache que `www.ejemplo1.com` y `ejemplo1.com` se sirven desde `/var/www/ejemplo1.com/public_html`.

**Explicación:** de momento solo configuramos HTTP (puerto 80). Lo necesitamos así para obtener el certificado; en el PASO 3.11 ampliaremos el archivo con HTTPS.

**Comando:** 

```bash title="Terminal"
sudo nano /etc/apache2/sites-available/ejemplo1.com.conf
```

**Archivo:** `/etc/apache2/sites-available/ejemplo1.com.conf` (nuevo, contenido completo):

```text title="Archivo de configuración"
<VirtualHost *:80>
    # Nombre principal del sitio
    ServerName www.ejemplo1.com
    # Otros nombres que deben servir este mismo sitio
    ServerAlias ejemplo1.com
    # Correo del responsable (aparece en algunas páginas de error)
    ServerAdmin administracion@ejemplo1.com

    # Carpeta que se publica
    DocumentRoot /var/www/ejemplo1.com/public_html

    <Directory /var/www/ejemplo1.com/public_html>
        # Sin listados de carpetas; se permiten enlaces simbólicos
        Options -Indexes +FollowSymLinks
        # No se procesan archivos .htaccess (más rápido y más seguro)
        AllowOverride None
        # Cualquiera puede ver el contenido
        Require all granted
    </Directory>

    # Logs propios de este sitio
    ErrorLog ${APACHE_LOG_DIR}/ejemplo1.com-error.log
    CustomLog ${APACHE_LOG_DIR}/ejemplo1.com-access.log combined
</VirtualHost>
```

Análisis de cada directiva:

| Directiva | Significado |
|---|---|
| `<VirtualHost *:80>` | Este bloque atiende peticiones en cualquier IP (`*`) del servidor, puerto 80. |
| `ServerName` | Nombre principal. Debe ser único entre todos los VirtualHosts del mismo puerto. |
| `ServerAlias` | Nombres adicionales (varios separados por espacio; admite comodines como `*.ejemplo1.com`). |
| `ServerAdmin` | Dirección de contacto. |
| `DocumentRoot` | Carpeta publicada. Sin barra final. |
| `<Directory ruta>` | Opciones que se aplican a esa carpeta y sus subcarpetas. |
| `Options -Indexes` | Desactiva el listado automático de carpetas. |
| `AllowOverride None` | Ignora los archivos `.htaccess`. Toda la configuración está en este archivo. |
| `Require all granted` | Concede acceso a todo el mundo. |
| `ErrorLog` / `CustomLog` | Rutas de los logs de errores y de accesos; `combined` es el formato. |

!!! tip "Consejo"
    Muchas aplicaciones (WordPress, Laravel...) traen un `.htaccess` con reglas de reescritura. Si las usas, cambia `AllowOverride None` por `AllowOverride All` (o mejor, copia esas reglas dentro de este bloque `<Directory>`).

**Siguiente paso:** el mismo archivo para ejemplo2.com.

### PASO 3.6 – Crear el VirtualHost de ejemplo2.com (HTTP)

**Archivo:** `/etc/apache2/sites-available/ejemplo2.com.conf` (nuevo, contenido completo):

```text title="Archivo de configuración"
<VirtualHost *:80>
    ServerName www.ejemplo2.com
    ServerAlias ejemplo2.com
    ServerAdmin soporte@ejemplo2.com

    DocumentRoot /var/www/ejemplo2.com/public_html

    <Directory /var/www/ejemplo2.com/public_html>
        Options -Indexes +FollowSymLinks
        AllowOverride None
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/ejemplo2.com-error.log
    CustomLog ${APACHE_LOG_DIR}/ejemplo2.com-access.log combined
</VirtualHost>
```

**Siguiente paso:** habilitar ambos sitios.

### PASO 3.7 – Habilitar los sitios

**Objetivo:** activar los dos VirtualHosts.

**Comando:** 

```bash title="Terminal"
sudo a2ensite ejemplo1.com ejemplo2.com
sudo apachectl configtest
sudo systemctl reload apache2
```

**Análisis del comando:** `a2ensite` admite el nombre del archivo sin `.conf`. Crea `/etc/apache2/sites-enabled/ejemplo1.com.conf` como enlace al de `sites-available`.

**Resultado esperado:** 

```text title="Salida esperada"
Enabling site ejemplo1.com.
Enabling site ejemplo2.com.
To activate the new configuration, you need to run:
  systemctl reload apache2
Syntax OK
```

**Comprobación:** la herramienta más importante de este módulo:

```bash title="Terminal"
sudo apachectl -S
```

```text title="Salida esperada"
VirtualHost configuration:
*:80                   is a NameVirtualHost
         default server mail.ejemplo1.com (/etc/apache2/sites-enabled/000-default.conf:4)
         port 80 namevhost mail.ejemplo1.com (/etc/apache2/sites-enabled/000-default.conf:4)
                 alias mail.ejemplo2.com
         port 80 namevhost www.ejemplo1.com (/etc/apache2/sites-enabled/ejemplo1.com.conf:1)
                 alias ejemplo1.com
         port 80 namevhost www.ejemplo2.com (/etc/apache2/sites-enabled/ejemplo2.com.conf:1)
                 alias ejemplo2.com
ServerRoot: "/etc/apache2"
Main DocumentRoot: "/var/www/html"
...
```

Interpretación: en el puerto 80 hay tres VirtualHosts por nombre; el **default server** (el que atiende nombres desconocidos) es el comodín de `000-default.conf`. Cada línea indica el archivo y la línea donde está definido: muy útil cuando algo no cuadra.

**Siguiente paso:** probar los dominios.

## 3.4 Probar los dominios

Hay tres formas de probar, de la más rápida a la definitiva.

### PASO 3.8 – Probar desde el propio servidor con curl (sin DNS)

**Objetivo:** verificar que Apache elige el VirtualHost correcto aunque el DNS aún no esté configurado.

**Comando:** 

```bash title="Terminal"
curl -s -H "Host: www.ejemplo1.com" http://127.0.0.1/ | grep '<h1>'
curl -s -H "Host: ejemplo2.com" http://127.0.0.1/ | grep '<h1>'
curl -s -H "Host: desconocido.com" http://127.0.0.1/
```

**Análisis del comando:** `-H "Host: ..."` envía la cabecera Host que queramos, simulando que el visitante escribió ese nombre. `-s` (silencioso) oculta la barra de progreso.

**Resultado esperado:** 

```text title="Salida esperada"
  <h1>Bienvenido a ejemplo1.com</h1>
  <h1>Bienvenido a ejemplo2.com</h1>
<!DOCTYPE html><html><head><meta charset="utf-8"><title>Servidor</title></head><body><p>Servidor en funcionamiento.</p></body></html>
```

El tercer caso demuestra que los nombres desconocidos caen en el comodín.

### PASO 3.9 – Probar desde tu navegador con el archivo hosts (sin DNS)

**Objetivo:** ver las webs en tu navegador antes de tocar el DNS (o en un laboratorio sin dominio real).

**Explicación:** el archivo `hosts` de tu ordenador tiene prioridad sobre el DNS. Añadiendo líneas, tu equipo creerá que esos nombres apuntan a tu servidor.

- En **Windows**: abre el Bloc de notas **como administrador** y edita `C:\Windows\System32\drivers\etc\hosts`.
- En **macOS / Linux**: `sudo nano /etc/hosts`.

**Archivo:** `hosts` de **tu ordenador** (añade al final):

```text title="Archivo de configuración"
203.0.113.10   ejemplo1.com www.ejemplo1.com mail.ejemplo1.com
203.0.113.10   ejemplo2.com www.ejemplo2.com mail.ejemplo2.com
```

**Comprobación:** visita `http://www.ejemplo1.com` y `http://www.ejemplo2.com`. Cada uno debe mostrar su página con su color de fondo.

!!! warning "Atención"
    Cuando configures el DNS real, **borra estas líneas**. Si no, tu equipo seguirá usándolas y podrías no darte cuenta de que el DNS está mal.

### PASO 3.10 – Crear los registros DNS de las webs

**Objetivo:** que cualquier persona del mundo llegue a tus webs.

**Explicación:** entra en el panel de tu proveedor DNS y crea, para **cada** dominio, los registros siguientes. Creamos también ahora los nombres `mail` porque los necesitaremos para el certificado del correo (el resto de registros de correo se crean en el Módulo 9).

Registros de `ejemplo1.com`:

| Tipo | Nombre | Valor | TTL |
|---|---|---|---|
| A | @ | 203.0.113.10 | 3600 |
| A | www | 203.0.113.10 | 3600 |
| A | mail | 203.0.113.10 | 3600 |
| AAAA (si tienes IPv6) | @, www y mail | 2001:db8:10::10 | 3600 |

Registros de `ejemplo2.com`:

| Tipo | Nombre | Valor | TTL |
|---|---|---|---|
| A | @ | 203.0.113.10 | 3600 |
| A | www | 203.0.113.10 | 3600 |
| A | mail | 203.0.113.10 | 3600 |
| AAAA (si tienes IPv6) | @, www y mail | 2001:db8:10::10 | 3600 |

!!! warning "Atención"
    Crea registros **AAAA** solo si tu servidor tiene IPv6 configurada **y funcionando**. Un AAAA que apunta a una IPv6 que no responde hace que muchos visitantes (y Let's Encrypt) fallen, porque prefieren IPv6. Comprueba la IPv6 del servidor con `ip -6 addr show scope global`.

**Comprobación (desde el servidor o desde tu ordenador):** 

```bash title="Terminal"
dig +short A www.ejemplo1.com
dig +short A ejemplo2.com
dig +short A mail.ejemplo2.com @1.1.1.1
```

Cada consulta debe devolver `203.0.113.10`. La opción `@1.1.1.1` pregunta directamente al resolvedor de Cloudflare, útil para saltarte cachés locales.

**Errores habituales:** si no devuelve nada, espera unos minutos (propagación) y revisa que no hayas escrito el dominio completo en el campo «Nombre» (algunos paneles añaden el dominio automáticamente y acabarías creando `www.ejemplo1.com.ejemplo1.com`).

## 3.5 Solucionar conflictos entre VirtualHosts

| Síntoma | Causa probable | Solución |
|---|---|---|
| Todos los dominios muestran la misma web. | El navegador cae en el vhost por defecto porque ningún `ServerName`/`ServerAlias` coincide, o los sitios no están habilitados. | `sudo apachectl -S`: comprueba que cada nombre aparece como `namevhost` o `alias`. |
| `ejemplo2.com` muestra la web de ejemplo1. | Falta `ServerAlias ejemplo2.com` o está en el archivo equivocado. | Corrige el alias y recarga. |
| Aviso `AH01630` o `AH00548: NameVirtualHost has no effect`. | Directiva obsoleta de Apache 2.2. | Bórrala: en 2.4 no hace falta. |
| Dos sitios con el mismo `ServerName`. | Gana el primero que se carga (orden alfabético de archivos). | Cada `ServerName` debe ser único; revisa `apachectl -S`. |
| 403 Forbidden en un dominio. | Permisos de carpeta (falta `x` para www-data) o falta el bloque `<Directory>`. | `sudo -u www-data ls /var/www/ejemplo1.com/public_html` debe funcionar. |
| El cambio no se aplica. | No recargaste Apache o editaste el archivo en `sites-available` de otro sitio. | `sudo systemctl reload apache2` y revisa el archivo indicado en `apachectl -S`. |
| `Could not open configuration file ... No such file or directory`. | Un enlace en `sites-enabled` apunta a un archivo borrado. | `sudo a2dissite nombre` o borra el enlace roto. |

Una herramienta muy útil para comprobar permisos de toda una ruta de golpe:

```bash title="Terminal"
namei -l /var/www/ejemplo1.com/public_html/index.html
```

```text title="Salida esperada"
f: /var/www/ejemplo1.com/public_html/index.html
drwxr-xr-x root root     /
drwxr-xr-x root root     var
drwxr-xr-x root root     www
drwxr-x--- web1 www-data ejemplo1.com
drwxr-s--- web1 www-data public_html
-rw-r----- web1 www-data index.html
```

## 3.6 HTTPS para cada dominio

Hasta ahora todo viaja **sin cifrar**: cualquiera en la red podría leer o modificar lo que se envía. HTTPS cifra la conexión con **TLS** y además demuestra, mediante un **certificado**, que el servidor es realmente quien dice ser.

Usaremos **Let's Encrypt**, una autoridad de certificación gratuita y automatizada, a través de su cliente oficial **Certbot**. El funcionamiento interno y la renovación se explican a fondo en el Módulo 10; aquí lo usamos para dejar las webs seguras desde ya.

Cómo demuestra Certbot que el dominio es tuyo (desafío **HTTP-01**):

```text title="Diagrama"
 1. Certbot pide un certificado para ejemplo1.com y www.ejemplo1.com
 2. Let's Encrypt responde: "publica este token en
        http://ejemplo1.com/.well-known/acme-challenge/<token>"
 3. Certbot escribe el archivo en /var/www/ejemplo1.com/public_html/.well-known/acme-challenge/
 4. Los servidores de Let's Encrypt descargan esa URL desde Internet (puerto 80)
 5. Si coincide → emiten el certificado (válido 90 días). Certbot borra el token.
```

Por eso es imprescindible que el DNS apunte ya a tu servidor y que el puerto 80 esté abierto.

!!! tip "Consejo"
    Usaremos el modo `certonly --webroot`: Certbot solo obtiene el certificado y **no toca** tus archivos de Apache. Así controlas al cien por cien tu configuración y la entiendes línea por línea. (El modo `certbot --apache` modifica tus VirtualHosts automáticamente; es cómodo, pero genera archivos `-le-ssl.conf` duplicados que confunden cuando más adelante añades PHP.)

### PASO 3.11 – Instalar Certbot y obtener los certificados de las webs

**Objetivo:** obtener un certificado para cada dominio.

**Comando:** 

```bash title="Terminal"
sudo apt install -y certbot
sudo certbot certonly --webroot -w /var/www/ejemplo1.com/public_html -d ejemplo1.com -d www.ejemplo1.com --email administracion@ejemplo1.com --agree-tos --no-eff-email
```

**Análisis del comando:** 

- `certonly`: solo obtener el certificado, sin instalarlo.
- `--webroot -w carpeta`: método de validación: escribir el token en esa carpeta (el DocumentRoot del sitio).
- `-d ejemplo1.com -d www.ejemplo1.com`: los nombres que cubrirá el certificado. El primero da nombre a la carpeta del certificado.
- `--email`: dirección a la que Let's Encrypt avisará si algo va mal con la renovación.
- `--agree-tos`: aceptas los términos del servicio de Let's Encrypt (léelos en su web).
- `--no-eff-email`: no compartir tu correo con la EFF para boletines.

**Resultado esperado:** 

```text title="Salida esperada"
Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/ejemplo1.com/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/ejemplo1.com/privkey.pem
This certificate expires on 2027-02-06.
Certbot has set up a scheduled task to automatically renew this certificate in the background.
```

Repite para el segundo dominio:

```bash title="Terminal"
sudo certbot certonly --webroot -w /var/www/ejemplo2.com/public_html -d ejemplo2.com -d www.ejemplo2.com --email soporte@ejemplo2.com --agree-tos --no-eff-email
```

**Comprobación:** 

```bash title="Terminal"
sudo certbot certificates
```

Debe listar los dos certificados con sus dominios y la fecha de caducidad.

**Errores habituales:** 

- `Timeout during connect (likely firewall problem)`: el puerto 80 está cerrado en UFW o en el cortafuegos del proveedor.
- `DNS problem: NXDOMAIN looking up A for www.ejemplo1.com`: el registro DNS aún no existe o no se ha propagado.
- `Invalid response from http://.../.well-known/acme-challenge/...: 404`: el DNS apunta a otro servidor, o el `-w` no coincide con el DocumentRoot del VirtualHost que atiende ese nombre.
- `too many certificates already issued`: has superado los límites de Let's Encrypt por hacer muchas pruebas. Para practicar, añade `--test-cert` (entorno de pruebas sin límites) y repite sin él cuando todo funcione.

!!! tip "Consejo"
    **Laboratorio sin dominio real:** omite este paso y, en el archivo del PASO 3.12, sustituye las rutas del certificado por las del certificado autofirmado de Ubuntu: `SSLCertificateFile /etc/ssl/certs/ssl-cert-snakeoil.pem` y `SSLCertificateKeyFile /etc/ssl/private/ssl-cert-snakeoil.key`. El navegador mostrará un aviso de seguridad que podrás aceptar.

**Siguiente paso:** configurar el VirtualHost HTTPS.

### PASO 3.12 – Configurar HTTPS en el VirtualHost de ejemplo1.com

**Objetivo:** servir la web por HTTPS y redirigir automáticamente todo el tráfico HTTP a `https://www.ejemplo1.com`.

**Explicación:** ampliamos el archivo con un segundo bloque `<VirtualHost *:443>`. El bloque del puerto 80 pasa a tener una única función: redirigir a HTTPS (excepto las peticiones de validación de Let's Encrypt). También redirigimos `ejemplo1.com` a `www.ejemplo1.com` para que la web tenga una única dirección canónica.

**Comando:** activa los módulos necesarios:

```bash title="Terminal"
sudo a2enmod ssl rewrite headers
```

**Archivo:** `/etc/apache2/sites-available/ejemplo1.com.conf` (sustituye todo el contenido):

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
    # Las validaciones de Let's Encrypt se sirven por HTTP sin redirigir
    RewriteCond %{REQUEST_URI} !^/\.well-known/acme-challenge/
    RewriteRule ^ https://www.ejemplo1.com%{REQUEST_URI} [L,R=301]

    ErrorLog ${APACHE_LOG_DIR}/ejemplo1.com-error.log
    CustomLog ${APACHE_LOG_DIR}/ejemplo1.com-access.log combined
</VirtualHost>

# ---------------------------------------------------------------
# ejemplo1.com - HTTPS: el sitio real
# ---------------------------------------------------------------
<VirtualHost *:443>
    ServerName www.ejemplo1.com
    ServerAlias ejemplo1.com
    ServerAdmin administracion@ejemplo1.com
    DocumentRoot /var/www/ejemplo1.com/public_html

    # Cifrado TLS con el certificado de Let's Encrypt
    SSLEngine on
    SSLCertificateFile    /etc/letsencrypt/live/ejemplo1.com/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/ejemplo1.com/privkey.pem
    Protocols h2 http/1.1

    # Dirección canónica: ejemplo1.com -> www.ejemplo1.com
    RewriteEngine On
    RewriteCond %{HTTP_HOST} !^www\.ejemplo1\.com$ [NC]
    RewriteRule ^ https://www.ejemplo1.com%{REQUEST_URI} [L,R=301]

    # Indica a los navegadores que usen siempre HTTPS durante 6 meses
    Header always set Strict-Transport-Security "max-age=15552000"

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
| `RewriteEngine On` | Activa el motor de reescritura en este VirtualHost. |
| `RewriteCond %{REQUEST_URI} !^/\.well-known/acme-challenge/` | Condición: la ruta pedida **no** (`!`) empieza por la carpeta de validación. |
| `RewriteRule ^ https://www.ejemplo1.com%{REQUEST_URI} [L,R=301]` | Para cualquier ruta (`^`), redirige a la misma ruta en HTTPS. `R=301`: redirección permanente; `L`: última regla. |
| `SSLEngine on` | Activa TLS en este VirtualHost. |
| `SSLCertificateFile` | Certificado más la cadena de certificados intermedios (`fullchain.pem`). |
| `SSLCertificateKeyFile` | Clave privada (solo root puede leerla). |
| `Protocols h2 http/1.1` | Ofrece HTTP/2 y, si el cliente no lo soporta, HTTP/1.1. |
| `RewriteCond %{HTTP_HOST} !^www\.ejemplo1\.com$ [NC]` | Si el nombre pedido no es exactamente `www.ejemplo1.com` (sin distinguir mayúsculas, `NC`)... |
| `Header always set Strict-Transport-Security` | Cabecera HSTS: el navegador recordará usar solo HTTPS. Actívala cuando estés seguro de que HTTPS funciona bien. |

!!! warning "Atención"
    Los protocolos y cifrados TLS globales están definidos en `/etc/apache2/mods-available/ssl.conf` (Ubuntu 24.04 ya desactiva SSLv3, TLS 1.0 y 1.1 con `SSLProtocol all -SSLv3 -TLSv1 -TLSv1.1`). Compruébalo con `grep -E '^\s*SSLProtocol' /etc/apache2/mods-available/ssl.conf`.

**Siguiente paso:** lo mismo para ejemplo2.com.

### PASO 3.13 – Configurar HTTPS en el VirtualHost de ejemplo2.com

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

    <Directory /var/www/ejemplo2.com/public_html>
        Options -Indexes +FollowSymLinks
        AllowOverride None
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/ejemplo2.com-error.log
    CustomLog ${APACHE_LOG_DIR}/ejemplo2.com-access.log combined
</VirtualHost>
```

### PASO 3.14 – Activar el comodín HTTPS y aplicar los cambios

**Objetivo:** que las peticiones HTTPS con nombres desconocidos no caigan en ejemplo1.com.

**Explicación:** Ubuntu incluye el sitio `default-ssl`, que usa el certificado autofirmado. Como su nombre empieza por «d», se carga antes que «ejemplo1.com» y será el comodín del puerto 443.

**Comando:** 

```bash title="Terminal"
sudo a2ensite default-ssl
sudo apachectl configtest
sudo systemctl restart apache2
sudo apachectl -S
```

**Resultado esperado (fragmento):** 

```text title="Salida esperada"
*:443                  is a NameVirtualHost
         default server mail.ejemplo1.com (/etc/apache2/sites-enabled/default-ssl.conf:2)
         port 443 namevhost mail.ejemplo1.com (/etc/apache2/sites-enabled/default-ssl.conf:2)
         port 443 namevhost www.ejemplo1.com (/etc/apache2/sites-enabled/ejemplo1.com.conf:22)
                 alias ejemplo1.com
         port 443 namevhost www.ejemplo2.com (/etc/apache2/sites-enabled/ejemplo2.com.conf:16)
                 alias ejemplo2.com
*:80                   is a NameVirtualHost
...
```

**Comprobación:** 

```bash title="Terminal"
curl -I http://ejemplo1.com
curl -I https://ejemplo1.com
curl -I https://www.ejemplo1.com
curl -sI https://www.ejemplo2.com | head -1
```

**Resultado esperado:** 

```text title="Salida esperada"
HTTP/1.1 301 Moved Permanently
Location: https://www.ejemplo1.com/
...
HTTP/2 301
location: https://www.ejemplo1.com/
...
HTTP/2 200
strict-transport-security: max-age=15552000
...
HTTP/2 200
```

En el navegador, `http://ejemplo1.com` debe acabar en `https://www.ejemplo1.com` con el candado cerrado.

**Errores habituales:** 

- `SSLCertificateFile: file '/etc/letsencrypt/live/...' does not exist or is empty`: la ruta no coincide con la que dio Certbot (consulta `sudo certbot certificates`).
- `Invalid command 'RewriteEngine'` o `Invalid command 'Header'`: falta `a2enmod rewrite headers`.
- `Invalid command 'SSLEngine'`: falta `a2enmod ssl`.
- Bucle de redirecciones (`ERR_TOO_MANY_REDIRECTS`): tienes un proxy o CDN (por ejemplo Cloudflare en modo «Flexible») que habla con tu servidor por HTTP. Cambia el modo a «Full (strict)».

**Siguiente paso:** Módulo 4, la base de datos MariaDB.

## 3.7 Ejercicios prácticos del Módulo 3

1. Deshabilita `ejemplo2.com` con `a2dissite`, recarga y visita `https://www.ejemplo2.com`. ¿Qué web aparece y por qué? Vuelve a habilitarlo.
2. Crea un tercer sitio `pruebas.ejemplo1.com` (usuario `web3`, carpeta propia, solo HTTP) y verifica con `curl -H "Host: ..."` que funciona. Después elimínalo por completo (sitio, carpeta y usuario).
3. Usa `namei -l` para encontrar por qué un archivo con permisos `600` da un error 403.
4. Revisa `/var/log/apache2/ejemplo1.com-access.log` y localiza las peticiones de validación de Let's Encrypt (`acme-challenge`).
5. Comprueba con `openssl s_client -connect www.ejemplo2.com:443 -servername www.ejemplo2.com </dev/null | head -20` qué certificado se presenta. Repite sin `-servername`. ¿Qué cambia y por qué? (Pista: SNI.)
