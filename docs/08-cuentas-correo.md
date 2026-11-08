# Módulo 8. Creación de cuentas de correo

## Objetivos del módulo

- Diferenciar con claridad las cuentas del sistema Linux de las cuentas virtuales de correo.
- Entender qué archivos intervienen en la existencia de una dirección de correo.
- Crear las cuentas `info@ejemplo1.com`, `administracion@ejemplo1.com` y `soporte@ejemplo2.com`, y los alias obligatorios.
- Automatizar el alta, la baja y el cambio de contraseña con scripts de administración.
- Impedir que un usuario envíe correo haciéndose pasar por otro.
- Configurar un cliente de correo real.

## 8.1 Cuentas del sistema frente a cuentas virtuales

Es un error muy frecuente mezclar estos dos conceptos. En nuestra infraestructura están completamente separados:

| Característica | Cuenta del sistema Linux | Cuenta virtual de correo |
|---|---|---|
| Ejemplos | `sysadmin`, `web1`, `vmail`, `root` | `info@ejemplo1.com`, `soporte@ejemplo2.com` |
| Dónde se define | `/etc/passwd` y `/etc/shadow` | `/etc/dovecot/users` (Dovecot) y `/etc/postfix/vmailbox` (Postfix) |
| Se crea con | `adduser` / `useradd` | El script `correo-alta` (este módulo) |
| ¿Puede entrar por SSH? | Sí (si se permite) | **Nunca** |
| ¿Tiene carpeta en /home? | Normalmente sí | No. Su buzón está en `/var/mail/vhosts/dominio/usuario/` |
| Propietario de sus archivos | Ella misma | El usuario técnico `vmail` (uid 5000) |
| Nombre de usuario al autenticarse | `sysadmin` | La dirección completa: `info@ejemplo1.com` |
| Correo que recibe | El de `@mail.ejemplo1.com` (dominio local, `mydestination`) | El de `@ejemplo1.com` y `@ejemplo2.com` (dominios virtuales) |

Ventajas de las cuentas virtuales:

- **Seguridad**: robar la contraseña de un buzón no da acceso al servidor.
- **Multidominio**: `info@ejemplo1.com` e `info@ejemplo2.com` pueden coexistir sin conflicto (en Linux solo puede haber un usuario `info`).
- **Gestión sencilla**: crear 100 buzones no llena el sistema de usuarios Linux.

## 8.2 Qué hace falta para que una dirección exista

```text title="Diagrama"
 Llega un correo para soporte@ejemplo2.com
        │
        ▼
 POSTFIX ── ¿ejemplo2.com está en virtual_domains? ───────── no → "Relay access denied"
        │ sí
        ├── ¿hay alias en /etc/postfix/virtual? ── sí → se reescribe el destinatario
        │
        ├── ¿la dirección está en /etc/postfix/vmailbox? ─── no → "User unknown" (550)
        │ sí
        ▼ LMTP
 DOVECOT ── ¿la dirección está en /etc/dovecot/users? ────── no → rebote
        │ sí: uid 5000, home /var/mail/vhosts/ejemplo2.com/soporte
        ▼
 Guarda el mensaje en /var/mail/vhosts/ejemplo2.com/soporte/Maildir/new/

 Cuando soporte@ejemplo2.com envía por el 587:
 POSTFIX ──SASL──► DOVECOT (/etc/dovecot/users: ¿contraseña correcta?)
        └── /etc/postfix/sender_login: ¿soporte@ puede usar este remitente?
```

Resumen de archivos:

| Archivo | Propietario del dato | Qué contiene | Tras modificarlo |
|---|---|---|---|
| `/etc/postfix/virtual_domains` | Postfix | Dominios de correo. | `postmap` |
| `/etc/postfix/vmailbox` | Postfix | Direcciones con buzón. | `postmap` |
| `/etc/postfix/virtual` | Postfix | Alias (reenvíos). | `postmap` |
| `/etc/postfix/sender_login` | Postfix | Qué usuario autenticado puede usar cada remitente. | `postmap` |
| `/etc/dovecot/users` | Dovecot | Contraseñas, uid, gid y carpeta de cada cuenta. | Nada (Dovecot lo relee solo) |

## 8.3 Proteger la identidad del remitente

Ahora mismo, cualquier usuario autenticado podría enviar un correo con `MAIL FROM:<administracion@ejemplo1.com>` aunque haya iniciado sesión como `info@`. Lo impedimos con una tabla que indica qué usuario puede usar cada dirección como remitente.

### PASO 8.1 – Crear la tabla de remitentes autorizados

**Archivo:** `/etc/postfix/sender_login` (nuevo):

```text title="Archivo de configuración"
# Remitente (MAIL FROM)            Usuarios SASL autorizados a usarlo (separados por comas)
info@ejemplo1.com                  info@ejemplo1.com
```

**Comando:** 

```bash title="Terminal"
sudo postmap /etc/postfix/sender_login
```

Añade a `/etc/postfix/main.cf` (conserva todo lo anterior y añade al final):

```text title="Archivo de configuración"
# --- Remitentes autorizados por usuario (Módulo 8) ----------------------
smtpd_sender_login_maps = hash:/etc/postfix/sender_login
```

### PASO 8.2 – Exigir la coincidencia en los puertos de envío

**Objetivo:** que en los puertos 587 y 465 se rechace un remitente que no pertenezca al usuario autenticado.

**Archivo:** `/etc/postfix/master.cf`. En los bloques `submission` y `smtps` que escribiste en el PASO 7.14, añade la línea `-o smtpd_sender_restrictions=...` justo debajo de la línea `-o smtpd_client_restrictions=...`. Los dos bloques quedan así (el resto del archivo no cambia):

```text title="Archivo de configuración"
submission inet n       -       y       -       -       smtpd
  -o syslog_name=postfix/submission
  -o smtpd_tls_security_level=encrypt
  -o smtpd_sasl_auth_enable=yes
  -o smtpd_tls_auth_only=yes
  -o smtpd_client_restrictions=permit_sasl_authenticated,reject
  -o smtpd_sender_restrictions=reject_sender_login_mismatch,permit_sasl_authenticated,reject
  -o smtpd_relay_restrictions=permit_sasl_authenticated,reject
  -o milter_macro_daemon_name=ORIGINATING
smtps     inet  n       -       y       -       -       smtpd
  -o syslog_name=postfix/smtps
  -o smtpd_tls_wrappermode=yes
  -o smtpd_sasl_auth_enable=yes
  -o smtpd_client_restrictions=permit_sasl_authenticated,reject
  -o smtpd_sender_restrictions=reject_sender_login_mismatch,permit_sasl_authenticated,reject
  -o smtpd_relay_restrictions=permit_sasl_authenticated,reject
  -o milter_macro_daemon_name=ORIGINATING
```

Análisis: `reject_sender_login_mismatch` rechaza el mensaje si el usuario autenticado no figura en `sender_login` como propietario del remitente indicado en `MAIL FROM`.

```bash title="Terminal"
sudo postfix check && sudo systemctl restart postfix
```

**Comprobación:** más adelante, cuando exista `administracion@`, intenta enviar como ella autenticado como `info@`:

```bash title="Terminal"
swaks --server mail.ejemplo1.com --port 587 --tls --auth LOGIN --auth-user info@ejemplo1.com --from administracion@ejemplo1.com --to tu-direccion@gmail.com
```

```text title="Salida esperada"
<** 553 5.7.1 <administracion@ejemplo1.com>: Sender address rejected: not owned by user info@ejemplo1.com
```

## 8.4 Scripts de administración de cuentas

Hacer las altas a mano en cuatro archivos es lento y propenso a errores (una cuenta en Postfix pero no en Dovecot provoca rebotes). Un administrador automatiza las tareas repetitivas. Crearemos tres scripts en `/usr/local/sbin/`, la carpeta estándar para herramientas de administración propias (solo root puede ejecutarlas con sentido, porque modifican archivos protegidos).

### PASO 8.3 – Script de alta de cuentas: correo-alta

**Comando:** 

```bash title="Terminal"
sudo nano /usr/local/sbin/correo-alta
```

**Archivo:** `/usr/local/sbin/correo-alta` (nuevo, contenido completo):

```bash title="Terminal"
#!/bin/bash
# =====================================================================
#  correo-alta  -  Crea una cuenta de correo virtual
#  Uso: sudo correo-alta usuario@dominio
# =====================================================================
set -euo pipefail

USERS=/etc/dovecot/users
VMAILBOX=/etc/postfix/vmailbox
DOMINIOS=/etc/postfix/virtual_domains
SENDERS=/etc/postfix/sender_login
VMAIL_UID=5000
VMAIL_GID=5000
BASE=/var/mail/vhosts

[ "$(id -u)" -eq 0 ] || { echo "Error: ejecuta este script con sudo." >&2; exit 1; }
[ $# -eq 1 ] || { echo "Uso: $0 usuario@dominio" >&2; exit 1; }

DIRECCION="${1,,}"                       # a minúsculas
if ! [[ "$DIRECCION" =~ ^[a-z0-9._-]+@[a-z0-9.-]+\.[a-z]{2,}$ ]]; then
    echo "Error: '$DIRECCION' no es una dirección válida." >&2; exit 1
fi
USUARIO="${DIRECCION%@*}"
DOMINIO="${DIRECCION#*@}"

# El dominio debe estar dado de alta en Postfix
if ! awk -v d="$DOMINIO" '$1==d {f=1} END {exit !f}' "$DOMINIOS"; then
    echo "Error: el dominio $DOMINIO no está en $DOMINIOS." >&2; exit 1
fi
# La cuenta no debe existir
if awk -F: -v a="$DIRECCION" '$1==a {f=1} END {exit !f}' "$USERS"; then
    echo "Error: la cuenta $DIRECCION ya existe." >&2; exit 1
fi

echo "Introduce la contraseña para $DIRECCION (mínimo 12 caracteres):"
HASH=$(doveadm pw -s SHA512-CRYPT)

# Copias de seguridad antes de modificar
cp -p "$USERS" "$USERS.bak"; cp -p "$VMAILBOX" "$VMAILBOX.bak"; cp -p "$SENDERS" "$SENDERS.bak"

echo "${DIRECCION}:${HASH}:${VMAIL_UID}:${VMAIL_GID}::${BASE}/${DOMINIO}/${USUARIO}::" >> "$USERS"
printf '%-35s %s\n' "$DIRECCION" "${DOMINIO}/${USUARIO}/" >> "$VMAILBOX"
printf '%-35s %s\n' "$DIRECCION" "$DIRECCION" >> "$SENDERS"

postmap "$VMAILBOX"
postmap "$SENDERS"

echo "Cuenta $DIRECCION creada correctamente."
echo "El buzón se creará en ${BASE}/${DOMINIO}/${USUARIO}/Maildir al recibir el primer correo o iniciar sesión."
```

Análisis del script:

| Línea | Qué hace |
|---|---|
| `#!/bin/bash` | Indica que el script se ejecuta con bash. |
| `set -euo pipefail` | Detiene el script ante cualquier error (`-e`), variable no definida (`-u`) o fallo en una tubería (`pipefail`). Evita dejar las cosas a medias. |
| `${1,,}` | Convierte el argumento a minúsculas. |
| `[[ ... =~ ... ]]` | Comprueba con una expresión regular que la dirección tiene un formato válido. |
| `${DIRECCION%@*}` / `${DIRECCION#*@}` | Extrae la parte antes y después de la `@`. |
| `awk -v d=... '$1==d'` | Busca coincidencias exactas en la primera columna (más seguro que `grep`, donde el punto significa «cualquier carácter»). |
| `doveadm pw` | Pide la contraseña dos veces y devuelve su hash. La contraseña nunca se escribe en disco ni en el historial. |
| `cp -p ... .bak` | Copia de seguridad conservando permisos. |
| `>>` | Añade una línea al final del archivo (nunca uses `>` aquí: borraría el contenido). |
| `postmap` | Recompila las tablas de Postfix. |

Hazlo ejecutable:

```bash title="Terminal"
sudo chmod 750 /usr/local/sbin/correo-alta
```

### PASO 8.4 – Script de cambio de contraseña: correo-clave

**Archivo:** `/usr/local/sbin/correo-clave` (nuevo, contenido completo):

```bash title="Terminal"
#!/bin/bash
# =====================================================================
#  correo-clave  -  Cambia la contraseña de una cuenta de correo virtual
#  Uso: sudo correo-clave usuario@dominio
# =====================================================================
set -euo pipefail
USERS=/etc/dovecot/users

[ "$(id -u)" -eq 0 ] || { echo "Error: ejecuta este script con sudo." >&2; exit 1; }
[ $# -eq 1 ] || { echo "Uso: $0 usuario@dominio" >&2; exit 1; }
DIRECCION="${1,,}"

if ! awk -F: -v a="$DIRECCION" '$1==a {f=1} END {exit !f}' "$USERS"; then
    echo "Error: la cuenta $DIRECCION no existe." >&2; exit 1
fi

echo "Nueva contraseña para $DIRECCION:"
HASH=$(doveadm pw -s SHA512-CRYPT)

cp -p "$USERS" "$USERS.bak"
TMP=$(mktemp)
awk -F: -v OFS=: -v a="$DIRECCION" -v h="$HASH" '$1==a {$2=h} {print}' "$USERS" > "$TMP"
cat "$TMP" > "$USERS"      # conserva propietario y permisos del archivo original
rm -f "$TMP"

# Cierra las sesiones abiertas para que vuelvan a autenticarse
doveadm kick "$DIRECCION" >/dev/null 2>&1 || true
echo "Contraseña de $DIRECCION actualizada."
```

Análisis: `awk -F: -v OFS=:` lee y escribe el archivo con `:` como separador y sustituye solo el segundo campo de la línea de esa cuenta. Escribir con `cat "$TMP" > "$USERS"` (en lugar de mover el archivo) conserva el propietario `root:dovecot` y los permisos `640`. `doveadm kick` desconecta las sesiones IMAP/POP3 activas de ese usuario.

### PASO 8.5 – Script de baja de cuentas: correo-baja

**Archivo:** `/usr/local/sbin/correo-baja` (nuevo, contenido completo):

```bash title="Terminal"
#!/bin/bash
# =====================================================================
#  correo-baja  -  Elimina una cuenta de correo virtual
#  Uso: sudo correo-baja usuario@dominio
#  El buzón NO se borra: se mueve a /var/mail/bajas para poder recuperarlo.
# =====================================================================
set -euo pipefail
USERS=/etc/dovecot/users
VMAILBOX=/etc/postfix/vmailbox
SENDERS=/etc/postfix/sender_login
BASE=/var/mail/vhosts
BAJAS=/var/mail/bajas

[ "$(id -u)" -eq 0 ] || { echo "Error: ejecuta este script con sudo." >&2; exit 1; }
[ $# -eq 1 ] || { echo "Uso: $0 usuario@dominio" >&2; exit 1; }
DIRECCION="${1,,}"
USUARIO="${DIRECCION%@*}"
DOMINIO="${DIRECCION#*@}"

if ! awk -F: -v a="$DIRECCION" '$1==a {f=1} END {exit !f}' "$USERS"; then
    echo "Error: la cuenta $DIRECCION no existe." >&2; exit 1
fi

read -r -p "¿Seguro que quieres dar de baja $DIRECCION? Escribe SI para confirmar: " RESP
[ "$RESP" = "SI" ] || { echo "Cancelado."; exit 0; }

for F in "$USERS" "$VMAILBOX" "$SENDERS"; do
    cp -p "$F" "$F.bak"
    TMP=$(mktemp)
    if [ "$F" = "$USERS" ]; then
        awk -F: -v a="$DIRECCION" '$1!=a' "$F" > "$TMP"
    else
        awk -v a="$DIRECCION" '$1!=a' "$F" > "$TMP"
    fi
    cat "$TMP" > "$F"; rm -f "$TMP"
done
postmap "$VMAILBOX"
postmap "$SENDERS"
doveadm kick "$DIRECCION" >/dev/null 2>&1 || true

if [ -d "$BASE/$DOMINIO/$USUARIO" ]; then
    install -d -o vmail -g vmail -m 750 "$BAJAS"
    mv "$BASE/$DOMINIO/$USUARIO" "$BAJAS/${DIRECCION}_$(date +%F)"
    echo "Buzón movido a $BAJAS/${DIRECCION}_$(date +%F)"
fi
echo "Cuenta $DIRECCION dada de baja. Revisa si tenía alias en /etc/postfix/virtual."
```

```bash title="Terminal"
sudo chmod 750 /usr/local/sbin/correo-clave /usr/local/sbin/correo-baja
ls -l /usr/local/sbin/correo-*
```

!!! warning "Atención"
    Los scripts guardan una copia `.bak` de cada archivo antes de modificarlo. Si algo sale mal, puedes restaurarla con `sudo cp -p /etc/dovecot/users.bak /etc/dovecot/users` (y lo mismo con los demás archivos, seguido de `postmap`).

## 8.5 Crear las cuentas del proyecto

### PASO 8.6 – Registrar la cuenta info@ creada a mano

**Objetivo:** añadir la cuenta del Módulo 7 a la tabla de remitentes (se creó antes de existir el script).

**Comando:** 

```bash title="Terminal"
grep -q '^info@ejemplo1.com' /etc/postfix/sender_login || echo 'info@ejemplo1.com                  info@ejemplo1.com' | sudo tee -a /etc/postfix/sender_login
sudo postmap /etc/postfix/sender_login
```

### PASO 8.7 – Crear administracion@ejemplo1.com y soporte@ejemplo2.com

**Comando:** 

```bash title="Terminal"
sudo correo-alta administracion@ejemplo1.com
sudo correo-alta soporte@ejemplo2.com
```

**Resultado esperado:** 

```text title="Salida esperada"
Introduce la contraseña para administracion@ejemplo1.com (mínimo 12 caracteres):
Enter new password:
Retype new password:
Cuenta administracion@ejemplo1.com creada correctamente.
El buzón se creará en /var/mail/vhosts/ejemplo1.com/administracion/Maildir al recibir el primer correo o iniciar sesión.
```

**Comprobación:** 

```bash title="Terminal"
sudo doveadm user '*'
sudo doveadm auth test soporte@ejemplo2.com
postmap -q soporte@ejemplo2.com hash:/etc/postfix/vmailbox
```

```text title="Salida esperada"
administracion@ejemplo1.com
info@ejemplo1.com
soporte@ejemplo2.com
Password:
passdb: soporte@ejemplo2.com auth succeeded
ejemplo2.com/soporte/
```

Análisis: `doveadm user '*'` lista todos los usuarios que conoce Dovecot. `postmap -q clave tabla` consulta una tabla de Postfix exactamente igual que lo haría Postfix al recibir un correo: es la mejor forma de comprobar las tablas.

### PASO 8.8 – Crear los alias obligatorios y de servicio

**Objetivo:** crear las direcciones que todo dominio debe tener y las que usaremos en módulos siguientes.

**Explicación:** los estándares (RFC 2142 y RFC 5321) exigen que exista `postmaster@` en todo dominio que recibe correo, y es muy recomendable `abuse@`. También crearemos `dmarc@` para los informes DMARC (Módulo 9) y `noreply@`, el remitente que usarán las aplicaciones web (Módulo 11), para que cualquier rebote o respuesta llegue a una persona.

**Archivo:** `/etc/postfix/virtual` (sustituye todo el contenido):

```text title="Archivo de configuración"
# =====================================================================
#  Alias virtuales:  dirección_alias        destino[, destino2...]
#  Tras modificar:   sudo postmap /etc/postfix/virtual
# =====================================================================

# ---- ejemplo1.com ----
postmaster@ejemplo1.com      administracion@ejemplo1.com
abuse@ejemplo1.com           administracion@ejemplo1.com
dmarc@ejemplo1.com           administracion@ejemplo1.com
noreply@ejemplo1.com         administracion@ejemplo1.com
root@ejemplo1.com            administracion@ejemplo1.com
# Ejemplo de lista de distribución: llega a dos buzones
equipo@ejemplo1.com          info@ejemplo1.com, administracion@ejemplo1.com

# ---- ejemplo2.com ----
postmaster@ejemplo2.com      soporte@ejemplo2.com
abuse@ejemplo2.com           soporte@ejemplo2.com
dmarc@ejemplo2.com           soporte@ejemplo2.com
noreply@ejemplo2.com         soporte@ejemplo2.com
```

**Comando:** 

```bash title="Terminal"
sudo postmap /etc/postfix/virtual
postmap -q equipo@ejemplo1.com hash:/etc/postfix/virtual
```

```text title="Salida esperada"
info@ejemplo1.com, administracion@ejemplo1.com
```

!!! tip "Consejo"
    Un alias **no** es una cuenta: no tiene contraseña ni buzón propio. Si quieres que `administracion@` pueda **enviar** como `postmaster@ejemplo1.com` o `noreply@ejemplo1.com`, añade esa relación a `/etc/postfix/sender_login`, por ejemplo: `postmaster@ejemplo1.com  administracion@ejemplo1.com` (y `postmap`).

!!! warning "Atención"
    Evita los alias *catch-all* (`@ejemplo1.com  info@ejemplo1.com`), que recogen el correo de cualquier dirección inventada. Atraen enormes cantidades de spam y anulan el rechazo de direcciones inexistentes.

### PASO 8.9 – Redirigir el correo del sistema a una cuenta real

**Objetivo:** que los avisos del servidor (cron, Fail2ban, actualizaciones) lleguen a `administracion@ejemplo1.com`.

**Archivo:** `/etc/aliases` (modifica solo la última línea; el resto se conserva):

```text title="Archivo de configuración"
postmaster:    root
abuse:         root
webmaster:     root
www-data:      root
root:          administracion@ejemplo1.com
```

**Comando:** 

```bash title="Terminal"
sudo newaliases
echo "Aviso de prueba del sistema" | mail -s "Prueba correo de root" root
sudo grep 'orig_to=<root>' /var/log/mail.log | tail -1
```

**Resultado esperado:** una línea con `to=<administracion@ejemplo1.com>, orig_to=<root>, relay=...dovecot-lmtp..., status=sent`.

## 8.6 Gestión de buzones y permisos

### PASO 8.10 – Verificar la estructura y los permisos de los buzones

**Comando:** 

```bash title="Terminal"
for d in info@ejemplo1.com administracion@ejemplo1.com soporte@ejemplo2.com; do
  echo "Bienvenida a $d" | mail -s "Bienvenida" "$d"
done
sleep 2
sudo ls -l /var/mail/vhosts/ /var/mail/vhosts/ejemplo1.com/
sudo doveadm mailbox status -u soporte@ejemplo2.com messages INBOX
```

Análisis: el bucle `for` envía un correo a cada cuenta, lo que provoca la creación de su buzón. `doveadm mailbox status` muestra cuántos mensajes hay en una carpeta.

**Resultado esperado:** 

```text title="Salida esperada"
drwx------ 3 vmail vmail 4096 nov  9 02:10 ejemplo1.com
drwx------ 3 vmail vmail 4096 nov  9 02:10 ejemplo2.com
drwx------ 3 vmail vmail 4096 nov  9 02:10 administracion
drwx------ 3 vmail vmail 4096 nov  9 01:40 info
INBOX messages=1
```

Todo pertenece a `vmail` con permisos `700`: ni las webs (`web1`, `web2`) ni Apache pueden leer el correo.

Otras operaciones útiles con buzones:

| Tarea | Comando |
|---|---|
| Espacio usado por cada buzón | `sudo du -sh /var/mail/vhosts/*/*` |
| Carpetas de un buzón | `sudo doveadm mailbox list -u info@ejemplo1.com` |
| Buscar mensajes de un remitente | `sudo doveadm search -u info@ejemplo1.com mailbox INBOX from pepe@gmail.com` |
| Vaciar la papelera | `sudo doveadm expunge -u info@ejemplo1.com mailbox Trash all` |
| Borrar mensajes de más de 30 días en Spam | `sudo doveadm expunge -u info@ejemplo1.com mailbox Junk savedbefore 30d` |
| Reconstruir índices dañados | `sudo doveadm force-resync -u info@ejemplo1.com '*'` |

!!! tip "Consejo"
    **Cuotas.** Dovecot puede limitar el tamaño de cada buzón con el plugin `quota`, añadiendo en el campo 8 de `/etc/dovecot/users` algo como `userdb_quota_rule=*:storage=2G` y activando el plugin en la configuración. Es una ampliación recomendable cuando tengas muchos usuarios; consulta la documentación de Dovecot 2.3 sobre «Quota».

## 8.7 Configurar un cliente de correo

### PASO 8.11 – Configurar Thunderbird (o cualquier cliente)

**Objetivo:** usar la cuenta desde un programa de correo real.

Datos de configuración para los usuarios de **ejemplo1.com**:

| Parámetro | Entrante (IMAP) | Saliente (SMTP) |
|---|---|---|
| Servidor | `mail.ejemplo1.com` | `mail.ejemplo1.com` |
| Puerto | 993 | 587 (o 465) |
| Seguridad de la conexión | SSL/TLS | STARTTLS (con 587) o SSL/TLS (con 465) |
| Método de autenticación | Contraseña normal | Contraseña normal |
| Nombre de usuario | La dirección completa: `info@ejemplo1.com` | La dirección completa |

Para los usuarios de **ejemplo2.com** los datos son idénticos, pero con el servidor `mail.ejemplo2.com` (el certificado incluye ambos nombres, así que no habrá avisos).

Si prefieres POP3: servidor `mail.ejemplo1.com`, puerto 995, SSL/TLS.

En Thunderbird: «Cuentas» → «Nueva cuenta de correo» → introduce nombre, dirección y contraseña → pulsa «Configurar manualmente» y escribe los valores de la tabla → «Volver a probar» → «Hecho».

**Comprobación:** 

1. Debe aparecer la bandeja de entrada con el mensaje «Bienvenida».
2. Envía un correo a tu dirección de Gmail y comprueba que llega (aunque aún pueda ir a spam).
3. Responde desde Gmail y comprueba que la respuesta llega a Thunderbird. Para que esto funcione con la dirección `@ejemplo1.com` necesitas el registro **MX** (Módulo 9).
4. En el servidor, mira el log mientras envías: `sudo tail -f /var/log/mail.log | grep -E 'submission|smtps|sasl'`. Verás `sasl_method=PLAIN, sasl_username=info@ejemplo1.com`.

**Errores habituales:** 

| Síntoma en el cliente | Causa | Solución |
|---|---|---|
| «No se puede conectar» | Puerto cerrado en UFW o en el cortafuegos del proveedor. | `sudo ufw status`; prueba con `openssl s_client`. |
| «Usuario o contraseña incorrectos» | Usaste solo `info` en vez de `info@ejemplo1.com`. | El usuario siempre es la dirección completa. |
| Aviso de certificado no válido | Usaste la IP o un nombre que no está en el certificado. | Usa `mail.ejemplo1.com` o `mail.ejemplo2.com`. |
| Recibe pero no envía | El cliente usa el puerto 25. | Cambia a 587/STARTTLS o 465/SSL. |
| «Sender address rejected: not owned by user» | El remitente configurado en el cliente no coincide con el usuario. | Corrige la dirección del remitente o añádela en `sender_login`. |

## 8.8 Ejercicios prácticos del Módulo 8

1. Crea la cuenta `ventas@ejemplo2.com`, comprueba que recibe correo y después dala de baja con `correo-baja`. Verifica dónde ha quedado su buzón.
2. Cambia la contraseña de `info@ejemplo1.com` con `correo-clave` y comprueba que la antigua ya no funciona con `doveadm auth test`.
3. Crea un alias `facturas@ejemplo1.com` que entregue en `administracion@` y autoriza a `administracion@` a enviar como `facturas@`. Pruébalo con swaks.
4. Intenta crear una cuenta en un dominio no registrado (`prueba@ejemplo3.com`). ¿Qué hace el script? ¿Qué tendrías que modificar para alojar un tercer dominio de correo?
5. Explica por qué la cuenta Linux `web1` no puede leer los correos de `info@ejemplo1.com` (pista: `ls -ld /var/mail/vhosts`).
