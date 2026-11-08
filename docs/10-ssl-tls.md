# Módulo 10. Certificados SSL/TLS

## Objetivos del módulo

- Comprender cómo funciona TLS y qué es exactamente un certificado.
- Conocer Let's Encrypt, el protocolo ACME y los métodos de validación.
- Entender la estructura de archivos de Certbot y qué certificado usa cada servicio.
- Configurar la renovación automática y la recarga de Apache, Postfix y Dovecot tras cada renovación.
- Verificar los certificados de todos los servicios y los parámetros de seguridad TLS.

## 10.1 Cómo funciona TLS

**TLS** (*Transport Layer Security*, sucesor del antiguo SSL) protege una conexión de tres maneras:

| Propiedad | Qué garantiza | Cómo |
|---|---|---|
| **Confidencialidad** | Nadie en el camino puede leer los datos. | Cifrado simétrico (AES, ChaCha20) con una clave de sesión. |
| **Integridad** | Nadie puede modificar los datos sin que se detecte. | Códigos de autenticación de mensajes. |
| **Autenticidad** | El servidor es quien dice ser. | Certificado firmado por una autoridad de confianza. |

Un **certificado** es un documento digital que dice: «la clave pública X pertenece a `www.ejemplo1.com`», firmado por una **Autoridad de Certificación** (CA). Los navegadores y sistemas operativos traen una lista de CA de confianza (en Ubuntu, en `/etc/ssl/certs/`).

```text title="Diagrama"
  Cadena de confianza
  ┌──────────────────────────────┐
  │ CA raíz ISRG Root X1         │ ← preinstalada en navegadores y sistemas
  └──────────────┬───────────────┘
                 │ firma
  ┌──────────────▼───────────────┐
  │ CA intermedia de Let's Encrypt│ ← se envía junto al certificado (chain)
  └──────────────┬───────────────┘
                 │ firma
  ┌──────────────▼───────────────┐
  │ www.ejemplo1.com             │ ← tu certificado (cert)
  │ clave pública + nombres SAN  │    su clave privada (privkey) nunca sale del servidor
  └──────────────────────────────┘
```

El saludo TLS (*handshake*), simplificado:

1. El cliente saluda, indica qué versiones y cifrados admite y el nombre al que quiere conectar (SNI).
2. El servidor responde con su certificado y la cadena intermedia.
3. El cliente verifica la cadena hasta una CA raíz de confianza, que el nombre está en el certificado y que no ha caducado.
4. Ambos acuerdan una clave de sesión mediante un intercambio Diffie-Hellman efímero (ECDHE). El servidor demuestra que posee la clave privada.
5. A partir de ahí, todo viaja cifrado con la clave de sesión.

## 10.2 Let's Encrypt y ACME

**Let's Encrypt** es una CA gratuita, automatizada y abierta. Sus certificados:

- Son válidos durante **90 días** (corta duración para limitar los daños si una clave se compromete y para obligar a automatizar).
- Se obtienen con el protocolo **ACME**, que demuestra que controlas el dominio.
- Pueden cubrir varios nombres (campo SAN, *Subject Alternative Name*).

Métodos de validación:

| Método | Cómo se demuestra el control | Cuándo usarlo |
|---|---|---|
| **HTTP-01** | Publicando un archivo en `http://dominio/.well-known/acme-challenge/`. | El habitual. Requiere el puerto 80 abierto. **Lo usamos con `--webroot`.** |
| DNS-01 | Creando un registro TXT `_acme-challenge.dominio`. | Certificados comodín (`*.ejemplo1.com`) o servidores sin puerto 80. Requiere un plugin para la API de tu DNS. |
| TLS-ALPN-01 | Por el puerto 443 con un protocolo especial. | Casos muy específicos. |

## 10.3 Los certificados de nuestra infraestructura

| Certificado (nombre en Certbot) | Nombres que cubre | Lo usa | Obtenido en |
|---|---|---|---|
| `ejemplo1.com` | ejemplo1.com, www.ejemplo1.com | Apache: VirtualHost 443 de ejemplo1 | PASO 3.11 |
| `ejemplo2.com` | ejemplo2.com, www.ejemplo2.com | Apache: VirtualHost 443 de ejemplo2 | PASO 3.11 |
| `mail.ejemplo1.com` | mail.ejemplo1.com, mail.ejemplo2.com | Postfix (25, 587, 465) y Dovecot (993, 995) | PASO 6.2 |

!!! tip "Consejo"
    **Coherencia entre servicios.** Un cliente solo acepta el certificado si el nombre al que se conecta está dentro de él. Por eso los clientes de correo deben usar `mail.ejemplo1.com` o `mail.ejemplo2.com` (nunca la IP ni `www`), y Postfix y Dovecot comparten el mismo certificado. Apache usa certificados distintos para cada web porque cada VirtualHost presenta el suyo gracias a SNI.

### Estructura de archivos de Certbot

```text title="Diagrama"
/etc/letsencrypt/
├── live/                         ← SIEMPRE apunta a la versión vigente (enlaces simbólicos)
│   └── mail.ejemplo1.com/
│       ├── cert.pem      → ../../archive/mail.ejemplo1.com/cert3.pem     certificado
│       ├── chain.pem     → ../../archive/mail.ejemplo1.com/chain3.pem    intermedios
│       ├── fullchain.pem → ../../archive/mail.ejemplo1.com/fullchain3.pem cert + intermedios
│       └── privkey.pem   → ../../archive/mail.ejemplo1.com/privkey3.pem  clave privada
├── archive/                      ← todas las versiones (cert1, cert2, cert3...)
├── renewal/
│   └── mail.ejemplo1.com.conf    ← cómo se obtuvo y cómo renovarlo (método, webroot...)
├── renewal-hooks/
│   ├── pre/                      ← scripts que se ejecutan antes de renovar
│   ├── deploy/                   ← scripts que se ejecutan tras CADA renovación con éxito
│   └── post/                     ← scripts que se ejecutan después de intentar renovar
└── accounts/                     ← tu cuenta ACME
```

!!! warning "Atención"
    Configura siempre los servicios con las rutas de `live/` y con `fullchain.pem` (no `cert.pem`): así, tras cada renovación, solo hay que recargar el servicio. Si usas `cert.pem` sin la cadena, muchos clientes (especialmente de correo y móviles) darán error de certificado.

Consulta tus certificados en cualquier momento:

```bash title="Terminal"
sudo certbot certificates
```

```text title="Salida esperada"
Found the following certs:
  Certificate Name: ejemplo1.com
    Domains: ejemplo1.com www.ejemplo1.com
    Expiry Date: 2027-02-06 22:41:03+00:00 (VALID: 89 days)
    Certificate Path: /etc/letsencrypt/live/ejemplo1.com/fullchain.pem
    Private Key Path: /etc/letsencrypt/live/ejemplo1.com/privkey.pem
  Certificate Name: ejemplo2.com
    ...
  Certificate Name: mail.ejemplo1.com
    Domains: mail.ejemplo1.com mail.ejemplo2.com
    ...
```

## 10.4 Instalación de Certbot: apt o snap

En el Módulo 3 instalamos Certbot desde los repositorios de Ubuntu (`apt install certbot`). Es una versión estable y se actualiza con el resto del sistema. El proyecto Certbot recomienda oficialmente instalarlo con **snap** para tener siempre la última versión. Ambas opciones son válidas; **no las mezcles**. Si quisieras pasar a snap:

```bash title="Terminal"
sudo apt remove certbot
sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot
```

La configuración de `/etc/letsencrypt` se conserva, y la renovación pasa a gestionarla el temporizador del snap (`snap.certbot.renew.timer`).

## 10.5 Renovación automática

### PASO 10.1 – Comprobar el temporizador de renovación

**Objetivo:** verificar que el sistema intenta renovar los certificados automáticamente.

**Explicación:** el paquete `certbot` instala un temporizador de systemd que ejecuta `certbot renew` dos veces al día. `renew` solo renueva los certificados a los que les quedan menos de 30 días; el resto los deja intactos.

**Comando:** 

```bash title="Terminal"
systemctl list-timers | grep -i certbot
systemctl cat certbot.service | grep ExecStart
```

**Resultado esperado:** 

```text title="Salida esperada"
Mon 2026-11-09 11:43:00 CET  8h left  Sun 2026-11-08 23:12:01 CET  2h ago  certbot.timer  certbot.service
ExecStart=/usr/bin/certbot -q renew --no-random-sleep-on-renew
```

**Errores habituales:** si no aparece, actívalo con `sudo systemctl enable --now certbot.timer`.

### PASO 10.2 – Simular una renovación

**Objetivo:** comprobar que la renovación funcionará sin esperar 60 días.

**Comando:** 

```bash title="Terminal"
sudo certbot renew --dry-run
```

Análisis: `--dry-run` hace todo el proceso contra el entorno de pruebas de Let's Encrypt sin guardar nada. Comprueba que el DNS, el puerto 80 y los webroots siguen bien.

**Resultado esperado:** 

```text title="Salida esperada"
Processing /etc/letsencrypt/renewal/ejemplo1.com.conf
Simulating renewal of an existing certificate for ejemplo1.com and www.ejemplo1.com
...
Congratulations, all simulated renewals succeeded:
  /etc/letsencrypt/live/ejemplo1.com/fullchain.pem (success)
  /etc/letsencrypt/live/ejemplo2.com/fullchain.pem (success)
  /etc/letsencrypt/live/mail.ejemplo1.com/fullchain.pem (success)
```

**Errores habituales:** los mismos del PASO 3.11 (puerto 80 cerrado, DNS que ya no apunta al servidor, webroot que no coincide con el DocumentRoot). Revisa el método guardado en `/etc/letsencrypt/renewal/NOMBRE.conf` (línea `webroot_path`).

### PASO 10.3 – Recargar los servicios tras cada renovación (deploy hook)

**Objetivo:** que Apache, Postfix y Dovecot empiecen a usar el certificado nuevo en cuanto se renueve.

**Explicación:** los servicios cargan el certificado en memoria al arrancar. Si Certbot renueva el archivo pero nadie recarga el servicio, este seguirá presentando el certificado antiguo hasta que caduque. Los scripts de `renewal-hooks/deploy/` se ejecutan tras cada renovación con éxito y reciben la variable `RENEWED_LINEAGE` con la carpeta del certificado renovado.

**Comando:** 

```bash title="Terminal"
sudo nano /etc/letsencrypt/renewal-hooks/deploy/recargar-servicios.sh
```

**Archivo:** `/etc/letsencrypt/renewal-hooks/deploy/recargar-servicios.sh` (nuevo, contenido completo):

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

Análisis del script:

| Línea | Qué hace |
|---|---|
| `logger -t certbot-deploy` | Escribe un mensaje en el registro del sistema (visible con `journalctl -t certbot-deploy`). |
| `case ... in */mail.ejemplo1.com)` | Si el certificado renovado es el del correo, recarga Postfix y Dovecot. |
| `apachectl configtest` antes de recargar | Nunca recarga Apache si su configuración tiene un error (evita tumbar las webs de madrugada). |
| `systemctl reload` | Recarga sin cortar conexiones. |

Hazlo ejecutable y pruébalo simulando la renovación del certificado de correo:

```bash title="Terminal"
sudo chmod 755 /etc/letsencrypt/renewal-hooks/deploy/recargar-servicios.sh
sudo RENEWED_LINEAGE=/etc/letsencrypt/live/mail.ejemplo1.com /etc/letsencrypt/renewal-hooks/deploy/recargar-servicios.sh
sudo RENEWED_LINEAGE=/etc/letsencrypt/live/ejemplo1.com /etc/letsencrypt/renewal-hooks/deploy/recargar-servicios.sh
sudo journalctl -t certbot-deploy -n 10 --no-pager
```

**Resultado esperado:** 

```text title="Salida esperada"
nov 09 03:40:11 mail certbot-deploy[9101]: Certificado renovado: /etc/letsencrypt/live/mail.ejemplo1.com
nov 09 03:40:11 mail certbot-deploy[9105]: Postfix y Dovecot recargados
nov 09 03:40:15 mail certbot-deploy[9112]: Certificado renovado: /etc/letsencrypt/live/ejemplo1.com
nov 09 03:40:15 mail certbot-deploy[9116]: Apache recargado
```

!!! tip "Consejo"
    `certbot renew --dry-run` **no** ejecuta los *deploy hooks* (porque no se instala ningún certificado real). Por eso los probamos llamando al script directamente, como acabamos de hacer.

## 10.6 Verificar los certificados de todos los servicios

### PASO 10.4 – Comprobar el certificado que presenta cada puerto

**Objetivo:** confirmar que cada servicio presenta el certificado correcto, vigente y con la cadena completa.

**Comandos:** 

```bash title="Terminal"
# Web (HTTPS, puerto 443) - con SNI
echo | openssl s_client -connect www.ejemplo1.com:443 -servername www.ejemplo1.com 2>/dev/null | openssl x509 -noout -subject -issuer -dates -ext subjectAltName
# SMTP con STARTTLS (puertos 25 y 587)
echo | openssl s_client -connect mail.ejemplo1.com:587 -starttls smtp 2>/dev/null | openssl x509 -noout -subject -dates -ext subjectAltName
# SMTPS (puerto 465, TLS implícito)
echo | openssl s_client -connect mail.ejemplo1.com:465 2>/dev/null | openssl x509 -noout -subject -dates
# IMAPS (993) y POP3S (995)
echo | openssl s_client -connect mail.ejemplo2.com:993 2>/dev/null | openssl x509 -noout -subject -dates -ext subjectAltName
echo | openssl s_client -connect mail.ejemplo1.com:995 2>/dev/null | openssl x509 -noout -subject -dates
```

**Análisis del comando:** 

- `echo |` envía una línea vacía para que `s_client` cierre la conexión tras el saludo.
- `-servername` activa SNI (necesario en Apache para obtener el certificado del VirtualHost correcto).
- `-starttls smtp` inicia la conexión en texto plano y la pasa a TLS con STARTTLS (también existen `-starttls imap` y `-starttls pop3`).
- `openssl x509 -noout -subject -issuer -dates -ext subjectAltName` extrae del certificado el titular, el emisor, las fechas y los nombres que cubre.

**Resultado esperado (para el 993 de mail.ejemplo2.com):** 

```text title="Salida esperada"
subject=CN = mail.ejemplo1.com
notBefore=Nov  9 00:41:03 2026 GMT
notAfter=Feb  7 00:41:02 2027 GMT
X509v3 Subject Alternative Name:
    DNS:mail.ejemplo1.com, DNS:mail.ejemplo2.com
```

Para comprobar que la cadena es válida (el cliente puede verificarla):

```bash title="Terminal"
echo | openssl s_client -connect mail.ejemplo1.com:993 -verify_return_error 2>&1 | grep -E 'Verify return code|Protocol|Cipher'
```

```text title="Salida esperada"
Protocol  : TLSv1.3
Cipher    : TLS_AES_256_GCM_SHA384
Verify return code: 0 (ok)
```

`Verify return code: 0 (ok)` significa cadena completa y válida.

### PASO 10.5 – Comprobar que los protocolos obsoletos están desactivados

**Comando:** 

```bash title="Terminal"
openssl s_client -connect www.ejemplo1.com:443 -servername www.ejemplo1.com -tls1_1 </dev/null 2>&1 | grep -E 'alert|Cipher is|error' | head -3
openssl s_client -connect mail.ejemplo1.com:993 -tls1_1 </dev/null 2>&1 | grep -E 'alert|Cipher is|error' | head -3
```

**Resultado esperado:** un error del tipo `alert protocol version` o `no protocols available` y `Cipher is (NONE)`. Significa que TLS 1.1 es rechazado.

!!! tip "Consejo"
    Para una auditoría completa, usa la herramienta libre **testssl.sh** (`./testssl.sh mail.ejemplo1.com:993`) o, para las webs, el test de **SSL Labs** (en su web). Ambas califican protocolos, cifrados y configuración.

Resumen de la configuración TLS en cada servicio:

| Servicio | Archivo | Directivas TLS clave |
|---|---|---|
| Apache | `/etc/apache2/sites-available/*.conf` y `mods-available/ssl.conf` | `SSLCertificateFile`, `SSLCertificateKeyFile`, `SSLProtocol` |
| Postfix entrante | `/etc/postfix/main.cf` | `smtpd_tls_cert_file`, `smtpd_tls_key_file`, `smtpd_tls_protocols`, `smtpd_tls_auth_only` |
| Postfix saliente | `/etc/postfix/main.cf` | `smtp_tls_security_level`, `smtp_tls_CApath` |
| Postfix 587 / 465 | `/etc/postfix/master.cf` | `smtpd_tls_security_level=encrypt` / `smtpd_tls_wrappermode=yes` |
| Dovecot | `/etc/dovecot/conf.d/10-ssl.conf` | `ssl = required`, `ssl_cert`, `ssl_key`, `ssl_min_protocol` |

## 10.7 Protección de las conexiones de autenticación

Las contraseñas solo deben viajar dentro de TLS. Esta es la lista de comprobación que ya cumple nuestra configuración:

| Conexión | Protección | Directiva |
|---|---|---|
| Inicio de sesión IMAP/POP3 | Solo por 993/995 (TLS implícito); 143/110 deshabilitados. | `port = 0` en `10-master.conf` |
| Cualquier autenticación en Dovecot | Rechazada sin TLS. | `ssl = required`, `disable_plaintext_auth = yes` |
| Autenticación SMTP en el puerto 25 | Desactivada. | `smtpd_sasl_auth_enable = no` |
| Autenticación SMTP en 587 | Solo tras STARTTLS. | `smtpd_tls_security_level=encrypt`, `smtpd_tls_auth_only=yes` |
| Autenticación SMTP en 465 | TLS desde el inicio. | `smtpd_tls_wrappermode=yes` |
| Paneles web (si los instalas) | Solo HTTPS + HSTS. | Redirección 80→443 y `Strict-Transport-Security` |

**Comprobación:** intenta autenticarte en el 587 sin STARTTLS:

```bash title="Terminal"
swaks --server mail.ejemplo1.com --port 587 --auth LOGIN --auth-user info@ejemplo1.com --auth-password prueba --quit-after AUTH
```

```text title="Salida esperada"
<** 530 5.7.0 Must issue a STARTTLS command first
```

## 10.8 Operaciones habituales con certificados

| Tarea | Comando |
|---|---|
| Añadir un nombre a un certificado existente | `sudo certbot certonly --webroot -w /var/www/ejemplo1.com/public_html --cert-name ejemplo1.com -d ejemplo1.com -d www.ejemplo1.com -d tienda.ejemplo1.com` |
| Forzar la renovación de uno | `sudo certbot renew --cert-name mail.ejemplo1.com --force-renewal` (úsalo con moderación: hay límites) |
| Revocar (clave comprometida) | `sudo certbot revoke --cert-name ejemplo2.com` |
| Borrar un certificado que ya no usas | `sudo certbot delete --cert-name ejemplo2.com` (antes quítalo de la configuración de Apache) |
| Ver los días que le quedan a un certificado | `sudo openssl x509 -enddate -noout -in /etc/letsencrypt/live/ejemplo1.com/cert.pem` |

!!! tip "Consejo"
    **Registro CAA (opcional).** Puedes publicar `ejemplo1.com. CAA 0 issue "letsencrypt.org"` para indicar que solo Let's Encrypt puede emitir certificados para tu dominio. Es una capa adicional de protección frente a emisiones fraudulentas.

!!! tip "Consejo"
    **Laboratorio.** Si usas el certificado autofirmado y cambias el nombre del servidor, regenera el certificado con `sudo make-ssl-cert generate-default-snakeoil --force-overwrite` y reinicia Apache, Postfix y Dovecot.

## 10.9 Ejercicios prácticos del Módulo 10

1. Ejecuta `sudo ls -l /etc/letsencrypt/live/ejemplo1.com/` y explica a qué apunta cada enlace.
2. Conéctate al puerto 443 **sin** `-servername` y observa qué certificado recibes. ¿Por qué es el del comodín `default-ssl`?
3. Configura Thunderbird con el servidor `203.0.113.10` (la IP) en lugar de `mail.ejemplo1.com`. ¿Qué aviso aparece y por qué?
4. Escribe un pequeño script que muestre los días que quedan para que caduque cada certificado de `/etc/letsencrypt/live/*/cert.pem` (pista: `openssl x509 -checkend`).
5. Simula una renovación forzada del certificado de correo y verifica en `journalctl -t certbot-deploy` que se recargaron Postfix y Dovecot.
