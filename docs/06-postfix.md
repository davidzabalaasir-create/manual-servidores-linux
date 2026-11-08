# Módulo 6. Instalación y configuración de Postfix

## Objetivos del módulo

- Comprender cómo viaja un correo electrónico por Internet y qué papel cumple cada componente.
- Distinguir entre recepción (puerto 25), envío autenticado (587 y 465) y relay.
- Instalar Postfix y entender sus archivos y parámetros principales: `myhostname`, `mydomain`, `myorigin`, `mydestination`, `mynetworks`, `inet_interfaces`.
- Configurar TLS y restricciones antispam básicas.
- Enviar correos de prueba, leer los logs y gestionar la cola de correo.

## 6.1 Qué es SMTP y cómo viaja un correo

**SMTP** (*Simple Mail Transfer Protocol*) es el protocolo con el que se **envía** y se **transporta** el correo entre servidores. No sirve para leer el correo: para eso existen IMAP y POP3 (Módulo 7).

En el correo intervienen varios componentes con nombres propios:

| Sigla | Nombre | Función | En nuestro servidor |
|---|---|---|---|
| MUA | *Mail User Agent* | Programa del usuario para leer y escribir correo. | Thunderbird, Outlook, el móvil, la app PHP. |
| MSA | *Mail Submission Agent* | Recibe el correo que envían **tus usuarios autenticados**. | Postfix en los puertos 587 y 465. |
| MTA | *Mail Transfer Agent* | Transporta el correo de servidor a servidor. | Postfix en el puerto 25. |
| MDA | *Mail Delivery Agent* | Deposita el correo en el buzón del destinatario. | Dovecot LMTP (Módulo 7). |
| MAA | *Mail Access Agent* | Permite al usuario consultar su buzón. | Dovecot IMAP/POP3. |

El viaje completo de un correo de `info@ejemplo1.com` a `pepe@gmail.com`:

```text title="Diagrama"
 Thunderbird de info@         Nuestro servidor                       Gmail
 ┌────────────┐  587/465   ┌──────────────────┐     25      ┌───────────────────┐
 │    MUA     │──────────► │ POSTFIX (MSA)    │───────────► │ MTA de Gmail      │
 │            │ usuario +  │ autentica, firma │  consulta   │ comprueba SPF,    │
 └────────────┘ contraseña │ DKIM y entrega   │  DNS MX de  │ DKIM, DMARC, PTR  │
                (TLS)      └──────────────────┘  gmail.com  └─────────┬─────────┘
                                                                      ▼
                                                              buzón de pepe@
 Y en sentido contrario, de pepe@gmail.com a info@ejemplo1.com:
 MTA de Gmail ──25──► consulta MX de ejemplo1.com → mail.ejemplo1.com → POSTFIX (MTA)
              ──LMTP──► DOVECOT ──► /var/mail/vhosts/ejemplo1.com/info/Maildir
              ──IMAP 993──► Thunderbird de info@
```

Una conversación SMTP real es texto plano. Así recibe nuestro servidor un mensaje (C = cliente, S = servidor):

```text
S: 220 mail.ejemplo1.com ESMTP
C: EHLO mail-ed1-f45.google.com
S: 250-mail.ejemplo1.com
S: 250-STARTTLS
S: 250 SMTPUTF8
C: MAIL FROM:<pepe@gmail.com>
S: 250 2.1.0 Ok
C: RCPT TO:<info@ejemplo1.com>
S: 250 2.1.5 Ok
C: DATA
S: 354 End data with <CR><LF>.<CR><LF>
C: From: Pepe <pepe@gmail.com>
C: To: info@ejemplo1.com
C: Subject: Hola
C:
C: Texto del mensaje.
C: .
S: 250 2.0.0 Ok: queued as 4ZK3t21XyPz5
C: QUIT
```

!!! tip "Consejo"
    Fíjate en que hay dos remitentes: el del **sobre** (`MAIL FROM`, también llamado *envelope sender* o *Return-Path*), que usan los servidores para devolver errores y comprobar SPF; y el de la **cabecera** (`From:`), que es el que ve el usuario y el que comprueba DMARC. Entender esta diferencia es clave en el Módulo 9.

## 6.2 Los puertos SMTP: recepción, envío y envío autenticado

| Puerto | Nombre | Quién se conecta | Autenticación | Cifrado | Uso |
|---|---|---|---|---|---|
| **25** | SMTP | Otros servidores de correo de Internet. | No (los servidores no tienen contraseña en el tuyo). | Opcional con STARTTLS. | **Recepción**: correo que llega a tus dominios. Postfix solo acepta mensajes **para tus dominios**; rechaza todo lo demás (evita ser un *relay abierto*). |
| **587** | Submission | Tus usuarios (Thunderbird, móvil, apps). | **Obligatoria** (usuario y contraseña). | Obligatorio con STARTTLS. | **Envío autenticado**: tus usuarios envían correo a cualquier destino. |
| **465** | SMTPS (Submissions) | Tus usuarios. | **Obligatoria**. | Obligatorio, TLS implícito desde el primer byte. | Igual que 587, pero con TLS directo. RFC 8314 lo recomienda; ambos son válidos. |

Conceptos:

- **STARTTLS**: la conexión empieza en texto plano y el cliente pide pasar a cifrado con el comando `STARTTLS`.
- **TLS implícito**: la conexión es cifrada desde el principio (como HTTPS).
- **Relay abierto**: un servidor que acepta y reenvía correo de cualquiera a cualquiera. Los spammers los buscan constantemente; si tu servidor lo fuera, en horas estaría en todas las listas negras. La regla de Postfix será: «acepto correo **para** mis dominios de cualquiera; acepto correo **hacia** cualquier destino solo de usuarios autenticados o del propio servidor».

!!! warning "Atención"
    Tus usuarios **nunca** deben enviar por el puerto 25. Muchos operadores de acceso a Internet bloquean el 25 saliente de sus clientes domésticos precisamente para frenar el spam de equipos infectados. Los clientes de correo usan siempre 587 o 465.

### PASO 6.1 – Comprobar si tu proveedor permite el puerto 25 saliente

**Objetivo:** saber si tu servidor podrá entregar correo a otros servidores.

**Comando:** 

```bash title="Terminal"
nc -vz -w 5 gmail-smtp-in.l.google.com 25
```

**Análisis del comando:** `nc` (netcat) abre una conexión; `-v` muestra detalles, `-z` solo comprueba si el puerto responde sin enviar datos, `-w 5` espera como máximo 5 segundos. `gmail-smtp-in.l.google.com` es uno de los servidores de recepción de Gmail.

**Resultado esperado:** 

```text title="Salida esperada"
Connection to gmail-smtp-in.l.google.com (142.250.153.27) 25 port [tcp/smtp] succeeded!
```

**Errores habituales:** si obtienes `Connection timed out`, tu proveedor bloquea el 25 saliente. Opciones: solicitar el desbloqueo (muchos lo hacen tras verificar tu cuenta), cambiar de proveedor o configurar un *relayhost* (sección 6.7). Podrás seguir el manual igualmente: la recepción no depende de esto.

**Siguiente paso:** obtener el certificado del servidor de correo.

### PASO 6.2 – Obtener el certificado TLS para mail.ejemplo1.com y mail.ejemplo2.com

**Objetivo:** tener un certificado válido que Postfix y Dovecot presentarán a clientes y servidores.

**Explicación:** los clientes de correo de ejemplo1 se conectarán a `mail.ejemplo1.com` y los de ejemplo2 a `mail.ejemplo2.com`. Un único certificado con ambos nombres sirve para los dos. Lo validamos a través del sitio comodín de Apache (PASO 3.4), cuyo DocumentRoot es `/var/www/html`.

**Comando:** 

```bash title="Terminal"
sudo certbot certonly --webroot -w /var/www/html -d mail.ejemplo1.com -d mail.ejemplo2.com --email administracion@ejemplo1.com --agree-tos --no-eff-email
```

**Resultado esperado:** 

```text title="Salida esperada"
Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/mail.ejemplo1.com/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/mail.ejemplo1.com/privkey.pem
```

**Comprobación:** `sudo certbot certificates` debe mostrar un tercer certificado `mail.ejemplo1.com` con los dominios `mail.ejemplo1.com mail.ejemplo2.com`.

!!! tip "Consejo"
    **Laboratorio sin dominio real:** usa el certificado autofirmado. En los archivos de Postfix y Dovecot de este manual sustituye `/etc/letsencrypt/live/mail.ejemplo1.com/fullchain.pem` por `/etc/ssl/certs/ssl-cert-snakeoil.pem` y `/etc/letsencrypt/live/mail.ejemplo1.com/privkey.pem` por `/etc/ssl/private/ssl-cert-snakeoil.key`. Los clientes de correo te pedirán aceptar una excepción de seguridad.

**Siguiente paso:** instalar Postfix.

## 6.3 Instalación de Postfix

### PASO 6.3 – Instalar Postfix

**Objetivo:** instalar el servidor SMTP y una utilidad para enviar correos desde la consola.

**Comando:** 

```bash title="Terminal"
sudo apt install -y postfix bsd-mailx
```

**Análisis del comando:** `postfix` es el servidor; `bsd-mailx` aporta el comando `mail` para enviar y leer correo desde el terminal.

Durante la instalación aparece un asistente de configuración (debconf):

| Pantalla | Respuesta | Explicación |
|---|---|---|
| General type of mail configuration | **Internet Site** | El servidor envía y recibe directamente con SMTP. |
| System mail name | **ejemplo1.com** | Dominio que se añade al correo generado localmente (se guarda en `/etc/mailname`). |

Si se abre alguna pantalla más (destinos, redes...), acepta el valor por defecto: sustituiremos la configuración completa en el paso siguiente. Si te equivocas, puedes repetir el asistente con `sudo dpkg-reconfigure postfix`.

**Resultado esperado:** termina con `Postfix (main.cf) is now set up with a default configuration.`

**Comprobación:** 

```bash title="Terminal"
postconf mail_version
systemctl status postfix@- --no-pager
```

```text title="Salida esperada"
mail_version = 3.8.6
● postfix@-.service - Postfix Mail Transport Agent (instance -)
     Active: active (running) since ...
```

!!! tip "Consejo"
    En Ubuntu, `postfix.service` es solo un «envoltorio» que arranca las instancias. La instancia real se llama `postfix@-.service`. Para reiniciar puedes usar indistintamente `systemctl restart postfix`, pero para ver el estado detallado y los errores usa `postfix@-`.

**Siguiente paso:** conocer la estructura de Postfix.

## 6.4 Estructura y arquitectura de Postfix

Postfix no es un único programa, sino un conjunto de pequeños procesos especializados coordinados por un proceso **master**. Este diseño lo hace rápido y seguro: cada proceso tiene los privilegios justos y muchos se ejecutan encerrados (*chroot*) en `/var/spool/postfix`.

```text title="Diagrama"
                    ┌────────────── master (proceso principal, root) ───────────────┐
  Internet :25 ───► │ smtpd ──► cleanup ──► cola incoming ──► qmgr (gestor de cola) │
  Usuarios :587 ──► │ smtpd                                      │                  │
  PHP/sendmail ───► │ pickup ──► cleanup ─┘                      ├──► smtp  ──► Internet
                    │                                            ├──► lmtp  ──► Dovecot
                    │                                            └──► local ──► /var/mail
                    └──────────────────────────────────────────────────────────────┘
```

| Archivo o carpeta | Contenido |
|---|---|
| `/etc/postfix/main.cf` | Configuración principal: parámetros `nombre = valor`. |
| `/etc/postfix/master.cf` | Qué servicios arrancan (smtpd en 25, 587, 465...) y con qué opciones. |
| `/etc/aliases` | Alias de las cuentas locales del sistema (root, postmaster...). |
| `/etc/mailname` | Dominio de correo del sistema. |
| `/var/spool/postfix/` | Colas de correo y sockets (`private/`, `public/`). |
| `/var/log/mail.log` | Log de todo el correo (lo escribe rsyslog). |

Herramientas de administración:

| Comando | Qué hace |
|---|---|
| `postconf -n` | Muestra solo los parámetros que **tú** has cambiado respecto a los valores por defecto. |
| `postconf -d parametro` | Muestra el valor por defecto de un parámetro. |
| `postconf parametro` | Muestra el valor actual. |
| `sudo postconf -e 'param = valor'` | Modifica un parámetro en `main.cf`. |
| `sudo postfix check` | Revisa configuración, permisos y archivos. |
| `sudo postmap archivo` | Compila un archivo de tabla a formato `.db` (lo verás en el Módulo 8). |
| `sudo newaliases` | Compila `/etc/aliases`. |
| `mailq` o `postqueue -p` | Muestra la cola de correo pendiente. |
| `sudo postqueue -f` | Fuerza el reintento inmediato de toda la cola. |
| `sudo postsuper -d ID` | Borra un mensaje de la cola (`-d ALL` borra todos). |
| `sudo postcat -q ID` | Muestra el contenido de un mensaje en cola. |

## 6.5 Los parámetros fundamentales

| Parámetro | Significado | Valor en nuestro servidor |
|---|---|---|
| `myhostname` | Nombre completo (FQDN) del servidor. Se usa en el saludo SMTP (`220 mail.ejemplo1.com`) y en el `EHLO` al enviar. **Debe** coincidir con el registro A y con el PTR. | `mail.ejemplo1.com` |
| `mydomain` | Dominio del servidor. Por defecto es `myhostname` sin la primera parte. | `ejemplo1.com` |
| `myorigin` | Dominio que se añade a los correos generados localmente sin dominio (por ejemplo, el que envía `cron` como `root`). | `$mydomain` → `root@ejemplo1.com` |
| `mydestination` | Dominios cuyos destinatarios son **cuentas del sistema Linux** (entrega local en `/var/mail/usuario`). | `$myhostname, localhost.$mydomain, localhost` |
| `mynetworks` | Redes «de confianza» que pueden enviar a cualquier destino sin autenticarse. | Solo el propio servidor: `127.0.0.0/8` y `[::1]/128`. |
| `inet_interfaces` | En qué interfaces de red escucha Postfix. | `all` (necesario para recibir de Internet). |
| `inet_protocols` | IPv4, IPv6 o ambos. | `all` (o `ipv4` si tu servidor no tiene IPv6 funcional). |
| `relayhost` | Servidor intermediario por el que enviar todo el correo saliente. | Vacío (entrega directa). |

!!! warning "Atención"
    **Muy importante:** `ejemplo1.com` y `ejemplo2.com` **no** se ponen en `mydestination`. Esos dominios tendrán buzones **virtuales** (Módulo 8), gestionados por Dovecot y desvinculados de las cuentas Linux. Un dominio nunca debe aparecer a la vez en `mydestination` y en `virtual_mailbox_domains`. En `mydestination` solo dejamos el nombre del propio servidor, para el correo administrativo del sistema.

Tipos de dominio que Postfix sabe gestionar:

| Clase de dominio | Parámetro | Buzones | Uso |
|---|---|---|---|
| Local | `mydestination` | Cuentas Linux (`/var/mail/usuario` o `~/Maildir`). | Correo del sistema (root, cron). |
| Alias virtual | `virtual_alias_domains` | Ninguno: todo se reenvía a otras direcciones. | Dominios que solo redirigen. |
| **Buzón virtual** | `virtual_mailbox_domains` | Buzones independientes de las cuentas Linux. | **Nuestros dominios de clientes.** |
| Relay | `relay_domains` | En otro servidor. | Servidor de respaldo (MX secundario). |

## 6.6 Configuración de Postfix

### PASO 6.4 – Escribir la configuración principal main.cf

**Objetivo:** dejar `main.cf` con una configuración limpia, comentada y segura.

**Explicación:** en lugar de modificar parámetro a parámetro, sustituimos el archivo por una versión completa que entiendes línea por línea. Primero guardamos una copia del original.

**Comando:** 

```bash title="Terminal"
sudo cp /etc/postfix/main.cf /etc/postfix/main.cf.original
sudo nano /etc/postfix/main.cf
```

**Archivo:** `/etc/postfix/main.cf` (sustituye todo el contenido; versión del Módulo 6):

```text title="Archivo de configuración"
# =====================================================================
#  /etc/postfix/main.cf  -  mail.ejemplo1.com
#  Versión del Módulo 6: SMTP básico con TLS.
#  (En los módulos 7, 8 y 9 se añadirán Dovecot, buzones virtuales y DKIM)
# =====================================================================

# --- Generales ---------------------------------------------------------
compatibility_level = 3.6
smtpd_banner = $myhostname ESMTP
biff = no
append_dot_mydomain = no
readme_directory = no

# --- Identidad del servidor -------------------------------------------
myhostname = mail.ejemplo1.com
mydomain = ejemplo1.com
myorigin = $mydomain
# Solo el propio servidor es destino LOCAL (cuentas del sistema)
mydestination = $myhostname, localhost.$mydomain, localhost
# Redes de confianza: solo el propio servidor
mynetworks = 127.0.0.0/8 [::ffff:127.0.0.0]/104 [::1]/128
inet_interfaces = all
inet_protocols = all
relayhost =

# --- Alias y límites ---------------------------------------------------
alias_maps = hash:/etc/aliases
alias_database = hash:/etc/aliases
recipient_delimiter = +
mailbox_size_limit = 0
# Tamaño máximo de un mensaje: 50 MB
message_size_limit = 52428800

# --- TLS cuando otros se conectan a nosotros (smtpd) -------------------
smtpd_tls_cert_file = /etc/letsencrypt/live/mail.ejemplo1.com/fullchain.pem
smtpd_tls_key_file = /etc/letsencrypt/live/mail.ejemplo1.com/privkey.pem
smtpd_tls_security_level = may
smtpd_tls_auth_only = yes
smtpd_tls_protocols = >=TLSv1.2
smtpd_tls_mandatory_protocols = >=TLSv1.2
smtpd_tls_loglevel = 1
smtpd_tls_session_cache_database = btree:${data_directory}/smtpd_scache

# --- TLS cuando nosotros enviamos a otros (smtp) -----------------------
smtp_tls_security_level = may
smtp_tls_CApath = /etc/ssl/certs
smtp_tls_loglevel = 1
smtp_tls_session_cache_database = btree:${data_directory}/smtp_scache

# --- Restricciones (antispam y anti relay abierto) ---------------------
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
```

Análisis por bloques:

| Parámetro | Significado |
|---|---|
| `compatibility_level = 3.6` | Activa los valores por defecto seguros de Postfix 3.6 en adelante. |
| `smtpd_banner = $myhostname ESMTP` | Saludo del servidor. No revela la versión de Postfix ni el sistema operativo. |
| `biff = no` | Desactiva un antiguo servicio de notificación local. |
| `append_dot_mydomain = no` | No completa direcciones incompletas (es tarea del cliente). |
| `recipient_delimiter = +` | Permite direcciones como `info+facturas@ejemplo1.com`, que se entregan a `info`. |
| `smtpd_tls_security_level = may` | En el puerto 25, TLS se ofrece pero no se exige (si lo exigiéramos, rechazaríamos correo de servidores antiguos). |
| `smtpd_tls_auth_only = yes` | Nunca se aceptan contraseñas sin cifrar. |
| `smtpd_tls_protocols = >=TLSv1.2` | Solo TLS 1.2 y 1.3. |
| `smtp_tls_security_level = may` | Al enviar, cifra si el destino lo admite. |
| `smtp_tls_CApath` | Autoridades de confianza para verificar los certificados de otros servidores. |
| `smtpd_helo_required` | El cliente debe presentarse con HELO/EHLO. |
| `disable_vrfy_command` | Desactiva `VRFY`, que permitiría a un spammer averiguar qué direcciones existen. |
| `reject_invalid_helo_hostname` / `reject_non_fqdn_helo_hostname` | Rechaza clientes que se presentan con nombres mal formados. Los servidores legítimos usan un FQDN. |
| `reject_non_fqdn_sender` / `reject_unknown_sender_domain` | Rechaza remitentes con dominios incompletos o inexistentes en el DNS. |
| `smtpd_relay_restrictions` | La regla anti relay abierto: permite enviar a cualquier destino solo a `mynetworks` y a usuarios autenticados (SASL); al resto solo les deja entregar a nuestros dominios. |
| `reject_unauth_destination` | Rechaza destinatarios que no son nuestros si el cliente no está autorizado. |
| `permit_sasl_authenticated` | Autoriza a usuarios autenticados. Aún no tenemos SASL (llega en el Módulo 7), pero dejarlo no hace daño. |

!!! tip "Consejo"
    Las restricciones se evalúan **en orden** y la primera que da un veredicto (`permit` o `reject`) gana. Por eso `permit_mynetworks` y `permit_sasl_authenticated` van siempre primero: tus usuarios no deben sufrir los filtros antispam pensados para desconocidos.

**Comprobación:** 

```bash title="Terminal"
sudo postfix check
postconf -n | head -20
```

`postfix check` no debe mostrar nada. `postconf -n` debe reflejar tus valores.

**Errores habituales:** `postconf: warning: /etc/postfix/main.cf, line N: missing '=' after attribute name` indica una línea mal escrita. Recuerda: las líneas de continuación (como los elementos de las restricciones) **deben empezar con espacios**; una línea que empieza sin espacio se interpreta como un parámetro nuevo.

**Siguiente paso:** los alias del sistema.

### PASO 6.5 – Configurar los alias del sistema

**Objetivo:** que el correo dirigido a `root` y a `postmaster` del servidor llegue al administrador.

**Explicación:** los servicios del sistema (cron, actualizaciones automáticas, Fail2ban, mdadm...) envían avisos a `root`. Nadie lee el buzón de root: los redirigimos al usuario `sysadmin`. En el Módulo 8 cambiaremos este destino por una dirección virtual real.

**Archivo:** `/etc/aliases` (sustituye todo el contenido):

```text title="Archivo de configuración"
# Alias del sistema: formato "alias: destino"
# Después de modificar este archivo ejecuta: sudo newaliases
postmaster:    root
abuse:         root
webmaster:     root
www-data:      root
root:          sysadmin
```

**Comando:** 

```bash title="Terminal"
sudo newaliases
sudo systemctl restart postfix
```

**Análisis del comando:** `newaliases` compila `/etc/aliases` en `/etc/aliases.db`, el formato que lee Postfix. Si editas el archivo y olvidas este comando, los cambios no se aplican.

**Siguiente paso:** abrir el puerto 25 y probar.

### PASO 6.6 – Abrir el puerto 25 en el cortafuegos

**Comando:** 

```bash title="Terminal"
sudo ufw allow 25/tcp comment 'SMTP recepcion'
sudo ss -tlnp | grep ':25 '
```

**Resultado esperado:** 

```text title="Salida esperada"
Rule added
Rule added (v6)
LISTEN 0  100   0.0.0.0:25   0.0.0.0:*  users:(("master",pid=6012,fd=13))
LISTEN 0  100      [::]:25      [::]:*  users:(("master",pid=6012,fd=14))
```

Postfix escucha en todas las interfaces (`0.0.0.0` y `[::]`) en el puerto 25.

## 6.7 Pruebas, logs y cola de correo

### PASO 6.7 – Enviar un correo local de prueba

**Objetivo:** comprobar la entrega local.

**Comando:** 

```bash title="Terminal"
echo "Este es el cuerpo del mensaje de prueba." | mail -s "Prueba local 1" root
sudo tail -n 15 /var/log/mail.log
```

**Análisis del comando:** `mail -s "asunto" destinatario` envía lo que recibe por la entrada estándar. Como `root` es alias de `sysadmin`, el mensaje debe llegar al buzón de `sysadmin`.

Resultado esperado en el log (cada línea comparte un **ID de cola**, aquí `5B2A13E0F1`, que te permite seguir el mensaje):

```text title="Salida esperada"
postfix/pickup[6100]: 5B2A13E0F1: uid=1000 from=<sysadmin>
postfix/cleanup[6170]: 5B2A13E0F1: message-id=<20261109001501.5B2A13E0F1@mail.ejemplo1.com>
postfix/qmgr[6101]: 5B2A13E0F1: from=<sysadmin@ejemplo1.com>, size=412, nrcpt=1 (queue active)
postfix/local[6172]: 5B2A13E0F1: to=<sysadmin@mail.ejemplo1.com>, orig_to=<root>, relay=local, delay=0.03, status=sent (delivered to mailbox)
postfix/qmgr[6101]: 5B2A13E0F1: removed
```

**Comprobación:** lee el correo con el comando `mail` (como `sysadmin`, sin sudo):

```bash title="Terminal"
mail
```

Verás la lista de mensajes; escribe el número para leerlo y `q` para salir. El buzón local está en `/var/mail/sysadmin`.

**Errores habituales:** `status=bounced (unknown user: "root")` indica que no ejecutaste `newaliases`.

### PASO 6.8 – Enviar un correo a una dirección externa

**Objetivo:** comprobar que el servidor entrega correo en Internet.

**Comando:** usa una dirección tuya de Gmail, Outlook u otro proveedor:

```bash title="Terminal"
echo "Prueba de envío desde mail.ejemplo1.com" | mail -s "Prueba externa" -r sysadmin@mail.ejemplo1.com tu-direccion@gmail.com
sudo tail -f /var/log/mail.log
```

**Análisis del comando:** `-r` fija el remitente del sobre. Usamos `sysadmin@mail.ejemplo1.com` porque es una dirección local que sí existe y puede recibir respuestas.

Resultado esperado en el log:

```text title="Salida esperada"
postfix/smtp[6230]: Trusted TLS connection established to gmail-smtp-in.l.google.com[142.250.153.27]:25: TLSv1.3 with cipher TLS_AES_256_GCM_SHA384 (256/256 bits)
postfix/smtp[6230]: 8C1D23E0F4: to=<tu-direccion@gmail.com>, relay=gmail-smtp-in.l.google.com[142.250.153.27]:25, delay=1.2, dsn=2.0.0, status=sent (250 2.0.0 OK  1762647301 ...)
```

`status=sent` con `250 2.0.0 OK` significa que Gmail **aceptó** el mensaje. Pulsa Ctrl+C para salir.

!!! warning "Atención"
    Que Gmail lo acepte no significa que llegue a la bandeja de entrada. Sin SPF, DKIM, DMARC y PTR (Módulo 9) es muy probable que acabe en spam o que sea rechazado con un mensaje como `550-5.7.1 ... does not meet IPv6 sending guidelines regarding PTR records and authentication`. Es normal en este punto.

**Errores habituales:** 

| Mensaje en mail.log | Significado | Solución |
|---|---|---|
| `connect to ...:25: Connection timed out` + `status=deferred` | El puerto 25 saliente está bloqueado. | PASO 6.1 y sección 6.8 (relayhost). |
| `status=bounced ... 550-5.7.1 ... PTR records` | Falta el DNS inverso. | Módulo 9. |
| `status=deferred ... 421-4.7.0 ... unusual rate of unsolicited mail` | La reputación de tu IP es mala o nueva. | Configura todo el Módulo 9 y envía poco volumen al principio. |
| `Network is unreachable` en direcciones IPv6 | El servidor no tiene IPv6 funcional. | `sudo postconf -e 'inet_protocols = ipv4'` y `sudo systemctl restart postfix`. |

### PASO 6.9 – Probar la recepción con una conversación SMTP manual

**Objetivo:** comprobar que el servidor recibe correo desde Internet y ver el protocolo con tus propios ojos.

**Explicación:** aún no hemos creado los registros MX, pero cuando un dominio no tiene MX, los servidores usan su registro A. Como `mail.ejemplo1.com` tiene registro A, podemos enviar a `sysadmin@mail.ejemplo1.com`. Haremos la conversación a mano desde **tu ordenador** (si tu conexión doméstica bloquea el 25, hazla desde otro servidor o desde el propio servidor contra su IP pública).

**Comando (en tu ordenador con Linux o macOS; en Windows instala `ncat` del paquete Nmap o usa `telnet`):** 

```bash title="Terminal"
nc mail.ejemplo1.com 25
```

Escribe las líneas marcadas con C (sin la C) y observa las respuestas:

```text
S: 220 mail.ejemplo1.com ESMTP
C: EHLO prueba.ejemplo.org
S: 250-mail.ejemplo1.com
S: 250-PIPELINING
S: 250-SIZE 52428800
S: 250-ETRN
S: 250-STARTTLS
S: 250-ENHANCEDSTATUSCODES
S: 250-8BITMIME
S: 250-DSN
S: 250-SMTPUTF8
S: 250 CHUNKING
C: MAIL FROM:<prueba@gmail.com>
S: 250 2.1.0 Ok
C: RCPT TO:<sysadmin@mail.ejemplo1.com>
S: 250 2.1.5 Ok
C: DATA
S: 354 End data with <CR><LF>.<CR><LF>
C: Subject: Prueba manual
C:
C: Hola desde una conversacion SMTP manual.
C: .
S: 250 2.0.0 Ok: queued as 1F0A43E0F7
C: QUIT
S: 221 2.0.0 Bye
```

**Comprobación:** ejecuta `mail` como `sysadmin` en el servidor; el mensaje debe estar ahí.

Ahora comprueba que **no** eres un relay abierto. Repite la conversación pero con un destinatario externo:

```text
C: RCPT TO:<alguien@gmail.com>
S: 454 4.7.1 <alguien@gmail.com>: Relay access denied
```

Ese rechazo es la prueba de que el servidor solo acepta correo para sus propios destinos cuando el cliente no está autenticado.

!!! tip "Consejo"
    También puedes enviar un correo desde tu Gmail a `sysadmin@mail.ejemplo1.com` y verlo llegar con `mail`.

### PASO 6.10 – Leer los logs y gestionar la cola

**Objetivo:** dominar las herramientas de diagnóstico del correo.

Comandos más útiles:

```bash title="Terminal"
# Seguir el log en tiempo real
sudo tail -f /var/log/mail.log
# Buscar todo lo relacionado con un mensaje por su ID de cola
sudo grep 8C1D23E0F4 /var/log/mail.log
# Buscar los envíos a una dirección
sudo grep 'to=<tu-direccion@gmail.com>' /var/log/mail.log
# Ver sólo errores y avisos
sudo grep -E 'warning|error|fatal|reject' /var/log/mail.log | tail -20
# También desde systemd
sudo journalctl -u postfix@- --since "1 hour ago" --no-pager
```

Cola de correo:

```bash title="Terminal"
mailq
sudo postcat -q 8C1D23E0F4 | less
sudo postqueue -f
sudo postsuper -d 8C1D23E0F4
sudo postsuper -d ALL deferred
```

Análisis: `mailq` lista los mensajes pendientes con el motivo del retraso; `postcat -q` muestra un mensaje concreto; `postqueue -f` fuerza el reintento; `postsuper -d` borra un mensaje (o todos los diferidos con `ALL deferred`). Postfix reintenta automáticamente los mensajes diferidos durante 5 días (`maximal_queue_lifetime`) antes de devolverlos al remitente.

Estados de entrega en el log:

| Estado | Significado |
|---|---|
| `status=sent` | Entregado (al buzón local o aceptado por el servidor remoto). |
| `status=deferred` | Error temporal (código 4xx): se reintentará. |
| `status=bounced` | Error permanente (código 5xx): se devuelve al remitente. |
| `status=expired` | Se agotó el tiempo de reintentos. |

## 6.8 Si tu proveedor bloquea el puerto 25: relayhost

Si no puedes enviar directamente, puedes entregar todo el correo saliente a un servicio de envío (por ejemplo, el *smarthost* de tu proveedor o un servicio transaccional) que lo reenvía por ti. La recepción por el puerto 25 no cambia. La configuración general es:

**Archivo:** `/etc/postfix/sasl_passwd` (nuevo; los datos te los da el servicio de relay):

```text title="Archivo de configuración"
[smtp.servicio-relay.example]:587 usuario_relay:<CONTRASEÑA_RELAY>
```

**Comando:** 

```bash title="Terminal"
sudo chmod 600 /etc/postfix/sasl_passwd
sudo postmap /etc/postfix/sasl_passwd
sudo postconf -e 'relayhost = [smtp.servicio-relay.example]:587'
sudo postconf -e 'smtp_sasl_auth_enable = yes'
sudo postconf -e 'smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd'
sudo postconf -e 'smtp_sasl_security_options = noanonymous'
sudo postconf -e 'smtp_tls_security_level = encrypt'
sudo systemctl restart postfix
```

Análisis: los corchetes `[...]` indican a Postfix que no busque registros MX de ese nombre. `postmap` compila el archivo de contraseñas. `smtp_tls_security_level = encrypt` obliga a cifrar la conexión con el relay, porque por ella viaja tu contraseña. Con un relay, el registro SPF (Módulo 9) debe incluir también los servidores del servicio de relay, según indique su documentación.

## 6.9 Ejercicios prácticos del Módulo 6

1. Ejecuta `postconf -d myorigin` y `postconf myorigin`. Explica la diferencia.
2. Envía un correo a una dirección inexistente de tu propio servidor (`noexiste@mail.ejemplo1.com`) y sigue en el log el rebote (`status=bounced`). ¿Dónde llega la notificación de error?
3. Repite la conversación SMTP manual pero usando `EHLO prueba` (sin dominio). ¿Qué restricción la rechaza y con qué código?
4. Detén temporalmente la salida al puerto 25 con `sudo ufw deny out 25/tcp`, envía un correo externo, observa la cola con `mailq` y vuelve a permitirlo con `sudo ufw delete deny out 25/tcp`. Fuerza la entrega con `postqueue -f`.
5. Dibuja el recorrido de un correo enviado por la aplicación PHP de ejemplo1.com indicando qué procesos de Postfix intervienen.
