# Módulo 1. Fundamentos de infraestructura

## Objetivos del módulo

- Entender qué es un servidor Linux y qué diferencia un servidor local de uno público.
- Comprender las direcciones IP, los dominios, el DNS y sus tipos de registro.
- Saber qué son los puertos, TCP y UDP, y cómo se comunican los servicios.
- Conectarte a tu servidor por SSH de forma segura con claves.
- Dejar el sistema actualizado, con nombre, zona horaria y un usuario administrador propio.
- Dominar lo básico de usuarios, grupos y permisos en Linux.

## 1.1 Qué es un servidor Linux

Un **servidor** es un ordenador cuya función es ofrecer servicios a otros ordenadores, llamados **clientes**. Cuando abres `https://www.ejemplo1.com` en tu navegador, tu navegador es el cliente y el ordenador que te entrega la página es el servidor.

Un servidor Linux es simplemente un ordenador con el sistema operativo Linux, normalmente **sin entorno gráfico**: se administra escribiendo comandos en un terminal. No tener escritorio no es una limitación, es una ventaja: consume menos memoria, tiene menos programas que actualizar y menos puntos débiles para un atacante.

Dentro del servidor, cada servicio es un programa que se queda funcionando en segundo plano (un **demonio** o *daemon*) esperando peticiones:

| Servicio | Programa (demonio) | Qué hace |
|---|---|---|
| Web | `apache2` | Entrega páginas web por HTTP/HTTPS. |
| Intérprete PHP | `php8.3-fpm` | Ejecuta el código PHP que Apache le pasa. |
| Base de datos | `mariadbd` | Guarda y entrega datos estructurados. |
| Correo SMTP | `master` (Postfix) | Recibe correo de Internet y envía el de tus usuarios. |
| Buzones | `dovecot` | Permite a los usuarios leer su correo (IMAP/POP3). |
| Firma DKIM | `opendkim` | Firma criptográficamente los correos salientes. |
| Acceso remoto | `sshd` | Te permite administrar el servidor a distancia. |

En Ubuntu, todos estos demonios los arranca, detiene y vigila **systemd**, el gestor de servicios del sistema. La herramienta para hablar con systemd es `systemctl`, que usarás constantemente.

## 1.2 Servidor local y servidor público

| Característica | Servidor local (laboratorio) | Servidor público (producción) |
|---|---|---|
| Dónde está | Tu casa, tu empresa, una máquina virtual. | Un centro de datos (VPS, servidor dedicado, nube). |
| Dirección IP | Privada (192.168.x.x, 10.x.x.x). | Pública, accesible desde todo Internet. |
| Quién puede acceder | Solo los equipos de tu red. | Cualquiera en el mundo (también atacantes). |
| Dominios | Simulados con el archivo `hosts`. | Reales, resueltos por DNS público. |
| Certificados | Autofirmados. | Let's Encrypt u otra autoridad. |
| Correo | Solo entre cuentas internas. | Intercambio con todo Internet. |

!!! warning "Atención"
    Un servidor público empieza a recibir intentos de intrusión **minutos** después de encenderse. Bots automáticos prueban contraseñas en SSH las 24 horas del día. Por eso en este módulo aseguraremos el acceso antes de instalar ningún servicio.

## 1.3 Direcciones IP públicas y privadas

Una **dirección IP** identifica a un equipo en una red, como el número de un portal en una calle. Hay dos versiones:

- **IPv4**: cuatro números de 0 a 255 separados por puntos, por ejemplo `203.0.113.10`. Hay unos 4.300 millones y están prácticamente agotadas.
- **IPv6**: ocho grupos hexadecimales separados por dos puntos, por ejemplo `2001:db8:10::10` (`::` abrevia grupos de ceros). Hay muchísimas más.

Las IP **privadas** solo valen dentro de una red local y no se pueden alcanzar directamente desde Internet. Los rangos reservados son:

| Rango privado IPv4 | Uso típico |
|---|---|
| 10.0.0.0 – 10.255.255.255 (10.0.0.0/8) | Redes de empresa, nubes internas. |
| 172.16.0.0 – 172.31.255.255 (172.16.0.0/12) | Redes de empresa, Docker. |
| 192.168.0.0 – 192.168.255.255 (192.168.0.0/16) | Redes domésticas. |
| 127.0.0.0/8 (sobre todo 127.0.0.1) | *Loopback*: el propio equipo hablándose a sí mismo. |

Una IP **pública** es única en todo Internet. Tu servidor necesita una IP pública para que el mundo pueda llegar a tu web y a tu correo.

La notación `/24`, `/8`, etc. (llamada CIDR) indica cuántos bits de la dirección son fijos. `192.168.1.0/24` significa «todas las direcciones de 192.168.1.0 a 192.168.1.255».

!!! tip "Consejo"
    La dirección `127.0.0.1` (y su equivalente IPv6 `::1`) es muy importante en este manual: los servicios que solo deben ser usados desde dentro del servidor (MariaDB, OpenDKIM) escucharán **solo** en esa dirección, de modo que sean invisibles desde Internet.

## 1.4 Qué es un dominio

Recordar `203.0.113.10` es incómodo; recordar `ejemplo1.com` es fácil. Un **dominio** es un nombre legible que se traduce a una dirección IP.

Los dominios se leen de derecha a izquierda, de lo general a lo particular:

```text title="Diagrama"
          www . ejemplo1 . com .
           │       │        │   └─ raíz (normalmente invisible)
           │       │        └───── dominio de nivel superior (TLD)
           │       └────────────── dominio que tú registras
           └────────────────────── subdominio (lo creas tú, gratis)
```

Tú **alquilas** `ejemplo1.com` a un **registrador** (por un año o varios). A partir de ahí puedes crear todos los subdominios que quieras: `www.ejemplo1.com`, `mail.ejemplo1.com`, `tienda.ejemplo1.com`...

## 1.5 Qué es el DNS

El **DNS** (*Domain Name System*) es la «guía telefónica» de Internet: un sistema distribuido de servidores que responde a preguntas como «¿qué IP tiene www.ejemplo1.com?» o «¿qué servidor recibe el correo de ejemplo1.com?».

Cuando tu navegador visita `www.ejemplo1.com` ocurre esto:

```text title="Diagrama"
 Navegador ──1──► Resolvedor DNS (de tu operador, 1.1.1.1, 8.8.8.8...)
                       │
                       ├──2──► Servidores raíz:      "¿.com?"        → pregunta a los de .com
                       ├──3──► Servidores de .com:   "¿ejemplo1.com?" → pregunta a ns1.tuproveedor
                       └──4──► Servidor autoritativo de ejemplo1.com:
                                "www.ejemplo1.com tiene la IP 203.0.113.10"
 Navegador ◄──5── 203.0.113.10   (la respuesta se guarda en caché durante el TTL)
 Navegador ──6──► Conecta con 203.0.113.10 puerto 443 y pide la página
```

El **servidor autoritativo** es el que tiene la última palabra sobre tu dominio. Normalmente es el de tu registrador o el de un servicio DNS como Cloudflare. Ahí es donde **tú** crearás los registros. En este manual no montaremos un servidor DNS propio (sería un riesgo y una complicación innecesaria): usaremos el panel de tu proveedor DNS.

El **TTL** (*Time To Live*) de cada registro indica cuántos segundos pueden guardar la respuesta en caché los resolvedores. Un TTL de 3600 significa que un cambio puede tardar hasta una hora en verse en todo el mundo. Antes de hacer cambios importantes, conviene bajar el TTL a 300 (5 minutos).

## 1.6 Tipos de registros DNS

| Registro | Para qué sirve | Ejemplo |
|---|---|---|
| **A** | Asocia un nombre a una IPv4. | `www.ejemplo1.com. A 203.0.113.10` |
| **AAAA** | Asocia un nombre a una IPv6. | `www.ejemplo1.com. AAAA 2001:db8:10::10` |
| **CNAME** | Hace que un nombre sea alias de otro nombre. | `blog.ejemplo1.com. CNAME www.ejemplo1.com.` |
| **MX** | Indica qué servidor recibe el correo del dominio, con una prioridad. | `ejemplo1.com. MX 10 mail.ejemplo1.com.` |
| **TXT** | Texto libre; se usa para SPF, DKIM, DMARC y verificaciones. | `ejemplo1.com. TXT "v=spf1 mx -all"` |
| **PTR** | DNS inverso: asocia una IP a un nombre. Lo gestiona el dueño de la IP. | `10.113.0.203.in-addr.arpa. PTR mail.ejemplo1.com.` |
| NS | Indica qué servidores son autoritativos para el dominio. | `ejemplo1.com. NS ns1.proveedor.net.` |
| CAA | Indica qué autoridades pueden emitir certificados para el dominio. | `ejemplo1.com. CAA 0 issue "letsencrypt.org"` |

Detalles importantes:

- El **@** en los paneles DNS significa «el propio dominio» (`ejemplo1.com` sin nada delante).
- Un **CNAME** no puede coexistir con otros registros del mismo nombre, por eso nunca se pone CNAME en `@`.
- Un **MX** debe apuntar a un **nombre** que tenga registro A/AAAA, **nunca** a una IP ni a un CNAME.
- En el **MX**, el número es la prioridad: cuanto más bajo, más preferido. Con un solo servidor, el valor da igual (usaremos 10).
- El **PTR** lo verás en detalle en el Módulo 9. Es crucial para el correo.

## 1.7 Qué es un puerto

La IP lleva el paquete hasta el servidor; el **puerto** indica a qué programa dentro del servidor va dirigido. Si la IP es el portal, el puerto es el número de piso. Hay 65.535 puertos; los inferiores a 1024 están reservados para servicios conocidos y solo root puede abrirlos.

| Puerto | Protocolo | Servicio | ¿Abierto a Internet en este manual? |
|---|---|---|---|
| 22 | TCP | SSH (administración remota) | Sí (protegido con claves y Fail2ban) |
| 25 | TCP | SMTP entre servidores (recepción de correo) | Sí |
| 53 | UDP/TCP | DNS | No (no montamos DNS propio) |
| 80 | TCP | HTTP (web sin cifrar; redirige a 443) | Sí |
| 110 | TCP | POP3 sin cifrar | No (deshabilitado) |
| 143 | TCP | IMAP sin cifrar / STARTTLS | No (deshabilitado) |
| 443 | TCP | HTTPS (web cifrada) | Sí |
| 465 | TCP | SMTPS: envío autenticado con TLS implícito | Sí |
| 587 | TCP | Submission: envío autenticado con STARTTLS | Sí |
| 993 | TCP | IMAPS: IMAP cifrado | Sí |
| 995 | TCP | POP3S: POP3 cifrado | Sí |
| 3306 | TCP | MariaDB | **No, nunca** (solo 127.0.0.1) |
| 8891 | TCP | OpenDKIM (milter) | **No** (solo 127.0.0.1) |

## 1.8 TCP y UDP

Ambos son protocolos de transporte: la forma en que viajan los datos entre dos puertos.

- **TCP** (*Transmission Control Protocol*) es orientado a conexión. Antes de enviar datos, cliente y servidor se saludan (el «saludo en tres pasos»: SYN, SYN-ACK, ACK). TCP garantiza que los datos llegan completos y en orden, y reenvía lo que se pierda. Lo usan la web, el correo, SSH y MariaDB.
- **UDP** (*User Datagram Protocol*) envía paquetes sueltos sin conexión ni garantías. Es más rápido y ligero. Lo usan el DNS (consultas cortas), la voz sobre IP, los juegos y el streaming.

Todos los servicios de este manual usan TCP, salvo las consultas DNS que tu servidor hace hacia fuera.

## 1.9 Cómo se comunican los servicios

Los servicios hablan entre sí de dos maneras:

1. **Por red (TCP/IP)**: un servicio escucha en `IP:puerto` y otro se conecta. Ejemplo: Postfix se conecta a OpenDKIM en `127.0.0.1:8891`.
2. **Por socket Unix**: un archivo especial del sistema por el que dos programas del **mismo** servidor se comunican sin pasar por la red. Es más rápido y se protege con permisos de archivo. Ejemplos: Apache habla con PHP-FPM por `/run/php/php8.3-fpm-web1.sock`; PHP habla con MariaDB por `/run/mysqld/mysqld.sock`; Postfix entrega el correo a Dovecot por `/var/spool/postfix/private/dovecot-lmtp`.

Esta es la infraestructura que vamos a construir:

```text title="Diagrama"
                               INTERNET
                                   │
        ┌──────────────────────────┼───────────────────────────┐
        │  Cortafuegos UFW: abre 22, 25, 80, 443, 465, 587, 993, 995  │
        └──────────────────────────┼───────────────────────────┘
                                   │
   ┌───────────────── SERVIDOR mail.ejemplo1.com (203.0.113.10) ──────────────────┐
   │                                                                              │
   │   :80 / :443                          :25 / :465 / :587       :993 / :995    │
   │  ┌──────────┐                         ┌──────────┐           ┌──────────┐    │
   │  │  APACHE  │                         │ POSTFIX  │──LMTP────►│ DOVECOT  │    │
   │  │ vhost 1  │                         │  (SMTP)  │◄──SASL────│(IMAP/POP)│    │
   │  │ vhost 2  │                         └────┬─────┘  socket   └────┬─────┘    │
   │  └────┬─────┘                              │ milter               │          │
   │       │ socket FastCGI                ┌────▼─────┐          ┌─────▼──────┐   │
   │  ┌────▼─────────────────┐             │ OPENDKIM │          │  Buzones   │   │
   │  │ PHP-FPM pool web1    │──sendmail──►│ :8891    │          │  Maildir   │   │
   │  │ PHP-FPM pool web2    │  (correo    │ (local)  │          │ /var/mail/ │   │
   │  └────┬─────────────────┘   de la     └──────────┘          │  vhosts/   │   │
   │       │ socket Unix         web)                            └────────────┘   │
   │  ┌────▼─────────────────┐                                                    │
   │  │ MARIADB (127.0.0.1)  │   web1_db ◄─ web1_user    web2_db ◄─ web2_user     │
   │  └──────────────────────┘                                                    │
   │                                                                              │
   │   Certbot (Let's Encrypt) → certificados para Apache, Postfix y Dovecot      │
   │   Fail2ban → vigila logs de SSH, Postfix y Dovecot y bloquea atacantes       │
   └──────────────────────────────────────────────────────────────────────────────┘
```

## 1.10 Qué es SSH

**SSH** (*Secure Shell*) es el protocolo que te permite abrir un terminal en el servidor desde tu ordenador, con todo el tráfico cifrado. En el servidor funciona el demonio `sshd` (puerto 22); en tu ordenador usas un cliente `ssh`.

SSH admite dos formas de autenticarte:

- **Contraseña**: cómoda, pero vulnerable a ataques de fuerza bruta.
- **Par de claves**: generas en tu ordenador una clave **privada** (que nunca sale de tu equipo) y una **pública** (que copias al servidor). El servidor comprueba matemáticamente que tienes la privada sin que esta viaje nunca por la red. Es muchísimo más segura.

Tu objetivo en este módulo: entrar por primera vez con contraseña, instalar tu clave pública y, después, desactivar el acceso por contraseña.

### PASO 1.1 – Conectarte al servidor por primera vez

**Objetivo:** abrir un terminal remoto en el servidor.

**Explicación:** tu proveedor te habrá dado una IP, un usuario (a menudo `root` o `ubuntu`) y una contraseña o una clave. Windows 10/11, macOS y Linux incluyen el cliente `ssh` de serie. En Windows ábrelo desde «Terminal» o «PowerShell».

**Comando:** escríbelo en **tu ordenador** (no en el servidor):

```bash title="Terminal"
ssh root@203.0.113.10
```

**Análisis del comando:** 

- `ssh`: el programa cliente de SSH.
- `root`: el usuario con el que entras. Si tu proveedor te dio `ubuntu`, escribe `ubuntu@203.0.113.10`.
- `@203.0.113.10`: la IP de tu servidor (sustitúyela por la tuya).

**Resultado esperado:** la primera vez verás un aviso de huella digital:

```text title="Salida esperada"
The authenticity of host '203.0.113.10 (203.0.113.10)' can't be established.
ED25519 key fingerprint is SHA256:k3Zr6c0f0yP1G0qkU8JxwO7V4p9k2aQ1Zb8s5m3Lq2E.
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

Escribe `yes` y pulsa Intro. Después introduce la contraseña (no verás nada mientras escribes: es normal). Llegarás a un indicador similar a `root@nombre-servidor:~#`.

**Comprobación:** ejecuta `whoami` (debe responder `root` o tu usuario) y `hostname` (el nombre actual del servidor).

**Errores habituales:** 

- `Connection timed out`: la IP es incorrecta, el servidor está apagado o un cortafuegos del proveedor bloquea el puerto 22. Revisa el panel del proveedor.
- `Permission denied (publickey)`: el servidor no admite contraseñas, solo claves. Usa la clave que te dio el proveedor: `ssh -i ruta/a/clave root@IP`.
- `WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!`: la huella del servidor cambió (por ejemplo, reinstalaste el sistema). Si sabes por qué, borra la huella antigua con `ssh-keygen -R 203.0.113.10` y reconecta. Si no sabes por qué, **no continúes**: podría ser un ataque.

**Siguiente paso:** actualizar el sistema.

## 1.11 Preparar el sistema

### PASO 1.2 – Actualizar el sistema

**Objetivo:** instalar todas las correcciones de seguridad publicadas desde que se creó la imagen del servidor.

**Explicación:** Ubuntu instala software con **APT** a partir de repositorios oficiales. `apt update` descarga la lista actualizada de paquetes disponibles (no instala nada); `apt full-upgrade` instala las versiones nuevas, incluyendo las que necesitan añadir o quitar dependencias.

**Comando:** 

```bash title="Terminal"
sudo apt update
sudo apt full-upgrade -y
```

**Análisis del comando:** 

- `sudo`: ejecuta como administrador (si ya eres root no hace falta, pero no molesta).
- `apt update`: refresca el índice de paquetes.
- `apt full-upgrade`: actualiza todo lo instalado.
- `-y`: responde «sí» automáticamente a la confirmación.

**Resultado esperado:** una lista de paquetes descargados e instalados que termina sin errores. Si aparece una pantalla morada preguntando qué hacer con un archivo de configuración modificado, elige la opción por defecto («keep the local version currently installed»).

**Comprobación:** 

```bash title="Terminal"
apt list --upgradable
ls /var/run/reboot-required 2>/dev/null && echo "Hay que reiniciar"
```

La primera orden no debe listar nada. Si la segunda muestra «Hay que reiniciar», ejecuta `sudo reboot`, espera un minuto y vuelve a conectarte.

**Errores habituales:** 

- `Could not get lock /var/lib/dpkg/lock-frontend`: otro proceso (normalmente las actualizaciones automáticas al arrancar) está usando APT. Espera unos minutos y repite.
- `Temporary failure resolving 'archive.ubuntu.com'`: el servidor no tiene DNS o red. Comprueba con `ping -c 3 1.1.1.1` y `resolvectl status`.

**Siguiente paso:** dar al servidor su nombre definitivo.

### PASO 1.3 – Configurar el nombre del servidor (hostname)

**Objetivo:** que el servidor se llame `mail.ejemplo1.com` y sepa resolver su propio nombre.

**Explicación:** el nombre del servidor es fundamental para el correo: Postfix lo usa para presentarse ante otros servidores, y debe coincidir con el registro A y con el DNS inverso (PTR). Hay dos partes: el nombre corto (`mail`) y el nombre completo o **FQDN** (`mail.ejemplo1.com`).

**Comando:** 

```bash title="Terminal"
sudo hostnamectl set-hostname mail.ejemplo1.com
```

**Análisis del comando:** `hostnamectl` es la herramienta de systemd para gestionar el nombre; `set-hostname` lo fija de forma permanente (lo guarda en `/etc/hostname`).

Ahora edita el archivo `/etc/hosts`, que el sistema consulta antes que el DNS:

```bash title="Terminal"
sudo nano /etc/hosts
```

`nano` es un editor de texto sencillo. Te mueves con las flechas, escribes normalmente, guardas con **Ctrl+O** seguido de Intro y sales con **Ctrl+X**.

**Archivo:** `/etc/hosts`. Deja su contenido así (conserva las líneas IPv6 que ya hubiera debajo):

```text title="Archivo de configuración"
127.0.0.1       localhost
203.0.113.10    mail.ejemplo1.com mail

# Las siguientes líneas son las de IPv6 que trae el sistema; consérvalas
::1             localhost ip6-localhost ip6-loopback
ff02::1         ip6-allnodes
ff02::2         ip6-allrouters
```

!!! warning "Atención"
    Si tu proveedor usa **cloud-init**, puede sobrescribir `/etc/hosts` al reiniciar. Si al reiniciar tus cambios desaparecen, edita `/etc/cloud/cloud.cfg` y cambia `manage_etc_hosts: true` por `manage_etc_hosts: false` (o añade la línea `preserve_hostname: true`).

**Comprobación:** 

```bash title="Terminal"
hostname
hostname -f
```

**Resultado esperado:** 

```text title="Salida esperada"
mail.ejemplo1.com
mail.ejemplo1.com
```

**Errores habituales:** si `hostname -f` muestra solo `mail` o da error, revisa que la línea de `/etc/hosts` tenga primero el FQDN y luego el nombre corto.

**Siguiente paso:** configurar la hora.

### PASO 1.4 – Configurar la zona horaria y la sincronización de hora

**Objetivo:** que los registros (logs) muestren tu hora local y que el reloj sea exacto.

**Explicación:** una hora incorrecta rompe los certificados TLS, confunde los logs y hace que otros servidores desconfíen de tus correos. Ubuntu sincroniza el reloj con `systemd-timesyncd`.

**Comando:** 

```bash title="Terminal"
sudo timedatectl set-timezone Europe/Madrid
timedatectl
```

**Análisis del comando:** `set-timezone` fija la zona horaria (lista todas con `timedatectl list-timezones`; por ejemplo `America/Mexico_City` o `America/Argentina/Buenos_Aires`).

**Resultado esperado:** 

```text title="Salida esperada"
               Local time: dom 2026-11-08 21:40:12 CET
           Universal time: dom 2026-11-08 20:40:12 UTC
                Time zone: Europe/Madrid (CET, +0100)
System clock synchronized: yes
              NTP service: active
```

**Comprobación:** las líneas `System clock synchronized: yes` y `NTP service: active` deben aparecer así.

**Errores habituales:** si `NTP service` aparece `inactive`, actívalo con `sudo timedatectl set-ntp true`.

**Siguiente paso:** crear tu usuario administrador.

## 1.12 Usuarios administrativos

Trabajar siempre como `root` es peligroso: un error de escritura puede destruir el sistema y, si alguien roba la sesión, tiene control total. La práctica profesional es usar un usuario normal que pide privilegios **solo cuando los necesita**, mediante `sudo`. Así cada acción administrativa queda registrada en `/var/log/auth.log`.

### PASO 1.5 – Crear el usuario administrador

**Objetivo:** crear el usuario `sysadmin` con permiso para usar `sudo`.

**Comando (ejecútalo como root o con sudo):** 

```bash title="Terminal"
sudo adduser sysadmin
sudo usermod -aG sudo sysadmin
```

**Análisis del comando:** 

- `adduser sysadmin`: crea el usuario, su grupo del mismo nombre y su carpeta `/home/sysadmin`. Te pedirá una contraseña (elige una larga) y datos opcionales (pulsa Intro para dejarlos vacíos).
- `usermod -aG sudo sysadmin`: añade (`-a`, *append*) el usuario al grupo (`-G`) `sudo`. En Ubuntu, los miembros de este grupo pueden usar `sudo`. Sin la `-a` el usuario sería **expulsado** de sus otros grupos: no la olvides.

**Comprobación:** 

```bash title="Terminal"
groups sysadmin
```

```text title="Salida esperada"
sysadmin : sysadmin sudo users
```

(`users` puede aparecer o no; lo importante es `sudo`.)

Ahora, **sin cerrar la sesión actual**, abre otro terminal en tu ordenador y entra con el nuevo usuario:

```bash title="Terminal"
ssh sysadmin@203.0.113.10
sudo whoami
```

`sudo whoami` debe pedirte la contraseña de `sysadmin` y responder `root`.

**Errores habituales:** `sysadmin is not in the sudoers file` significa que el `usermod` no se aplicó; repítelo y vuelve a iniciar sesión (los grupos se cargan al iniciar sesión).

**Siguiente paso:** configurar el acceso por clave.

### PASO 1.6 – Crear un par de claves SSH y copiar la pública al servidor

**Objetivo:** entrar al servidor sin contraseña, con una clave criptográfica.

**Explicación:** generarás las claves en **tu ordenador**. El algoritmo Ed25519 es moderno, rápido y seguro. La clave privada se puede proteger con una frase de paso: si alguien te roba el archivo, no podrá usarlo sin ella.

**Comando (en **tu ordenador**, Windows, macOS o Linux):** 

```bash title="Terminal"
ssh-keygen -t ed25519 -C "sysadmin@portatil"
```

**Análisis del comando:** 

- `-t ed25519`: tipo de clave.
- `-C "..."`: comentario para identificarla (por ejemplo, desde qué equipo se usa).

Acepta la ruta por defecto pulsando Intro y escribe una frase de paso. Se crean dos archivos en la carpeta `.ssh` de tu usuario: `id_ed25519` (privada, **no la compartas nunca**) y `id_ed25519.pub` (pública).

Para copiar la clave pública al servidor desde **macOS o Linux**:

```bash title="Terminal"
ssh-copy-id sysadmin@203.0.113.10
```

Desde **Windows (PowerShell)**, que no incluye `ssh-copy-id`:

```powershell title="PowerShell (Windows)"
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh sysadmin@203.0.113.10 "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

Análisis del comando de Windows: `type` muestra el contenido de la clave pública; `|` lo envía como entrada al comando remoto, que crea la carpeta `~/.ssh` con permisos `700` (solo el dueño), añade la clave al final de `authorized_keys` y deja ese archivo con permisos `600`. SSH **rechaza** las claves si estos permisos son más abiertos.

**Comprobación:** cierra la sesión y vuelve a entrar con `ssh sysadmin@203.0.113.10`. Ahora te pedirá la **frase de paso de la clave** (o nada, si no pusiste) en lugar de la contraseña del servidor.

**Errores habituales:** si sigue pidiendo la contraseña del servidor, en el servidor revisa permisos con `ls -ld ~/.ssh && ls -l ~/.ssh/authorized_keys` (deben ser `drwx------` y `-rw-------`) y que el propietario sea `sysadmin`.

**Siguiente paso:** endurecer la configuración de SSH.

### PASO 1.7 – Desactivar el acceso de root y por contraseña

**Objetivo:** que solo se pueda entrar con clave y nunca directamente como root.

**Explicación:** en Ubuntu 24.04, el archivo principal `/etc/ssh/sshd_config` incluye al principio todos los archivos de `/etc/ssh/sshd_config.d/*.conf` por orden alfabético, y **para cada opción gana el primer valor que se lee**. Algunos proveedores dejan un archivo `50-cloud-init.conf` con `PasswordAuthentication yes`. Por eso nuestro archivo debe empezar por `00-`, para leerse antes.

!!! warning "Atención"
    **Antes de este paso**, confirma que puedes entrar como `sysadmin` con tu clave y usar `sudo`. Mantén abierta una sesión mientras haces el cambio: si algo sale mal, podrás corregirlo desde ella.

**Comando:** 

```bash title="Terminal"
sudo nano /etc/ssh/sshd_config.d/00-endurecimiento.conf
```

**Archivo:** `/etc/ssh/sshd_config.d/00-endurecimiento.conf` (archivo nuevo, contenido completo):

```text title="Archivo de configuración"
# Endurecimiento de SSH - se lee antes que el resto de archivos .conf
# Prohíbe entrar directamente como root
PermitRootLogin no
# Solo autenticación por clave pública
PubkeyAuthentication yes
PasswordAuthentication no
KbdInteractiveAuthentication no
# Número máximo de intentos por conexión
MaxAuthTries 3
# Tiempo máximo para completar el inicio de sesión
LoginGraceTime 30
# Solo este usuario puede entrar por SSH (añade otros separados por espacios)
AllowUsers sysadmin
# Sin reenvío gráfico
X11Forwarding no
```

Comprueba la sintaxis y aplica:

```bash title="Terminal"
sudo sshd -t
sudo systemctl restart ssh
```

**Análisis del comando:** `sshd -t` (*test*) revisa la configuración sin aplicarla; si no muestra nada, está bien. `systemctl restart ssh` reinicia el servicio (en Ubuntu se llama `ssh`, no `sshd`).

**Comprobación:** desde **otro** terminal de tu ordenador:

```bash title="Terminal"
ssh root@203.0.113.10
ssh -o PubkeyAuthentication=no sysadmin@203.0.113.10
ssh sysadmin@203.0.113.10
```

Las dos primeras deben fallar con `Permission denied`; la tercera debe funcionar. También puedes consultar el valor efectivo de cada opción en el servidor:

```bash title="Terminal"
sudo sshd -T | grep -Ei 'permitrootlogin|passwordauthentication'
```

```text title="Salida esperada"
permitrootlogin no
passwordauthentication no
```

**Errores habituales:** si `sshd -t` muestra `Bad configuration option`, hay una errata en el archivo. Si te quedas fuera, usa la **consola web** del proveedor (todos la ofrecen) para entrar y borrar el archivo.

**Siguiente paso:** entender usuarios, grupos y permisos.

## 1.13 Permisos y privilegios en Linux

Cada archivo y carpeta tiene un **propietario** (un usuario), un **grupo** y tres juegos de permisos: para el propietario (*u*), para el grupo (*g*) y para el resto (*o*). Cada juego puede tener lectura (**r**), escritura (**w**) y ejecución (**x**).

```bash title="Terminal"
ls -l /etc/passwd
```

```text title="Salida esperada"
-rw-r--r-- 1 root root 1932 nov  8 21:45 /etc/passwd
```

Cómo se lee `-rw-r--r--`:

```text title="Diagrama"
  -    rw-    r--    r--
  │     │      │      └── otros: solo lectura
  │     │      └───────── grupo (root): solo lectura
  │     └──────────────── propietario (root): lectura y escritura
  └────────────────────── tipo: - archivo, d directorio, l enlace
```

En una **carpeta**, `r` permite listar su contenido, `w` crear o borrar archivos dentro y `x` entrar en ella (atravesarla). Sin `x` en una carpeta no puedes acceder a nada de lo que contiene, aunque tengas permiso sobre los archivos.

Los permisos se expresan también en **octal**: r = 4, w = 2, x = 1, y se suman por cada juego.

| Octal | Simbólico | Uso típico en este manual |
|---|---|---|
| 755 | rwxr-xr-x | Carpetas públicas genéricas. |
| 750 | rwxr-x--- | Carpetas web: dueño todo, grupo (www-data) entra y lee, otros nada. |
| 700 | rwx------ | Carpetas privadas (~/.ssh, configuración con contraseñas). |
| 644 | rw-r--r-- | Archivos de lectura pública. |
| 640 | rw-r----- | Archivos web: dueño escribe, grupo lee, otros nada. |
| 600 | rw------- | Secretos: claves privadas, archivos con contraseñas. |

Comandos esenciales:

| Comando | Qué hace | Ejemplo |
|---|---|---|
| `chmod` | Cambia permisos. | `chmod 640 index.php` |
| `chown` | Cambia propietario y grupo. | `sudo chown web1:www-data index.php` |
| `chown -R` / `chmod -R` | Lo mismo, recursivamente. | `sudo chown -R web1:www-data /var/www/ejemplo1.com` |
| `id` | Muestra UID, GID y grupos de un usuario. | `id web1` |
| `useradd` / `adduser` | Crea usuarios (bajo nivel / interactivo). | `sudo adduser sysadmin` |
| `usermod -aG` | Añade un usuario a un grupo. | `sudo usermod -aG www-data sysadmin` |
| `sudo -u` | Ejecuta un comando como otro usuario. | `sudo -u web1 ls /var/www/ejemplo1.com` |

!!! tip "Consejo"
    **Principio de mínimo privilegio:** cada programa y cada persona debe tener exactamente los permisos que necesita y ninguno más. Es la idea que guía todo el diseño de este manual: cada web tendrá su propio usuario Linux, su propio usuario de base de datos y su propio proceso PHP, de modo que si una web es atacada, el atacante no pueda tocar la otra.

### PASO 1.8 – Instalar herramientas básicas de administración

**Objetivo:** disponer de las utilidades que usaremos en los diagnósticos.

**Comando:** 

```bash title="Terminal"
sudo apt install -y curl wget dnsutils net-tools htop unzip ca-certificates rsyslog
```

**Análisis del comando:** `curl` y `wget` hacen peticiones web; `dnsutils` aporta `dig` para consultar el DNS; `net-tools` aporta comandos clásicos de red; `htop` muestra procesos y consumo; `unzip` descomprime; `ca-certificates` contiene las autoridades de certificación de confianza; `rsyslog` escribe los logs clásicos en `/var/log` (`mail.log`, `auth.log`), que Fail2ban leerá más adelante (en Ubuntu 24.04 ya suele venir instalado).

**Comprobación:** 

```bash title="Terminal"
dig +short ubuntu.com
ls /var/log/auth.log
```

Debe mostrar una o varias IP y existir el archivo `auth.log`.

**Siguiente paso:** activar un cortafuegos mínimo antes de exponer servicios.

### PASO 1.9 – Activar el cortafuegos básico

**Objetivo:** bloquear todo el tráfico entrante salvo SSH. Iremos abriendo puertos a medida que instalemos servicios (el Módulo 12 lo trata en profundidad).

**Comando:** 

```bash title="Terminal"
sudo ufw allow OpenSSH
sudo ufw enable
sudo ufw status verbose
```

**Análisis del comando:** `ufw allow OpenSSH` permite el perfil de aplicación de SSH (puerto 22/tcp). `ufw enable` activa el cortafuegos (te avisará de que puede cortar conexiones SSH; responde `y`: ya has permitido SSH). `status verbose` muestra las reglas.

**Resultado esperado:** 

```text title="Salida esperada"
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), deny (routed)
New profiles: skip

To                         Action      From
--                         ------      ----
22/tcp (OpenSSH)           ALLOW IN    Anywhere
22/tcp (OpenSSH (v6))      ALLOW IN    Anywhere (v6)
```

!!! warning "Atención"
    **Nunca** ejecutes `ufw enable` sin haber permitido antes SSH: te quedarías fuera del servidor.

**Siguiente paso:** Módulo 2, instalación de Apache.

## 1.14 Ejercicios prácticos del Módulo 1

1. Consulta con `dig` los registros A, MX y TXT de un dominio conocido (`dig A google.com +short`, `dig MX google.com +short`, `dig TXT google.com +short`) y explica qué significa cada respuesta.
2. Averigua qué puertos están escuchando en tu servidor con `sudo ss -tlnp`. Identifica el proceso de cada uno.
3. Crea un usuario `prueba`, crea en su carpeta un archivo y ponle permisos `600`. Comprueba con `sudo -u sysadmin cat /home/prueba/archivo` que otro usuario no puede leerlo. Después elimina el usuario con `sudo deluser --remove-home prueba`.
4. Explica con tus palabras por qué un MX no puede apuntar a una IP y por qué el nombre del servidor debe coincidir con su PTR.
5. Calcula los permisos octales de `rwxr-x---` y de `rw-r-----`.
