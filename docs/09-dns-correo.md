# Módulo 9. Configuración DNS del servidor de correo

## Objetivos del módulo

- Entender por qué el DNS determina si tus correos llegan a la bandeja de entrada o al spam.
- Configurar los registros A, MX, SPF, DKIM y DMARC de cada dominio y el DNS inverso (PTR).
- Instalar y configurar OpenDKIM, generar las claves y conectarlo con Postfix.
- Comprobar la propagación de los registros y verificar la autenticación con servidores reales.
- Conocer los factores de reputación de la IP y las buenas prácticas de entregabilidad.

## 9.1 Por qué el DNS es crítico para el correo

El protocolo SMTP se diseñó en 1982, cuando Internet era una red de confianza: cualquiera puede escribir `MAIL FROM:<presidente@banco.com>`. Para combatir la suplantación y el spam, los grandes proveedores (Gmail, Outlook, Yahoo...) comprueban varias cosas antes de aceptar un mensaje. Desde 2024, Gmail y Yahoo **exigen** SPF, DKIM, DMARC y DNS inverso a quien envía a sus usuarios.

```text title="Diagrama"
 Nuestro servidor (203.0.113.10) envía a Gmail un correo From: info@ejemplo1.com
                         │
                         ▼
 ┌──────────────────── Servidor de Gmail comprueba ─────────────────────┐
 │ 1. PTR:  ¿203.0.113.10 tiene nombre? → mail.ejemplo1.com             │
 │          ¿y mail.ejemplo1.com apunta a 203.0.113.10? (FCrDNS)        │
 │ 2. SPF:  ¿203.0.113.10 está autorizada en el TXT de ejemplo1.com?    │
 │ 3. DKIM: ¿la firma de la cabecera DKIM-Signature es válida con la    │
 │          clave pública publicada en mail._domainkey.ejemplo1.com?    │
 │ 4. DMARC: ¿SPF o DKIM pasan Y su dominio coincide con el del From:?  │
 │          ¿qué política pide _dmarc.ejemplo1.com si no coincide?      │
 │ 5. Reputación de la IP y del dominio, listas negras, contenido...    │
 └──────────────────────────────┬───────────────────────────────────────┘
                                ▼
              Bandeja de entrada · Spam · Rechazo
```

| Mecanismo | Qué demuestra | Dónde se publica |
|---|---|---|
| **PTR** (DNS inverso) | Que la IP pertenece a un servidor de correo identificado y bien configurado. | En la zona inversa de la IP (la gestiona tu proveedor del servidor). |
| **SPF** | Qué IP están autorizadas a enviar correo con ese dominio en el sobre (`MAIL FROM`). | TXT en el dominio. |
| **DKIM** | Que el mensaje no ha sido alterado y que lo firmó el dominio. | TXT en `selector._domainkey.dominio`. |
| **DMARC** | Qué hacer si SPF/DKIM fallan o no coinciden con el `From:` visible, y a dónde enviar informes. | TXT en `_dmarc.dominio`. |

## 9.2 Registros A y MX

### PASO 9.1 – Verificar los registros A y crear los registros MX

**Objetivo:** indicar a Internet qué servidor recibe el correo de cada dominio.

**Explicación:** los registros A de `mail.ejemplo1.com` y `mail.ejemplo2.com` los creaste en el PASO 3.10. Ahora añade los MX en el panel de tu proveedor DNS. Cada dominio apunta a «su» nombre de servidor de correo, aunque ambos resuelvan a la misma IP: así cada cliente ve un nombre coherente con su dominio.

Registros de `ejemplo1.com`:

| Tipo | Nombre | Prioridad | Valor | TTL |
|---|---|---|---|---|
| MX | @ | 10 | mail.ejemplo1.com | 3600 |

Registros de `ejemplo2.com`:

| Tipo | Nombre | Prioridad | Valor | TTL |
|---|---|---|---|---|
| MX | @ | 10 | mail.ejemplo2.com | 3600 |

En formato de archivo de zona (por si tu proveedor permite importarlo), los registros completos de correo de ejemplo1.com quedarán así al final del módulo:

```text title="Registros DNS"
ejemplo1.com.                  3600 IN MX   10 mail.ejemplo1.com.
mail.ejemplo1.com.             3600 IN A    203.0.113.10
mail.ejemplo1.com.             3600 IN AAAA 2001:db8:10::10
ejemplo1.com.                  3600 IN TXT  "v=spf1 mx ~all"
mail.ejemplo1.com.             3600 IN TXT  "v=spf1 a -all"
mail._domainkey.ejemplo1.com.  3600 IN TXT  "v=DKIM1; h=sha256; k=rsa; p=MIIBIjANBgkq...(clave pública)..."
_dmarc.ejemplo1.com.           3600 IN TXT  "v=DMARC1; p=none; rua=mailto:dmarc@ejemplo1.com; adkim=r; aspf=r"
```

!!! warning "Atención"
    El valor del MX es un **nombre**, nunca una IP, y ese nombre debe tener registro A (o AAAA) propio; no puede ser un CNAME. Algunos paneles exigen escribir el punto final (`mail.ejemplo1.com.`); otros lo añaden solos. Si escribes el nombre sin punto en un panel que no lo añade, acabarás con `mail.ejemplo1.com.ejemplo1.com`.

**Comprobación:** 

```bash title="Terminal"
dig +short MX ejemplo1.com
dig +short MX ejemplo2.com
dig +short A mail.ejemplo2.com
```

```text title="Salida esperada"
10 mail.ejemplo1.com.
10 mail.ejemplo2.com.
203.0.113.10
```

Ahora ya puedes enviar desde Gmail a `info@ejemplo1.com` y a `soporte@ejemplo2.com`: Gmail buscará el MX y entregará en tu servidor.

## 9.3 El DNS inverso (PTR)

El registro **PTR** traduce una IP a un nombre. A diferencia del resto de registros, **no** lo configuras en el panel de tu dominio, porque la «zona inversa» de una IP pertenece a quien la posee: tu proveedor del servidor.

Los servidores receptores comprueban el **FCrDNS** (*Forward-Confirmed reverse DNS*):

```text title="Diagrama"
  203.0.113.10 ──PTR──► mail.ejemplo1.com ──A──► 203.0.113.10    ✓ coinciden
```

Además, ese nombre debería coincidir con el `myhostname` de Postfix (que se presenta en el `EHLO`).

### PASO 9.2 – Configurar el PTR en el panel de tu proveedor

**Objetivo:** que la IP del servidor tenga como nombre inverso `mail.ejemplo1.com`.

**Explicación:** busca en el panel de tu proveedor una opción llamada «Reverse DNS», «rDNS», «PTR» o similar, normalmente en la sección de red o de IPs del servidor. Escribe `mail.ejemplo1.com` para la IPv4 y, si la tienes, también para la IPv6. Algunos proveedores lo configuran automáticamente con el nombre que des al servidor; otros requieren abrir una incidencia.

!!! tip "Consejo"
    Una IP solo puede tener un PTR. No pasa nada por que `ejemplo2.com` use un servidor cuyo PTR es `mail.ejemplo1.com`: es lo normal en cualquier proveedor de alojamiento. Lo que importa es que el PTR exista, resuelva de vuelta a la misma IP y no sea genérico (como `203-0-113-10.static.proveedor.net`).

**Comprobación:** 

```bash title="Terminal"
dig +short -x 203.0.113.10
dig +short -x 2001:db8:10::10
dig +short A mail.ejemplo1.com
```

```text title="Salida esperada"
mail.ejemplo1.com.
mail.ejemplo1.com.
203.0.113.10
```

**Errores habituales:** si devuelve un nombre genérico del proveedor, el cambio aún no se ha aplicado o no se ha hecho. Si tu proveedor no permite cambiar el PTR (algo habitual en conexiones domésticas), no podrás tener buena entregabilidad desde esa IP: usa un relayhost (sección 6.8).

## 9.4 SPF (Sender Policy Framework)

SPF es un registro TXT que enumera qué servidores pueden enviar correo con tu dominio en el **remitente del sobre** (`MAIL FROM` / `Return-Path`).

Sintaxis:

| Elemento | Significado |
|---|---|
| `v=spf1` | Versión. Obligatorio al principio. |
| `mx` | Autoriza las IP de los servidores MX del dominio. |
| `a` | Autoriza la IP del registro A del dominio. |
| `a:mail.ejemplo1.com` | Autoriza la IP de ese nombre concreto. |
| `ip4:203.0.113.10` / `ip6:2001:db8:10::10` | Autoriza IP concretas. |
| `include:dominio` | Autoriza también los servidores que autorice el SPF de otro dominio (servicios externos). |
| `-all` | **Fallo**: el resto de IP no están autorizadas; el receptor debería rechazar. |
| `~all` | **Fallo suave**: no autorizadas, pero se aceptan marcadas como sospechosas. |
| `?all` | Neutral (no aporta nada). |

!!! warning "Atención"
    Solo puede haber **un** registro SPF (TXT que empieza por `v=spf1`) por nombre. Si añades servicios, se combinan en la misma línea. El SPF admite como máximo 10 consultas DNS (`mx`, `a`, `include`...) en total.

### PASO 9.3 – Crear los registros SPF

**Objetivo:** autorizar a nuestro servidor (y solo a él) a enviar correo de cada dominio.

Registros:

| Dominio | Tipo | Nombre | Valor |
|---|---|---|---|
| ejemplo1.com | TXT | @ | `v=spf1 mx ~all` |
| ejemplo1.com | TXT | mail | `v=spf1 a -all` |
| ejemplo2.com | TXT | @ | `v=spf1 mx ~all` |
| ejemplo2.com | TXT | mail | `v=spf1 a -all` |

Análisis:

- `v=spf1 mx ~all` en el dominio: autoriza a los servidores MX (`mail.ejemplo1.com` → 203.0.113.10, y su IPv6 si tiene AAAA). Empezamos con `~all` mientras verificamos que todo funciona; cuando DMARC confirme que todo pasa, puedes endurecerlo a `-all`.
- `v=spf1 a -all` en `mail`: protege el nombre del propio servidor, que se usa en el `EHLO` y en los correos del sistema (`@mail.ejemplo1.com`).

**Comprobación:** 

```bash title="Terminal"
dig +short TXT ejemplo1.com
dig +short TXT mail.ejemplo1.com
dig +short TXT ejemplo2.com
```

```text title="Salida esperada"
"v=spf1 mx ~all"
"v=spf1 a -all"
"v=spf1 mx ~all"
```

## 9.5 DKIM y OpenDKIM

**DKIM** (*DomainKeys Identified Mail*) añade a cada correo saliente una **firma digital** en la cabecera `DKIM-Signature`. El servidor firma con una clave **privada** que solo él tiene, y publica la clave **pública** en el DNS. El receptor descarga la clave pública y verifica la firma: si alguien modificó el mensaje por el camino, o lo envió otro servidor, la firma no cuadra.

```text title="Diagrama"
  Postfix ──milter──► OpenDKIM firma con /etc/opendkim/keys/ejemplo1.com/mail.private
                          │
                          ▼   DKIM-Signature: v=1; a=rsa-sha256; d=ejemplo1.com; s=mail; bh=...; b=...
  Gmail recibe ── lee d=ejemplo1.com y s=mail ── consulta TXT mail._domainkey.ejemplo1.com
                          │
                          └── verifica la firma con la clave pública → dkim=pass
```

- **Selector** (`s=mail`): nombre que identifica la clave. Permite tener varias claves (por ejemplo, para rotarlas). Usaremos el selector `mail`.
- **Milter** (*mail filter*): protocolo con el que Postfix pasa cada mensaje a un programa externo (OpenDKIM) antes de enviarlo.

### PASO 9.4 – Instalar OpenDKIM

**Comando:** 

```bash title="Terminal"
sudo apt install -y opendkim opendkim-tools
```

Análisis: `opendkim` es el servicio que firma y verifica; `opendkim-tools` incluye `opendkim-genkey` (generar claves) y `opendkim-testkey` (comprobar el DNS).

### PASO 9.5 – Generar las claves de cada dominio

**Objetivo:** crear un par de claves RSA de 2048 bits para cada dominio.

**Comando:** 

```bash title="Terminal"
sudo mkdir -p /etc/opendkim/keys/ejemplo1.com /etc/opendkim/keys/ejemplo2.com
sudo opendkim-genkey -b 2048 -d ejemplo1.com -D /etc/opendkim/keys/ejemplo1.com -s mail -v
sudo opendkim-genkey -b 2048 -d ejemplo2.com -D /etc/opendkim/keys/ejemplo2.com -s mail -v
sudo chown -R opendkim:opendkim /etc/opendkim
sudo chmod 700 /etc/opendkim/keys /etc/opendkim/keys/*
sudo chmod 600 /etc/opendkim/keys/*/mail.private
sudo ls -l /etc/opendkim/keys/ejemplo1.com/
```

**Análisis del comando:** 

| Opción | Significado |
|---|---|
| `-b 2048` | Longitud de la clave. 2048 bits es el estándar actual (1024 se considera débil). |
| `-d ejemplo1.com` | Dominio que firmará. |
| `-D carpeta` | Dónde guardar los archivos. |
| `-s mail` | Selector. |
| `-v` | Muestra lo que hace. |

**Resultado esperado:** 

```text title="Salida esperada"
-rw------- 1 opendkim opendkim 1679 nov  9 03:00 mail.private
-rw------- 1 opendkim opendkim  503 nov  9 03:00 mail.txt
```

`mail.private` es la clave privada (**nunca** la compartas ni la copies fuera del servidor salvo en copias de seguridad cifradas). `mail.txt` contiene el registro DNS que debes publicar.

### PASO 9.6 – Configurar las tablas de OpenDKIM

**Objetivo:** indicar qué clave usar para cada dominio y qué hosts son de confianza.

**Archivo:** `/etc/opendkim/key.table` (nuevo, contenido completo):

```text title="Archivo de configuración"
# nombre_de_la_clave                 dominio:selector:ruta_clave_privada
mail._domainkey.ejemplo1.com         ejemplo1.com:mail:/etc/opendkim/keys/ejemplo1.com/mail.private
mail._domainkey.ejemplo2.com         ejemplo2.com:mail:/etc/opendkim/keys/ejemplo2.com/mail.private
```

**Archivo:** `/etc/opendkim/signing.table` (nuevo, contenido completo):

```text title="Archivo de configuración"
# remitente                          clave con la que se firma
*@ejemplo1.com                       mail._domainkey.ejemplo1.com
*@ejemplo2.com                       mail._domainkey.ejemplo2.com
```

**Archivo:** `/etc/opendkim/trusted.hosts` (nuevo, contenido completo):

```text title="Archivo de configuración"
# Hosts cuyo correo se firma (correo saliente generado en el propio servidor)
127.0.0.1
::1
localhost
mail.ejemplo1.com
```

Análisis:

| Tabla | Para qué sirve |
|---|---|
| `key.table` | Asocia un nombre de clave con su dominio, selector y archivo de clave privada. |
| `signing.table` | Decide, según el remitente del mensaje, qué clave usar. `*@ejemplo1.com` = cualquier dirección del dominio. |
| `trusted.hosts` | El correo procedente de estas direcciones se considera **saliente** y se firma; el resto se considera entrante y se **verifica**. El correo de usuarios autenticados por SMTP (587/465) también se firma. |

```bash title="Terminal"
sudo chown opendkim:opendkim /etc/opendkim/*.table /etc/opendkim/trusted.hosts
```

### PASO 9.7 – Configurar OpenDKIM (opendkim.conf)

**Comando:** 

```bash title="Terminal"
sudo cp /etc/opendkim.conf /etc/opendkim.conf.original
sudo nano /etc/opendkim.conf
```

**Archivo:** `/etc/opendkim.conf` (sustituye todo el contenido):

```text title="Archivo de configuración"
# =====================================================================
#  /etc/opendkim.conf - Firma DKIM para ejemplo1.com y ejemplo2.com
# =====================================================================
# Registro en syslog (/var/log/mail.log)
Syslog                  yes
SyslogSuccess           yes
LogWhy                  yes

# Permisos de los archivos que crea y usuario del servicio
UMask                   007
UserID                  opendkim
PidFile                 /run/opendkim/opendkim.pid

# s = firmar el correo saliente, v = verificar el entrante
Mode                    sv
# Normalización de cabeceras/cuerpo antes de firmar (tolerante a cambios menores)
Canonicalization        relaxed/simple
SignatureAlgorithm      rsa-sha256
# Impide que se añada una segunda cabecera From: tras la firma
OversignHeaders         From

# Tablas de claves, de firma y de hosts de confianza
KeyTable                /etc/opendkim/key.table
SigningTable            refile:/etc/opendkim/signing.table
ExternalIgnoreList      refile:/etc/opendkim/trusted.hosts
InternalHosts           refile:/etc/opendkim/trusted.hosts
# El correo que Postfix marca como ORIGINATING (puertos 587/465) también se firma
MTA                     ORIGINATING

# Socket TCP solo local (Postfix funciona en chroot y no ve /run)
Socket                  inet:8891@localhost

# Raíz de confianza DNSSEC para validar las consultas DNS
TrustAnchorFile         /usr/share/dns/root.key
```

Análisis de las directivas principales:

| Directiva | Significado |
|---|---|
| `Mode sv` | Firma (*sign*) y verifica (*verify*). |
| `Canonicalization relaxed/simple` | `relaxed` en cabeceras (tolera cambios de espacios o mayúsculas que hacen algunos servidores), `simple` en el cuerpo. |
| `OversignHeaders From` | Protege contra la inserción de un segundo `From:` falso. |
| `refile:` | El archivo admite expresiones con comodines (`*@ejemplo1.com`). |
| `MTA ORIGINATING` | Firma el correo que llega por los servicios de Postfix con `milter_macro_daemon_name=ORIGINATING` (los puertos de envío). |
| `Socket inet:8891@localhost` | Escucha en `127.0.0.1:8891`, inaccesible desde fuera. |

### PASO 9.8 – Asegurar que el servicio usa el socket configurado

**Objetivo:** evitar que el arranque del servicio imponga otro socket distinto del de `opendkim.conf`.

**Explicación:** en Ubuntu, la unidad de systemd de OpenDKIM arranca el programa pasando el socket por línea de comandos (opción `-p`), y la línea de comandos **tiene prioridad** sobre `opendkim.conf`. Lo comprobamos y, para no depender de ello, creamos una configuración adicional (*drop-in*) de systemd que fija nuestro socket.

**Comando:** 

```bash title="Terminal"
systemctl cat opendkim | grep ExecStart
sudo mkdir -p /etc/systemd/system/opendkim.service.d
sudo tee /etc/systemd/system/opendkim.service.d/override.conf > /dev/null <<'EOF'
[Service]
ExecStart=
ExecStart=/usr/sbin/opendkim -P /run/opendkim/opendkim.pid -p inet:8891@localhost
EOF
sudo systemctl daemon-reload
sudo systemctl restart opendkim
sudo systemctl enable opendkim
sudo ss -tlnp | grep 8891
```

**Análisis del comando:** 

- `systemctl cat opendkim`: muestra la unidad del servicio; verás si incluye `-p local:/run/opendkim/opendkim.sock`.
- En el *drop-in*, la primera línea `ExecStart=` (vacía) **borra** el comando original; la segunda define el nuestro.
- `daemon-reload` hace que systemd relea las unidades.

**Resultado esperado:** 

```text title="Salida esperada"
LISTEN 0  4096  127.0.0.1:8891  0.0.0.0:*  users:(("opendkim",pid=8120,fd=3))
```

**Errores habituales:** si OpenDKIM no arranca, mira `sudo journalctl -u opendkim -n 30 --no-pager`. Mensajes típicos: `opendkim: /etc/opendkim/key.table: ... Permission denied` (revisa propietarios del PASO 9.5) o `can't load key from /etc/opendkim/keys/...: Permission denied`.

### PASO 9.9 – Conectar Postfix con OpenDKIM

**Archivo:** `/etc/postfix/main.cf`. Conserva todo y **añade al final**:

```text title="Archivo de configuración"
# --- Firma DKIM con OpenDKIM (Módulo 9) ---------------------------------
milter_default_action = accept
milter_protocol = 6
smtpd_milters = inet:localhost:8891
non_smtpd_milters = $smtpd_milters
```

Análisis:

| Parámetro | Significado |
|---|---|
| `milter_default_action = accept` | Si OpenDKIM no responde, el correo se acepta igualmente (sin firma) en lugar de bloquear todo el servidor. |
| `milter_protocol = 6` | Versión del protocolo milter. |
| `smtpd_milters` | Milters para el correo que llega por SMTP (puertos 25, 587, 465). |
| `non_smtpd_milters` | Milters para el correo generado localmente (`sendmail`, como el de PHP o cron). |

**Comando:** 

```bash title="Terminal"
sudo postfix check && sudo systemctl restart postfix
```

### PASO 9.10 – Publicar las claves DKIM en el DNS

**Objetivo:** publicar la clave pública de cada dominio.

**Comando:** muestra el contenido de los archivos generados:

```bash title="Terminal"
sudo cat /etc/opendkim/keys/ejemplo1.com/mail.txt
```

```text title="Salida esperada"
mail._domainkey	IN	TXT	( "v=DKIM1; h=sha256; k=rsa; "
	  "p=MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAx3Q...primera parte..."
	  "...segunda parte...IDAQAB" )  ; ----- DKIM key mail for ejemplo1.com
```

**Explicación:** un texto TXT no puede superar 255 caracteres por cadena, y una clave de 2048 bits es más larga, por eso el archivo la divide en varias cadenas entre comillas. En el panel DNS:

- **Nombre**: `mail._domainkey`
- **Tipo**: TXT
- **Valor**: une todas las partes entre comillas en una sola línea, **sin** las comillas ni los paréntesis: `v=DKIM1; h=sha256; k=rsa; p=MIIBIjANBgkq...IDAQAB`. La mayoría de paneles dividen el texto automáticamente. Si tu panel exige las cadenas separadas, pégalas con sus comillas tal como aparecen.

Este comando te muestra el valor ya unido, listo para copiar:

```bash title="Terminal"
sudo sh -c "tr -d '\n' < /etc/opendkim/keys/ejemplo1.com/mail.txt | sed -e 's/.*( *//' -e 's/ *).*//' -e 's/\"[[:space:]]*\"//g' -e 's/\"//g'"; echo
```

Repite para `ejemplo2.com` (archivo `/etc/opendkim/keys/ejemplo2.com/mail.txt`, nombre `mail._domainkey` en la zona de ejemplo2.com).

**Comprobación (cuando el registro se haya propagado):** 

```bash title="Terminal"
dig +short TXT mail._domainkey.ejemplo1.com
sudo opendkim-testkey -d ejemplo1.com -s mail -vvv
sudo opendkim-testkey -d ejemplo2.com -s mail -vvv
```

**Resultado esperado:** 

```text title="Salida esperada"
opendkim-testkey: using default configfile /etc/opendkim.conf
opendkim-testkey: checking key 'mail._domainkey.ejemplo1.com'
opendkim-testkey: key not secure
opendkim-testkey: key OK
```

`key OK` significa que la clave publicada coincide con la privada. `key not secure` solo indica que tu dominio no usa DNSSEC; no es un error.

**Errores habituales:** `record not found` (aún no propagado o nombre mal escrito); `key data does not match` (copiaste mal el valor o falta un fragmento).

## 9.6 DMARC

**DMARC** une SPF y DKIM con el remitente **visible** (`From:`). Un mensaje pasa DMARC si:

- pasa SPF **y** el dominio del `MAIL FROM` coincide (está *alineado*) con el del `From:`, **o**
- pasa DKIM **y** el dominio de la firma (`d=`) coincide con el del `From:`.

Además, el propietario del dominio publica qué debe hacerse con los mensajes que no pasan y a dónde enviar informes.

| Etiqueta | Significado | Valores |
|---|---|---|
| `v=DMARC1` | Versión (obligatoria, al principio). | |
| `p=` | Política para el dominio. | `none` (solo informar), `quarantine` (a spam), `reject` (rechazar). |
| `sp=` | Política para subdominios (si no se indica, usa `p`). | Igual que `p`. |
| `rua=` | Dirección para informes agregados diarios. | `mailto:dmarc@ejemplo1.com` |
| `adkim=` / `aspf=` | Alineación estricta (`s`) o relajada (`r`) de DKIM y SPF. | `r` permite subdominios. |
| `pct=` | Porcentaje de mensajes a los que se aplica la política. | 0–100 |

### PASO 9.11 – Crear los registros DMARC

Registros:

| Dominio | Tipo | Nombre | Valor |
|---|---|---|---|
| ejemplo1.com | TXT | _dmarc | `v=DMARC1; p=none; rua=mailto:dmarc@ejemplo1.com; adkim=r; aspf=r` |
| ejemplo2.com | TXT | _dmarc | `v=DMARC1; p=none; rua=mailto:dmarc@ejemplo2.com; adkim=r; aspf=r` |

Análisis: empezamos con `p=none` (modo observación). Durante unas semanas recibirás informes XML en `dmarc@` (que es alias de `administracion@`/`soporte@`, PASO 8.8) indicando qué IP envían en nombre de tu dominio y si pasan. Cuando todo pase, sube a `p=quarantine` y después a `p=reject`, que es la protección completa contra la suplantación de tu dominio.

**Comprobación:** 

```bash title="Terminal"
dig +short TXT _dmarc.ejemplo1.com
dig +short TXT _dmarc.ejemplo2.com
```

## 9.7 Comprobar la propagación de los registros

Cuando creas o cambias un registro, los resolvedores que tenían en caché la respuesta anterior la mantienen hasta que expira su TTL. Para saber si un cambio ya es visible:

```bash title="Terminal"
# Preguntar a varios resolvedores públicos
dig +short MX ejemplo1.com @1.1.1.1
dig +short MX ejemplo1.com @8.8.8.8
dig +short MX ejemplo1.com @9.9.9.9
# Preguntar directamente a los servidores autoritativos (la «fuente de verdad»)
dig +short NS ejemplo1.com
dig +short TXT _dmarc.ejemplo1.com @ns1.tuproveedordns.example
# Ver la cadena completa de resolución desde la raíz
dig +trace TXT mail._domainkey.ejemplo1.com
```

Análisis: si los autoritativos ya devuelven el valor nuevo pero `1.1.1.1` no, solo es cuestión de esperar (como máximo el TTL anterior). Si los autoritativos tampoco lo devuelven, el registro está mal creado en el panel. También existen herramientas web que consultan desde muchos países a la vez («DNS propagation checker»).

## 9.8 Verificación final de la autenticación

### PASO 9.12 – Verificar SPF, DKIM y DMARC con un correo real

**Objetivo:** confirmar que los servidores receptores dan por buenos los tres mecanismos.

**Comando:** envía un correo desde Thunderbird (o con swaks por el puerto 587) desde `info@ejemplo1.com` a tu cuenta de Gmail. Antes, comprueba en el log que OpenDKIM firma:

```bash title="Terminal"
sudo tail -f /var/log/mail.log | grep -i dkim
```

```text title="Salida esperada"
opendkim[8120]: 7B3C13E101: DKIM-Signature field added (s=mail, d=ejemplo1.com)
```

Comprobación en Gmail: abre el mensaje, pulsa los tres puntos → «Mostrar original». Debes ver:

```text title="Salida esperada"
SPF:   PASS con la IP 203.0.113.10
DKIM:  'PASS' con el dominio ejemplo1.com
DMARC: 'PASS'
```

Y en las cabeceras:

```text title="Salida esperada"
Authentication-Results: mx.google.com;
       dkim=pass header.i=@ejemplo1.com header.s=mail header.b=Ab3dEfGh;
       spf=pass (google.com: domain of info@ejemplo1.com designates 203.0.113.10 as permitted sender) smtp.mailfrom=info@ejemplo1.com;
       dmarc=pass (p=NONE sp=NONE dis=NONE) header.from=ejemplo1.com
```

Repite el envío desde `soporte@ejemplo2.com`.

Otras herramientas de verificación:

- **mail-tester.com**: te da una dirección temporal; envías un correo y te devuelve una puntuación sobre 10 con todos los problemas detectados (SPF, DKIM, DMARC, PTR, listas negras, contenido).
- **check-auth@verifier.port25.com**: si le envías un correo, responde con un informe detallado de SPF, DKIM y DMARC.

**Errores habituales:** 

| Resultado | Causa | Solución |
|---|---|---|
| `dkim=none` | OpenDKIM no firmó. | En el log no aparece `DKIM-Signature field added`: revisa `smtpd_milters`, el socket 8891 y la `signing.table`. |
| `dkim=fail (bad signature)` | La clave del DNS no corresponde a la privada, o el mensaje se modificó tras firmar. | `opendkim-testkey`; evita reescrituras posteriores (firmas de antivirus, pies añadidos). |
| `spf=softfail` / `spf=fail` | La IP de envío no está en el SPF (por ejemplo, se envió por IPv6 y el MX no tiene AAAA). | Añade `ip6:` o el registro AAAA, o usa `inet_protocols = ipv4`. |
| `dmarc=fail` | SPF y DKIM fallan o no están alineados con el `From:`. | Corrige primero DKIM, que es el mecanismo más robusto. |
| Gmail rechaza con `550-5.7.25 ... PTR record` | Falta PTR o no coincide (habitual en IPv6). | PASO 9.2 para IPv4 **e** IPv6. |

## 9.9 Reputación de la IP y entregabilidad

Tener los registros perfectos es necesario, pero no suficiente. Los grandes proveedores también valoran la **reputación**:

- **IP limpia**: antes de usar un servidor para correo, comprueba que su IP no está en listas negras (Spamhaus, Barracuda, SpamCop...). Hay comprobadores web que consultan decenas de listas a la vez (busca «multi RBL check»). Si tu IP está listada por un uso anterior, solicita la retirada en la web de la lista o pide otra IP al proveedor.
- **IP residencial**: las IP de conexiones domésticas aparecen en listas como la PBL de Spamhaus y su correo se rechaza casi siempre.
- **Calentamiento** (*warm-up*): una IP nueva no tiene historial. Empieza enviando poco volumen y a destinatarios que interactúan (abren, responden) y aumenta gradualmente.
- **Contenido y comportamiento**: sin listas compradas, con baja tasa de quejas, con enlace de baja en los envíos masivos, sin acortadores de URL sospechosos.
- **Paneles de los proveedores**: Google Postmaster Tools y Microsoft SNDS te muestran cómo ven tu dominio e IP.
- **Coherencia**: `myhostname` = PTR = registro A = nombre en el `EHLO`. Ya lo tenemos.

!!! tip "Consejo"
    Para envíos masivos (boletines con miles de destinatarios), lo recomendable es no usar el mismo servidor e IP que el correo personal: un problema de reputación con el boletín afectaría a todos los buzones.

## 9.10 Ejercicios prácticos del Módulo 9

1. Envía un correo a `check-auth@verifier.port25.com` desde cada dominio y analiza el informe que recibes.
2. Simula una suplantación: con swaks, conéctate al puerto 25 de Gmail desde otro servidor (no autorizado en tu SPF) usando `--from info@ejemplo1.com`. ¿Qué resultado de SPF y DMARC obtiene? (Solo con direcciones de tu propiedad.)
3. Cambia temporalmente el selector a uno inexistente en la `key.table` y observa qué ocurre con `dkim` en Gmail. Restaura la configuración.
4. Calcula cuántas consultas DNS consume tu registro SPF. ¿Qué pasaría si añadieras 11 `include:`?
5. Tras unos días con `p=none`, abre un informe DMARC recibido en `dmarc@ejemplo1.com` (es un XML comprimido) e identifica las IP y los resultados.
