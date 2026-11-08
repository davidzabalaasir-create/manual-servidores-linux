# Módulo 7. Instalación y configuración de Dovecot

## Objetivos del módulo

- Comprender IMAP y POP3 y el papel de Dovecot en la infraestructura de correo.
- Instalar Dovecot 2.3 y conocer su estructura de configuración.
- Preparar el almacenamiento de buzones en formato Maildir con un usuario técnico `vmail`.
- Configurar la autenticación de usuarios virtuales, los protocolos, los sockets y TLS.
- Integrar Dovecot con Postfix en ambos sentidos: entrega de correo (LMTP) y autenticación de envío (SASL).
- Activar el envío autenticado en los puertos 587 y 465 y probar el acceso a los buzones.

## 7.1 Qué son IMAP y POP3

Postfix sabe recibir y enviar correo, pero no permite a los usuarios **leerlo**. Para eso existen dos protocolos:

| Característica | IMAP | POP3 |
|---|---|---|
| Dónde quedan los mensajes | En el servidor. El cliente muestra una vista sincronizada. | Se descargan al dispositivo y, normalmente, se borran del servidor. |
| Carpetas | Sí (Enviados, Borradores, Papelera, carpetas propias). | No, solo la bandeja de entrada. |
| Varios dispositivos | Perfecto: móvil, portátil y webmail ven lo mismo. | Problemático: cada dispositivo descarga su copia. |
| Estado (leído, marcado) | Sincronizado. | No. |
| Recomendación | **Usar IMAP.** | Solo para casos concretos (un único equipo, poco espacio en el servidor). |

**Dovecot** es el servidor IMAP/POP3 más utilizado. En nuestra infraestructura tiene **tres** funciones:

1. **MAA**: atiende a los clientes de correo por IMAP y POP3.
2. **MDA**: recibe de Postfix los mensajes por **LMTP** y los guarda en el buzón correcto.
3. **Autenticador**: Postfix le pregunta, a través de un socket **SASL**, si el usuario y la contraseña de quien quiere enviar son correctos. Así hay una única base de usuarios para todo.

```text title="Diagrama"
                    ┌──────────────────────── DOVECOT ──────────────────────────┐
 Postfix ──LMTP────►│ lmtp ──► userdb (/etc/dovecot/users) ──► escribe Maildir  │
 (entrega)          │                                                           │
 Postfix ──SASL────►│ auth ──► passdb (/etc/dovecot/users) ──► "OK" / "NO"      │
 (¿clave válida?)   │                                                           │
 Cliente ──993─────►│ imap-login ──► auth ──► imap ──► lee Maildir              │
 Cliente ──995─────►│ pop3-login ──► auth ──► pop3 ──► lee Maildir              │
                    └───────────────────────────────────────────────────────────┘
                                                │
                     /var/mail/vhosts/ejemplo1.com/info/Maildir/  (propietario vmail)
```

## 7.2 Puertos de IMAP y POP3

| Puerto | Protocolo | Cifrado | Estado en nuestro servidor |
|---|---|---|---|
| 143 | IMAP | Texto plano con posibilidad de STARTTLS. | **Deshabilitado.** |
| **993** | IMAPS | TLS implícito desde el inicio. | **Habilitado y abierto.** |
| 110 | POP3 | Texto plano con posibilidad de STLS. | **Deshabilitado.** |
| **995** | POP3S | TLS implícito desde el inicio. | **Habilitado y abierto.** |

Los puertos 143 y 110 podrían usarse de forma segura obligando a STARTTLS, pero no aportan nada que no den 993 y 995, y un error de configuración podría hacer que alguien enviara su contraseña sin cifrar. La configuración profesional más simple y robusta es: **solo 993 y 995**, y si no necesitas POP3, ni siquiera eso.

## 7.3 Maildir: dónde y cómo se guardan los mensajes

Hay dos formatos clásicos de buzón:

- **mbox**: todos los mensajes de una carpeta en un único archivo. Simple, pero lento y frágil (un fallo puede corromper el buzón entero).
- **Maildir**: cada mensaje es un archivo independiente. Rápido, seguro ante fallos y fácil de copiar. **Es el que usaremos.**

Estructura de un Maildir:

```text title="Diagrama"
/var/mail/vhosts/                         ← raíz de todos los buzones (vmail:vmail)
└── ejemplo1.com/                         ← un directorio por dominio
    └── info/                             ← "home" del usuario virtual info@ejemplo1.com
        └── Maildir/
            ├── new/                      ← mensajes recién llegados, no vistos por el cliente
            ├── cur/                      ← mensajes ya vistos por el cliente
            ├── tmp/                      ← escritura temporal durante la entrega
            ├── .Sent/   (cur/ new/ tmp/) ← carpeta Enviados
            ├── .Drafts/                  ← Borradores
            ├── .Trash/                   ← Papelera
            ├── .Junk/                    ← Correo no deseado
            └── dovecot.index*            ← índices de Dovecot (aceleran las búsquedas)
```

Todos los buzones pertenecen a un único usuario técnico de Linux, **vmail**. Las cuentas de correo **no** son cuentas de Linux: no pueden iniciar sesión en el servidor, no tienen contraseña de sistema y solo existen para Dovecot y Postfix. Lo trataremos en detalle en el Módulo 8.

### PASO 7.1 – Instalar Dovecot

**Objetivo:** instalar Dovecot con IMAP, POP3 y LMTP.

**Comando:** 

```bash title="Terminal"
sudo apt install -y dovecot-core dovecot-imapd dovecot-pop3d dovecot-lmtpd
```

**Análisis del comando:** 

| Paquete | Aporta |
|---|---|
| `dovecot-core` | El núcleo: autenticación, gestión de buzones, `doveadm`. |
| `dovecot-imapd` | El servicio IMAP. |
| `dovecot-pop3d` | El servicio POP3 (omítelo si no lo vas a usar). |
| `dovecot-lmtpd` | El servicio LMTP, por el que Postfix entrega el correo. |

**Comprobación:** 

```bash title="Terminal"
dovecot --version
systemctl status dovecot --no-pager
```

```text title="Salida esperada"
2.3.21 (47349e2482)
● dovecot.service - Dovecot IMAP/POP3 email server
     Active: active (running) since ...
```

!!! warning "Atención"
    Si la versión empieza por **2.4**, la configuración de este módulo no es válida tal cual: estás en Debian 13, Ubuntu 26.04 o posterior. Consulta la documentación oficial de Dovecot 2.4 («Upgrading Dovecot CE from 2.3 to 2.4») antes de continuar.

**Siguiente paso:** conocer la estructura de configuración.

## 7.4 Estructura de configuración de Dovecot

```text title="Diagrama"
/etc/dovecot/
├── dovecot.conf                ← archivo principal; incluye conf.d/*.conf
├── conf.d/
│   ├── 10-auth.conf            ← mecanismos de autenticación y qué backend se usa
│   ├── auth-system.conf.ext    ← backend de usuarios del sistema (PAM) – lo desactivaremos
│   ├── auth-passwdfile.conf.ext← backend de archivo de usuarios – lo activaremos
│   ├── 10-mail.conf            ← ubicación y formato de los buzones
│   ├── 10-master.conf          ← servicios, puertos y sockets
│   ├── 10-ssl.conf             ← certificados y TLS
│   ├── 15-mailboxes.conf       ← carpetas especiales (Enviados, Papelera...)
│   ├── 20-imap.conf / 20-pop3.conf / 20-lmtp.conf ← opciones por protocolo
│   └── ...
└── users                       ← (lo crearemos) usuarios virtuales y contraseñas
/usr/share/dovecot/protocols.d/ ← un archivo por protocolo instalado (imap, pop3, lmtp)
```

Comandos de administración:

| Comando | Qué hace |
|---|---|
| `sudo doveconf -n` | Muestra la configuración efectiva que difiere de los valores por defecto. Es lo primero que se pide en cualquier foro de ayuda. |
| `sudo doveconf protocols` | Muestra un parámetro concreto. |
| `sudo doveadm reload` | Recarga la configuración. |
| `sudo doveadm user usuario` | Muestra cómo ve Dovecot a un usuario (uid, home, buzón). |
| `sudo doveadm auth test usuario` | Prueba una contraseña. |
| `sudo doveadm pw -s SHA512-CRYPT` | Genera el hash de una contraseña. |
| `sudo doveadm mailbox list -u usuario` | Lista las carpetas de un buzón. |
| `sudo doveadm who` | Muestra quién está conectado. |

Comprueba qué protocolos están activos (los activa automáticamente cada paquete instalado):

```bash title="Terminal"
sudo doveconf protocols
```

```text title="Salida esperada"
protocols = imap pop3 lmtp
```

### PASO 7.2 – Crear el usuario técnico vmail y la carpeta de buzones

**Objetivo:** crear el usuario Linux que será propietario de todos los buzones virtuales.

**Comando:** 

```bash title="Terminal"
sudo groupadd --system --gid 5000 vmail
sudo useradd --system --uid 5000 --gid vmail --home-dir /var/mail/vhosts --shell /usr/sbin/nologin vmail
sudo install -d -o vmail -g vmail -m 750 /var/mail/vhosts
id vmail
ls -ld /var/mail/vhosts
```

**Análisis del comando:** 

- `groupadd --gid 5000 vmail`: crea el grupo con el identificador numérico fijo 5000. Usar un número fijo facilita restaurar copias en otro servidor.
- `useradd --uid 5000 ... --shell /usr/sbin/nologin`: crea el usuario sin posibilidad de iniciar sesión.
- `install -d -o vmail -g vmail -m 750`: crea la carpeta raíz de buzones; solo `vmail` (y su grupo) puede entrar.

**Resultado esperado:** 

```text title="Salida esperada"
uid=5000(vmail) gid=5000(vmail) groups=5000(vmail)
drwxr-x--- 2 vmail vmail 4096 nov  9 01:10 /var/mail/vhosts
```

**Errores habituales:** `GID '5000' already exists` o `UID 5000 is not unique`: otro usuario usa ya ese número. Elige otro libre (por ejemplo 5001) y úsalo de forma coherente en todo el manual.

**Siguiente paso:** crear el archivo de usuarios.

### PASO 7.3 – Crear el archivo de usuarios con la primera cuenta

**Objetivo:** crear `/etc/dovecot/users` con la cuenta `info@ejemplo1.com`, que usaremos para probar todo el módulo.

**Explicación:** Dovecot puede leer los usuarios de muchas fuentes (MariaDB, LDAP, archivos). Usaremos un archivo con el formato `passwd-file`, que es sencillo, rápido y suficiente para decenas o cientos de cuentas. Las contraseñas no se guardan en claro, sino como *hash* **SHA512-CRYPT** con sal aleatoria.

Primero genera el hash de la contraseña de la cuenta:

```bash title="Terminal"
sudo doveadm pw -s SHA512-CRYPT
```

Escribe la contraseña dos veces. El resultado es algo así (cada vez diferente, aunque la contraseña sea la misma, por la sal aleatoria):

```text title="Salida esperada"
Enter new password:
Retype new password:
{SHA512-CRYPT}$6$Qm2xY7bQe0nU5d1T$gq1q9...resto del hash...Xh0
```

Copia toda la línea, desde `{SHA512-CRYPT}` hasta el final.

**Archivo:** `/etc/dovecot/users` (nuevo):

```text title="Archivo de configuración"
info@ejemplo1.com:{SHA512-CRYPT}$6$...PEGA-AQUI-EL-HASH...:5000:5000::/var/mail/vhosts/ejemplo1.com/info::
```

Formato de cada línea (campos separados por `:`):

| Posición | Campo | Valor |
|---|---|---|
| 1 | Usuario | `info@ejemplo1.com` (la dirección completa) |
| 2 | Contraseña | El hash con su prefijo de esquema. |
| 3 | UID | `5000` (vmail) |
| 4 | GID | `5000` (vmail) |
| 5 | (GECOS) | Vacío. |
| 6 | Home | Carpeta del usuario: `/var/mail/vhosts/dominio/usuario` |
| 7 | (Shell) | Vacío. |
| 8 | Campos extra | Vacío (aquí se pueden añadir cuotas, por ejemplo). |

Protege el archivo (contiene hashes de contraseñas):

```bash title="Terminal"
sudo chown root:dovecot /etc/dovecot/users
sudo chmod 640 /etc/dovecot/users
```

!!! tip "Consejo"
    Para no tener que hacer esto a mano con cada cuenta, en el Módulo 8 crearemos un script que genera el hash, añade la línea en Dovecot y registra la dirección en Postfix en un único paso.

**Siguiente paso:** configurar la autenticación.

### PASO 7.4 – Configurar la autenticación (10-auth.conf)

**Objetivo:** que Dovecot autentique solo a los usuarios virtuales del archivo, nunca a las cuentas Linux, y solo por canales cifrados.

**Explicación:** por defecto Dovecot autentica a los usuarios del sistema mediante PAM (`auth-system.conf.ext`). Lo desactivamos y activamos el archivo de usuarios (`auth-passwdfile.conf.ext`). Haremos los cambios con `sed` para que sean exactos, y luego los verificaremos.

**Comando:** 

```bash title="Terminal"
sudo cp /etc/dovecot/conf.d/10-auth.conf /etc/dovecot/conf.d/10-auth.conf.original
sudo sed -i 's/^#\?disable_plaintext_auth = .*/disable_plaintext_auth = yes/' /etc/dovecot/conf.d/10-auth.conf
sudo sed -i 's/^auth_mechanisms = .*/auth_mechanisms = plain login/' /etc/dovecot/conf.d/10-auth.conf
sudo sed -i 's/^!include auth-system.conf.ext/#!include auth-system.conf.ext/' /etc/dovecot/conf.d/10-auth.conf
sudo sed -i 's/^#!include auth-passwdfile.conf.ext/!include auth-passwdfile.conf.ext/' /etc/dovecot/conf.d/10-auth.conf
grep -Ev '^\s*(#|$)' /etc/dovecot/conf.d/10-auth.conf
```

**Análisis del comando:** 

- `sed -i 's/buscar/reemplazar/' archivo`: edita el archivo en el sitio (`-i`) sustituyendo el texto. `^` significa «al principio de la línea»; `#\?` significa «una almohadilla opcional».
- El último `grep -Ev '^\s*(#|$)'` muestra solo las líneas activas (ni comentarios ni líneas vacías).

**Resultado esperado (las líneas activas del archivo):** 

```text title="Salida esperada"
disable_plaintext_auth = yes
auth_mechanisms = plain login
!include auth-passwdfile.conf.ext
```

| Directiva | Significado |
|---|---|
| `disable_plaintext_auth = yes` | Rechaza contraseñas si la conexión no está cifrada. |
| `auth_mechanisms = plain login` | Mecanismos aceptados. `PLAIN` es el estándar; `LOGIN` lo necesitan algunos clientes antiguos (Outlook). Ambos envían la contraseña tal cual, por eso solo se permiten **dentro** de TLS. |
| `#!include auth-system.conf.ext` | Comentado: las cuentas Linux no pueden usar el correo. |
| `!include auth-passwdfile.conf.ext` | Activa el backend de archivo. |

**Errores habituales:** si el `grep` final no muestra exactamente esas tres líneas, abre el archivo con `sudo nano /etc/dovecot/conf.d/10-auth.conf` y corrígelo a mano.

**Siguiente paso:** configurar el backend de archivo.

### PASO 7.5 – Configurar el backend de usuarios (auth-passwdfile.conf.ext)

**Archivo:** `/etc/dovecot/conf.d/auth-passwdfile.conf.ext` (sustituye todo el contenido):

```text title="Archivo de configuración"
# Usuarios virtuales definidos en /etc/dovecot/users
# passdb: comprueba la contraseña
passdb {
  driver = passwd-file
  args = scheme=SHA512-CRYPT username_format=%Lu /etc/dovecot/users
}

# userdb: obtiene uid, gid y carpeta (home) del usuario
userdb {
  driver = passwd-file
  args = username_format=%Lu /etc/dovecot/users
}
```

Análisis:

| Elemento | Significado |
|---|---|
| `passdb` | Base de datos de **contraseñas**: se consulta cuando alguien inicia sesión. |
| `userdb` | Base de datos de **usuarios**: indica uid, gid y home. La consultan IMAP, POP3 y LMTP. |
| `driver = passwd-file` | Formato del archivo (similar a `/etc/passwd`). |
| `scheme=SHA512-CRYPT` | Esquema por defecto de las contraseñas (el prefijo `{SHA512-CRYPT}` lo confirma). |
| `username_format=%Lu` | Busca al usuario por su nombre completo (`%u` = `usuario@dominio`) convertido a minúsculas (`L`). Así `Info@Ejemplo1.com` funciona igual. |

### PASO 7.6 – Configurar la ubicación de los buzones (10-mail.conf)

**Objetivo:** indicar que los buzones son Maildir dentro del home de cada usuario.

**Comando:** 

```bash title="Terminal"
sudo cp /etc/dovecot/conf.d/10-mail.conf /etc/dovecot/conf.d/10-mail.conf.original
sudo sed -i 's|^mail_location = .*|mail_location = maildir:~/Maildir|' /etc/dovecot/conf.d/10-mail.conf
grep -E '^(mail_location|mail_privileged_group|first_valid_uid)' /etc/dovecot/conf.d/10-mail.conf
```

**Resultado esperado:** 

```text title="Salida esperada"
mail_location = maildir:~/Maildir
mail_privileged_group = mail
```

Análisis: `maildir:~/Maildir` significa «formato Maildir, en la subcarpeta `Maildir` del home del usuario». Con el home `/var/mail/vhosts/ejemplo1.com/info`, el buzón queda en `/var/mail/vhosts/ejemplo1.com/info/Maildir`. Dovecot crea las carpetas automáticamente con el usuario `vmail` la primera vez que llega un correo o que el usuario inicia sesión. El parámetro `first_valid_uid` (por defecto 500) impide usar uid inferiores; nuestro 5000 cumple.

### PASO 7.7 – Configurar las carpetas especiales (15-mailboxes.conf)

**Objetivo:** que cada buzón nuevo tenga automáticamente las carpetas Enviados, Borradores, Papelera y Spam.

**Archivo:** `/etc/dovecot/conf.d/15-mailboxes.conf` (sustituye todo el contenido):

```text title="Archivo de configuración"
# Carpetas especiales creadas y suscritas automáticamente en cada buzón
namespace inbox {
  mailbox Drafts {
    auto = subscribe
    special_use = \Drafts
  }
  mailbox Junk {
    auto = subscribe
    special_use = \Junk
  }
  mailbox Trash {
    auto = subscribe
    special_use = \Trash
  }
  mailbox Sent {
    auto = subscribe
    special_use = \Sent
  }
  mailbox "Sent Messages" {
    special_use = \Sent
  }
}
```

Análisis: `special_use` comunica a los clientes (RFC 6154) qué carpeta es cada una, de modo que Thunderbird o el móvil guarden los enviados en `Sent` sin configurar nada. `auto = subscribe` crea la carpeta y la suscribe automáticamente.

### PASO 7.8 – Configurar los servicios, puertos y sockets (10-master.conf)

**Objetivo:** definir en qué puertos escucha Dovecot y crear los dos sockets que usará Postfix.

**Explicación:** este es el archivo de integración más importante. Creamos:

- El socket **LMTP** dentro del *chroot* de Postfix (`/var/spool/postfix/private/dovecot-lmtp`), para que Postfix entregue el correo.
- El socket de **autenticación** (`/var/spool/postfix/private/auth`), para que Postfix compruebe contraseñas.

Ambos están en `/var/spool/postfix/private/` porque los procesos de Postfix se ejecutan encerrados en `/var/spool/postfix` y no pueden ver nada fuera de esa carpeta.

**Comando:** 

```bash title="Terminal"
sudo cp /etc/dovecot/conf.d/10-master.conf /etc/dovecot/conf.d/10-master.conf.original
sudo nano /etc/dovecot/conf.d/10-master.conf
```

**Archivo:** `/etc/dovecot/conf.d/10-master.conf` (sustituye todo el contenido):

```text title="Archivo de configuración"
# =====================================================================
#  Servicios de Dovecot - mail.ejemplo1.com
# =====================================================================

# IMAP: solo el puerto cifrado 993. El 143 queda deshabilitado (port = 0)
service imap-login {
  inet_listener imap {
    port = 0
  }
  inet_listener imaps {
    port = 993
    ssl = yes
  }
}

# POP3: solo el puerto cifrado 995. El 110 queda deshabilitado
service pop3-login {
  inet_listener pop3 {
    port = 0
  }
  inet_listener pop3s {
    port = 995
    ssl = yes
  }
}

# LMTP: Postfix entrega aquí el correo entrante (socket dentro del chroot de Postfix)
service lmtp {
  unix_listener /var/spool/postfix/private/dovecot-lmtp {
    mode = 0600
    user = postfix
    group = postfix
  }
}

service imap {
}

service pop3 {
}

# Autenticación
service auth {
  # Postfix pregunta aquí si un usuario y contraseña son válidos (SASL)
  unix_listener /var/spool/postfix/private/auth {
    mode = 0660
    user = postfix
    group = postfix
  }
  # Socket interno para las consultas de usuarios (lo usa LMTP y doveadm)
  unix_listener auth-userdb {
    mode = 0600
    user = vmail
    group = vmail
  }
  # El proceso de autenticación se ejecuta sin privilegios de root
  user = dovecot
}

service auth-worker {
  user = $default_internal_user
}
```

Análisis:

| Bloque | Significado |
|---|---|
| `inet_listener imap { port = 0 }` | Desactiva el puerto 143. |
| `inet_listener imaps { port = 993; ssl = yes }` | IMAP con TLS implícito. |
| `unix_listener /var/spool/postfix/private/dovecot-lmtp` | Socket LMTP. Propietario `postfix`, permisos `0600`: solo Postfix puede usarlo. |
| `unix_listener /var/spool/postfix/private/auth` | Socket SASL para Postfix. |
| `unix_listener auth-userdb` | Socket interno (en `/run/dovecot/`) para consultas de usuarios. |
| `user = dovecot` (en `service auth`) | El proceso que lee `/etc/dovecot/users` es `dovecot`, no root. Por eso el archivo es `root:dovecot 640`. |
| `service auth-worker { user = $default_internal_user }` | Los procesos auxiliares de autenticación tampoco se ejecutan como root (solo serían necesarios como root para leer `/etc/shadow`, que no usamos). |

### PASO 7.9 – Configurar TLS (10-ssl.conf)

**Objetivo:** que Dovecot use el certificado de Let's Encrypt y exija cifrado.

**Comando:** primero comprueba si existe el archivo de parámetros Diffie-Hellman que trae el paquete:

```bash title="Terminal"
ls -l /usr/share/dovecot/dh.pem
sudo cp /etc/dovecot/conf.d/10-ssl.conf /etc/dovecot/conf.d/10-ssl.conf.original
sudo nano /etc/dovecot/conf.d/10-ssl.conf
```

**Archivo:** `/etc/dovecot/conf.d/10-ssl.conf` (sustituye todo el contenido):

```text title="Archivo de configuración"
# TLS obligatorio para todas las conexiones de clientes
ssl = required

# Certificado de Let's Encrypt (el símbolo < indica "leer el contenido del archivo")
ssl_cert = </etc/letsencrypt/live/mail.ejemplo1.com/fullchain.pem
ssl_key = </etc/letsencrypt/live/mail.ejemplo1.com/privkey.pem

# Autoridades de confianza del sistema
ssl_client_ca_dir = /etc/ssl/certs

# Solo TLS 1.2 o superior; el servidor elige el cifrado
ssl_min_protocol = TLSv1.2
ssl_prefer_server_ciphers = yes

# Parámetros Diffie-Hellman (borra esta línea si el ls anterior dio error)
ssl_dh = </usr/share/dovecot/dh.pem
```

Análisis:

| Directiva | Significado |
|---|---|
| `ssl = required` | Ningún cliente puede autenticarse sin TLS. |
| `ssl_cert = <ruta` | El `<` es imprescindible: indica que el valor es el contenido del archivo. Sin él, Dovecot da error. |
| `ssl_min_protocol` | Rechaza TLS 1.0 y 1.1, obsoletos. |
| `ssl_dh` | Parámetros para el intercambio de claves Diffie-Hellman clásico. |

!!! tip "Consejo"
    Dovecot lee los certificados al arrancar, con privilegios de root, así que puede leer `privkey.pem` aunque solo sea legible por root. Cuando Certbot renueve el certificado habrá que recargar Dovecot: lo automatizaremos en el Módulo 10.

### PASO 7.10 – Configurar LMTP (20-lmtp.conf)

**Archivo:** `/etc/dovecot/conf.d/20-lmtp.conf` (sustituye todo el contenido):

```text title="Archivo de configuración"
protocol lmtp {
  # Dirección que aparece como remitente en los rechazos generados por LMTP
  postmaster_address = postmaster@ejemplo1.com
  mail_plugins = $mail_plugins
}
```

### PASO 7.11 – Comprobar la configuración y reiniciar Dovecot

**Comando:** 

```bash title="Terminal"
sudo doveconf -n > /dev/null && echo "Configuración correcta"
sudo systemctl restart dovecot
systemctl status dovecot --no-pager
sudo ss -tlnp | grep dovecot
ls -l /var/spool/postfix/private/ | grep -E 'auth|dovecot'
```

Análisis: `doveconf -n` analiza toda la configuración; si hay un error, lo muestra con el archivo y la línea.

**Resultado esperado:** 

```text title="Salida esperada"
Configuración correcta
● dovecot.service - Dovecot IMAP/POP3 email server
     Active: active (running) since ...
LISTEN 0  100  0.0.0.0:993  0.0.0.0:*  users:(("dovecot",pid=7120,fd=36))
LISTEN 0  100  0.0.0.0:995  0.0.0.0:*  users:(("dovecot",pid=7120,fd=38))
LISTEN 0  100     [::]:993     [::]:*  users:(("dovecot",pid=7120,fd=37))
LISTEN 0  100     [::]:995     [::]:*  users:(("dovecot",pid=7120,fd=39))
srw-rw---- 1 postfix postfix 0 nov  9 01:30 auth
srw------- 1 postfix postfix 0 nov  9 01:30 dovecot-lmtp
```

No deben aparecer los puertos 143 ni 110.

**Comprobación:** consulta cómo ve Dovecot al usuario y prueba su contraseña:

```bash title="Terminal"
sudo doveadm user info@ejemplo1.com
sudo doveadm auth test info@ejemplo1.com
```

```text title="Salida esperada"
field   value
uid     5000
gid     5000
home    /var/mail/vhosts/ejemplo1.com/info
mail    maildir:~/Maildir
Password:
passdb: info@ejemplo1.com auth succeeded
extra fields:
  user=info@ejemplo1.com
```

**Errores habituales:** 

| Mensaje | Causa | Solución |
|---|---|---|
| `doveconf: Fatal: Error in configuration file ... line N` | Errata (llave sin cerrar, directiva mal escrita). | Revisa el archivo y la línea indicados. |
| `ssl_cert: Can't open file ...` | Ruta del certificado incorrecta o falta el `<`. | `sudo ls -l /etc/letsencrypt/live/mail.ejemplo1.com/`. |
| `auth failed` en `doveadm auth test` | Hash mal copiado o archivo con permisos incorrectos. | Regenera el hash; comprueba `root:dovecot 640`. |
| `userdb lookup: user info@ejemplo1.com doesn't exist` | La línea del usuario está mal formada (faltan `:`). | Cuenta los campos: deben ser 8 separados por 7 `:` hasta el final. |
| `Error: passwd-file /etc/dovecot/users: open() failed: Permission denied` | El proceso auth (usuario dovecot) no puede leer el archivo. | `sudo chown root:dovecot /etc/dovecot/users && sudo chmod 640 /etc/dovecot/users`. |

**Siguiente paso:** integrar Postfix con Dovecot.

## 7.5 Integración de Dovecot con Postfix

Ahora Postfix debe saber tres cosas:

1. Que `ejemplo1.com` y `ejemplo2.com` son dominios de **buzones virtuales**.
2. Qué direcciones existen (para rechazar en el acto el correo a direcciones inexistentes, en lugar de aceptarlo y devolverlo después, lo que generaría *backscatter*).
3. Que la entrega se hace por **LMTP** a Dovecot y que la autenticación se pregunta a Dovecot por **SASL**.

### PASO 7.12 – Crear las tablas de dominios y buzones de Postfix

**Archivo:** `/etc/postfix/virtual_domains` (nuevo, contenido completo):

```text title="Archivo de configuración"
# Dominios con buzones virtuales. Formato: dominio  OK
ejemplo1.com    OK
ejemplo2.com    OK
```

**Archivo:** `/etc/postfix/vmailbox` (nuevo; de momento, solo la primera cuenta):

```text title="Archivo de configuración"
# Direcciones que tienen buzón. Formato: dirección  dominio/usuario/
info@ejemplo1.com           ejemplo1.com/info/
```

**Archivo:** `/etc/postfix/virtual` (nuevo; alias de direcciones virtuales):

```text title="Archivo de configuración"
# Alias virtuales. Formato: dirección_alias  destino[, destino2...]
# Direcciones obligatorias por RFC (se completarán en el Módulo 8)
postmaster@ejemplo1.com     info@ejemplo1.com
abuse@ejemplo1.com          info@ejemplo1.com
```

**Comando:** compila las tablas al formato que lee Postfix:

```bash title="Terminal"
sudo postmap /etc/postfix/virtual_domains
sudo postmap /etc/postfix/vmailbox
sudo postmap /etc/postfix/virtual
ls /etc/postfix/*.db
```

Análisis: `postmap` crea un archivo `.db` (base de datos indexada) junto a cada archivo de texto. Postfix consulta el `.db`, no el texto. **Cada vez** que modifiques uno de estos archivos debes volver a ejecutar `postmap` sobre él (no hace falta reiniciar Postfix: detecta el cambio).

!!! tip "Consejo"
    En `vmailbox`, el valor de la derecha (`ejemplo1.com/info/`) no se usa para la entrega, porque la ruta real la decide Dovecot. Postfix solo necesita saber que la dirección **existe**. Lo dejamos escrito así por claridad y por convención.

### PASO 7.13 – Añadir la integración a main.cf

**Objetivo:** activar los buzones virtuales, la entrega por LMTP y la autenticación SASL.

**Comando:** 

```bash title="Terminal"
sudo nano /etc/postfix/main.cf
```

**Archivo:** `/etc/postfix/main.cf`. **Conserva todo** lo que escribiste en el Módulo 6 y **añade al final** este bloque:

```text title="Archivo de configuración"
# --- Buzones virtuales gestionados por Dovecot (Módulo 7) ---------------
virtual_mailbox_domains = hash:/etc/postfix/virtual_domains
virtual_mailbox_maps = hash:/etc/postfix/vmailbox
virtual_alias_maps = hash:/etc/postfix/virtual
virtual_transport = lmtp:unix:private/dovecot-lmtp

# --- Autenticación SASL a través de Dovecot (Módulo 7) -----------------
smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth
# Desactivada en el puerto 25; se activa solo en 587 y 465 (master.cf)
smtpd_sasl_auth_enable = no
smtpd_sasl_security_options = noanonymous, noplaintext
smtpd_sasl_tls_security_options = noanonymous
```

Análisis:

| Parámetro | Significado |
|---|---|
| `virtual_mailbox_domains` | Lista de dominios virtuales (tabla `virtual_domains`). |
| `virtual_mailbox_maps` | Direcciones válidas de esos dominios. Las demás se rechazan con `550 User unknown`. |
| `virtual_alias_maps` | Alias: reescriben el destinatario antes de entregar. |
| `virtual_transport = lmtp:unix:private/dovecot-lmtp` | Entrega por LMTP en el socket que creó Dovecot (ruta relativa a `/var/spool/postfix`). |
| `smtpd_sasl_type = dovecot` | El autenticador es Dovecot. |
| `smtpd_sasl_path = private/auth` | Socket de autenticación (relativo a `/var/spool/postfix`). |
| `smtpd_sasl_auth_enable = no` | En el puerto 25 nadie puede autenticarse: es solo para recibir de otros servidores. |
| `smtpd_sasl_security_options = noanonymous, noplaintext` | Sin TLS no se aceptan mecanismos que envíen la contraseña en claro. |
| `smtpd_sasl_tls_security_options = noanonymous` | Con TLS sí se aceptan PLAIN y LOGIN (la contraseña va cifrada por TLS). |

### PASO 7.14 – Activar los puertos de envío autenticado 587 y 465 (master.cf)

**Objetivo:** abrir los servicios *submission* (587, STARTTLS) y *submissions* (465, TLS implícito), ambos con autenticación obligatoria.

**Explicación:** `master.cf` define los procesos que arranca Postfix. Cada servicio es una línea con 8 columnas, seguida opcionalmente de líneas `-o parametro=valor` (que **deben empezar por espacios**) que sobrescriben `main.cf` solo para ese servicio. Ubuntu trae estos servicios comentados.

**Comando:** 

```bash title="Terminal"
sudo cp /etc/postfix/master.cf /etc/postfix/master.cf.original
sudo nano /etc/postfix/master.cf
```

**Archivo:** `/etc/postfix/master.cf`. Localiza las líneas que empiezan por `#submission inet` y `#smtps     inet` (están comentadas, cerca del principio, justo después de la línea `smtp inet ... smtpd`). **Bórralas junto con sus líneas `#  -o ...`** y escribe en su lugar este bloque. Conserva el resto del archivo sin cambios.

```text title="Archivo de configuración"
submission inet n       -       y       -       -       smtpd
  -o syslog_name=postfix/submission
  -o smtpd_tls_security_level=encrypt
  -o smtpd_sasl_auth_enable=yes
  -o smtpd_tls_auth_only=yes
  -o smtpd_client_restrictions=permit_sasl_authenticated,reject
  -o smtpd_relay_restrictions=permit_sasl_authenticated,reject
  -o milter_macro_daemon_name=ORIGINATING
smtps     inet  n       -       y       -       -       smtpd
  -o syslog_name=postfix/smtps
  -o smtpd_tls_wrappermode=yes
  -o smtpd_sasl_auth_enable=yes
  -o smtpd_client_restrictions=permit_sasl_authenticated,reject
  -o smtpd_relay_restrictions=permit_sasl_authenticated,reject
  -o milter_macro_daemon_name=ORIGINATING
```

Análisis de las columnas de la primera línea (`submission inet n - y - - smtpd`):

| Columna | Valor | Significado |
|---|---|---|
| Servicio | `submission` | Nombre del puerto en `/etc/services` (587). `smtps` es el 465. |
| Tipo | `inet` | Escucha en red TCP. |
| Privado | `n` | No es un servicio interno. |
| Sin privilegios | `-` | Se ejecuta con el usuario sin privilegios de Postfix. |
| Chroot | `y` | Se ejecuta encerrado en `/var/spool/postfix` (por defecto en Ubuntu). |
| Despertar / Máx. procesos | `-` `-` | Valores por defecto. |
| Comando | `smtpd` | El demonio SMTP. |

Y de las opciones:

| Opción | Significado |
|---|---|
| `syslog_name=postfix/submission` | En el log aparecerá `postfix/submission/smtpd`, así distingues qué puerto usó cada cliente. |
| `smtpd_tls_security_level=encrypt` | En el 587 el cifrado STARTTLS es obligatorio. |
| `smtpd_tls_wrappermode=yes` | En el 465 TLS empieza desde el primer byte. |
| `smtpd_sasl_auth_enable=yes` | Aquí sí se permite autenticarse. |
| `smtpd_client_restrictions=permit_sasl_authenticated,reject` | Solo pueden enviar los usuarios autenticados. |
| `milter_macro_daemon_name=ORIGINATING` | Marca el correo como saliente para que OpenDKIM lo firme (Módulo 9). |

Comprueba y aplica:

```bash title="Terminal"
sudo postfix check
sudo systemctl restart postfix
sudo ss -tlnp | grep master
```

```text title="Salida esperada"
LISTEN 0  100  0.0.0.0:25   0.0.0.0:*  users:(("master",pid=7301,fd=13))
LISTEN 0  100  0.0.0.0:465  0.0.0.0:*  users:(("master",pid=7301,fd=21))
LISTEN 0  100  0.0.0.0:587  0.0.0.0:*  users:(("master",pid=7301,fd=17))
...
```

**Errores habituales:** `postfix/master: fatal: /etc/postfix/master.cf: line N: bad field count` indica que una línea `-o` no empieza con espacios o que falta una columna.

### PASO 7.15 – Abrir los puertos de correo de los clientes

**Comando:** 

```bash title="Terminal"
sudo ufw allow 587/tcp comment 'SMTP submission'
sudo ufw allow 465/tcp comment 'SMTPS'
sudo ufw allow 993/tcp comment 'IMAPS'
sudo ufw allow 995/tcp comment 'POP3S'
sudo ufw status numbered
```

## 7.6 Pruebas de funcionamiento

### PASO 7.16 – Probar la entrega en el buzón (Postfix → LMTP → Dovecot)

**Comando:** 

```bash title="Terminal"
echo "Primer correo para info." | mail -s "Prueba de buzón virtual" info@ejemplo1.com
sudo grep 'info@ejemplo1.com' /var/log/mail.log | tail -5
sudo ls -R /var/mail/vhosts/ejemplo1.com/info/Maildir | head -20
```

Resultado esperado en el log:

```text title="Salida esperada"
postfix/lmtp[7420]: 2D9F03E0FA: to=<info@ejemplo1.com>, relay=mail.ejemplo1.com[private/dovecot-lmtp], delay=0.05, dsn=2.0.0, status=sent (250 2.0.0 <info@ejemplo1.com> 8Kx0BQ5WD2fjHAAA3sKhVQ Saved)
dovecot: lmtp(info@ejemplo1.com)<7421><8Kx0BQ5WD2fjHAAA3sKhVQ>: msgid=<...@mail.ejemplo1.com>: saved mail to INBOX
```

Y el mensaje aparecerá como archivo en `Maildir/new/`.

Prueba también el alias y el rechazo de direcciones inexistentes:

```bash title="Terminal"
echo "Correo al postmaster" | mail -s "Prueba alias" postmaster@ejemplo1.com
echo "No debería llegar" | mail -s "Prueba inexistente" noexiste@ejemplo1.com
sudo tail -n 20 /var/log/mail.log | grep -E 'postmaster|noexiste'
```

El primero se entregará en el buzón de `info@` (`orig_to=<postmaster@ejemplo1.com>`); el segundo será rechazado con `unknown user`.

**Errores habituales:** 

| Log | Causa | Solución |
|---|---|---|
| `connect to private/dovecot-lmtp: No such file or directory` | El socket no existe: Dovecot parado o error en `10-master.conf`. | `ls -l /var/spool/postfix/private/` y `systemctl status dovecot`. |
| `status=bounced (... User doesn't exist: info@ejemplo1.com)` | La dirección está en `vmailbox` de Postfix pero no en `/etc/dovecot/users`. | Ambos archivos deben coincidir. |
| `Recipient address rejected: User unknown in virtual mailbox table` | La dirección no está en `vmailbox` o no ejecutaste `postmap`. | Añádela y `sudo postmap /etc/postfix/vmailbox`. |
| `do not list domain ejemplo1.com in BOTH mydestination and virtual_mailbox_domains` | El dominio aparece en ambos sitios. | Quítalo de `mydestination`. |
| `Permission denied` al crear el Maildir | `/var/mail/vhosts` no pertenece a vmail. | `sudo chown vmail:vmail /var/mail/vhosts`. |

### PASO 7.17 – Probar IMAP por TLS manualmente (puerto 993)

**Objetivo:** iniciar sesión en el buzón «a mano», igual que haría un cliente de correo.

**Comando:** 

```bash title="Terminal"
openssl s_client -connect mail.ejemplo1.com:993 -quiet
```

**Análisis del comando:** `openssl s_client` abre una conexión TLS; `-quiet` oculta los detalles del certificado. Cada comando IMAP empieza con una etiqueta inventada (`a1`, `a2`...) que el servidor repite en su respuesta.

Escribe las líneas que empiezan por `a` (sustituye la contraseña):

```text
* OK [CAPABILITY IMAP4rev1 SASL-IR LOGIN-REFERRALS ID ENABLE IDLE LITERAL+ AUTH=PLAIN AUTH=LOGIN] Dovecot (Ubuntu) ready.
a1 LOGIN info@ejemplo1.com <CONTRASEÑA_INFO>
a1 OK [CAPABILITY ...] Logged in
a2 LIST "" "*"
* LIST (\HasNoChildren \Drafts) "." Drafts
* LIST (\HasNoChildren \Junk) "." Junk
* LIST (\HasNoChildren \Trash) "." Trash
* LIST (\HasNoChildren \Sent) "." Sent
* LIST (\HasNoChildren) "." INBOX
a2 OK List completed
a3 SELECT INBOX
* 2 EXISTS
a3 OK [READ-WRITE] Select completed
a4 FETCH 1 BODY[HEADER.FIELDS (FROM SUBJECT DATE)]
* 1 FETCH (BODY[HEADER.FIELDS (FROM SUBJECT DATE)] {...}
Subject: Prueba de buzón virtual
From: sysadmin@ejemplo1.com
...
a4 OK Fetch completed
a5 LOGOUT
```

**Comprobación:** también puedes ver las sesiones activas mientras estás conectado con `sudo doveadm who` desde otro terminal.

!!! warning "Atención"
    La contraseña quedará en el historial del terminal de tu ordenador si la escribes en la línea de comandos de otros programas. Aquí se escribe dentro de la sesión TLS, así que no queda en el historial del shell.

### PASO 7.18 – Probar POP3 por TLS (puerto 995)

**Comando:** 

```bash title="Terminal"
openssl s_client -connect mail.ejemplo1.com:995 -quiet
```

```text
+OK Dovecot (Ubuntu) ready.
USER info@ejemplo1.com
+OK
PASS <CONTRASEÑA_INFO>
+OK Logged in.
STAT
+OK 2 1024
LIST
+OK 2 messages:
1 498
2 526
.
QUIT
+OK Logging out.
```

### PASO 7.19 – Probar el envío autenticado (puertos 587 y 465)

**Objetivo:** verificar que un usuario autenticado puede enviar a cualquier destino y que sin autenticación no puede.

**Explicación:** usaremos **swaks** (*Swiss Army Knife for SMTP*), la herramienta estándar para probar servidores SMTP.

**Comando:** 

```bash title="Terminal"
sudo apt install -y swaks
swaks --server mail.ejemplo1.com --port 587 --tls --auth LOGIN --auth-user info@ejemplo1.com --from info@ejemplo1.com --to tu-direccion@gmail.com --header "Subject: Prueba por el 587"
```

**Análisis del comando:** 

| Opción | Significado |
|---|---|
| `--server` / `--port` | Servidor y puerto. |
| `--tls` | Usa STARTTLS. Para el puerto 465 se usa `--tlsc` (TLS al conectar). |
| `--auth LOGIN` | Mecanismo de autenticación. |
| `--auth-user` | Usuario. Como no indicamos `--auth-password`, swaks la pide de forma interactiva y no queda en el historial. |
| `--from` / `--to` | Remitente y destinatario. |
| `--header` | Cabecera adicional (el asunto). |

**Resultado esperado (fragmento):** 

```text title="Salida esperada"
<~  220 2.0.0 Ready to start TLS
=== TLS started with cipher TLSv1.3:TLS_AES_256_GCM_SHA384:256
 ~> AUTH LOGIN
<~* 235 2.7.0 Authentication successful
 ~> MAIL FROM:<info@ejemplo1.com>
<~* 250 2.1.0 Ok
 ~> RCPT TO:<tu-direccion@gmail.com>
<~* 250 2.1.5 Ok
<~* 250 2.0.0 Ok: queued as 6A1F23E0FD
```

Repite por el puerto 465:

```bash title="Terminal"
swaks --server mail.ejemplo1.com --port 465 --tlsc --auth LOGIN --auth-user info@ejemplo1.com --from info@ejemplo1.com --to tu-direccion@gmail.com --header "Subject: Prueba por el 465"
```

Y comprueba que sin autenticación el 587 no deja enviar:

```bash title="Terminal"
swaks --server mail.ejemplo1.com --port 587 --tls --from info@ejemplo1.com --to tu-direccion@gmail.com
```

```text title="Salida esperada"
<** 554 5.7.1 <unknown[203.0.113.10]>: Client host rejected: Access denied
```

**Errores habituales:** 

| Síntoma | Causa | Solución |
|---|---|---|
| `535 5.7.8 Error: authentication failed` | Contraseña incorrecta. | `sudo doveadm auth test info@ejemplo1.com`; mira `mail.log` para ver el motivo exacto. |
| `warning: SASL: Connect to private/auth failed: No such file or directory` | El socket de autenticación no existe. | Revisa el bloque `service auth` de `10-master.conf` y reinicia Dovecot. |
| `503 5.5.1 Error: authentication not enabled` | Te conectaste al 25, o falta `smtpd_sasl_auth_enable=yes` en master.cf. | Revisa puerto y `master.cf`. |
| La conexión se corta en el 465 | Usaste `--tls` en vez de `--tlsc`. | En el 465 el TLS es implícito. |

**Siguiente paso:** Módulo 8, creación y gestión de las cuentas de correo.

## 7.7 Ejercicios prácticos del Módulo 7

1. Ejecuta `sudo doveconf -n` y explica cada línea de la salida.
2. Intenta conectar por IMAP sin cifrar con `nc mail.ejemplo1.com 143`. ¿Qué ocurre y por qué?
3. Desde la sesión IMAP manual, crea una carpeta con `a6 CREATE Facturas` y comprueba en el servidor que aparece `/var/mail/vhosts/ejemplo1.com/info/Maildir/.Facturas`.
4. Envía un correo a `info+pruebas@ejemplo1.com`. ¿Llega? ¿A qué buzón? ¿Qué parámetro lo hace posible?
5. Introduce a propósito una contraseña errónea en swaks tres veces y busca los intentos en `/var/log/mail.log`. Esos mensajes son los que vigilará Fail2ban en el Módulo 12.
