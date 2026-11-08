# Módulo 11. Integración completa de los servicios

## Objetivos del módulo

- Ver la infraestructura como un todo: qué recursos tiene cada dominio y cómo se relacionan.
- Seguir paso a paso el recorrido de una visita web y el de un correo electrónico.
- Conectar el sistema web con el sistema de correo: una aplicación PHP que envía correos firmados con DKIM.
- Disponer de un inventario completo y coherente de la infraestructura.

## 11.1 Inventario de la infraestructura

Cada dominio es un «inquilino» independiente que comparte el servidor, pero no sus recursos:

| Recurso | ejemplo1.com | ejemplo2.com |
|---|---|---|
| Registros DNS web | A `@`, A `www` → 203.0.113.10 | A `@`, A `www` → 203.0.113.10 |
| Registros DNS correo | MX → mail.ejemplo1.com; SPF; DKIM `mail._domainkey`; DMARC | MX → mail.ejemplo2.com; SPF; DKIM `mail._domainkey`; DMARC |
| VirtualHost | `/etc/apache2/sites-available/ejemplo1.com.conf` | `/etc/apache2/sites-available/ejemplo2.com.conf` |
| Certificado web | `/etc/letsencrypt/live/ejemplo1.com/` | `/etc/letsencrypt/live/ejemplo2.com/` |
| Usuario Linux de la web | `web1` | `web2` |
| Carpeta web | `/var/www/ejemplo1.com/public_html` | `/var/www/ejemplo2.com/public_html` |
| Pool PHP-FPM | `web1` → `/run/php/php8.3-fpm-web1.sock` | `web2` → `/run/php/php8.3-fpm-web2.sock` |
| Aplicación PHP | Agenda de contactos + formulario de contacto | Agenda de contactos + formulario de contacto |
| Credenciales | `/var/www/ejemplo1.com/config/config.php` | `/var/www/ejemplo2.com/config/config.php` |
| Base de datos | `web1_db` | `web2_db` |
| Usuario de base de datos | `'web1_user'@'localhost'` | `'web2_user'@'localhost'` |
| Cuentas de correo | info@, administracion@ | soporte@ |
| Alias | postmaster@, abuse@, dmarc@, noreply@, root@, equipo@ | postmaster@, abuse@, dmarc@, noreply@ |
| Buzones | `/var/mail/vhosts/ejemplo1.com/` | `/var/mail/vhosts/ejemplo2.com/` |
| Clave DKIM | `/etc/opendkim/keys/ejemplo1.com/mail.private` | `/etc/opendkim/keys/ejemplo2.com/mail.private` |
| Servidor para los clientes de correo | mail.ejemplo1.com | mail.ejemplo2.com |

Recursos compartidos: la IP, el nombre `mail.ejemplo1.com` (PTR y `myhostname`), el certificado del correo, los servicios Apache, PHP-FPM, MariaDB, Postfix, Dovecot y OpenDKIM, el cortafuegos y Fail2ban.

## 11.2 El sistema web: INTERNET → DNS → APACHE → PHP → MARIADB

```text title="Diagrama"
 ┌───────────┐  1. ¿IP de www.ejemplo2.com?   ┌─────────────────┐
 │ NAVEGADOR │ ─────────────────────────────► │ DNS             │
 │           │ ◄───────────────────────────── │ A → 203.0.113.10│
 └─────┬─────┘  2. 203.0.113.10               └─────────────────┘
       │ 3. TCP 443 + TLS (SNI: www.ejemplo2.com)
       ▼
 ┌──────────────────────────────── SERVIDOR ────────────────────────────────────┐
 │  UFW: 443 permitido                                                          │
 │        ▼                                                                     │
 │  APACHE (mpm_event, usuario www-data)                                        │
 │   4. SNI → VirtualHost www.ejemplo2.com → certificado ejemplo2.com           │
 │   5. GET /  → DirectoryIndex index.php → FilesMatch \.php$                   │
 │        ▼ FastCGI por /run/php/php8.3-fpm-web2.sock                           │
 │  PHP-FPM pool web2 (usuario web2, open_basedir /var/www/ejemplo2.com/)       │
 │   6. Ejecuta index.php → lee ../config/config.php → PDO                      │
 │        ▼ socket /run/mysqld/mysqld.sock (usuario BD web2_user)               │
 │  MARIADB (127.0.0.1)                                                         │
 │   7. Comprueba privilegios en web2_db → SELECT ... FROM contactos            │
 │        ▲ filas                                                               │
 │  PHP genera HTML (8) → Apache cifra y responde (9)                           │
 │  Logs: /var/log/apache2/ejemplo2.com-access.log · /var/www/ejemplo2.com/logs │
 └──────────────────────────────────────────────────────────────────────────────┘
```

| Etapa | Componente | Si falla aquí, verás... |
|---|---|---|
| 1-2 | DNS | `ERR_NAME_NOT_RESOLVED` en el navegador. |
| 3 | Red / UFW | El navegador se queda esperando (timeout). |
| 4 | Apache / TLS | Aviso de certificado, o la web de otro dominio. |
| 5 | Apache → PHP-FPM | 503 o el código PHP en texto plano. |
| 6 | PHP | 500, página en blanco; error en el log de PHP de la web. |
| 7 | MariaDB | Mensaje de error de la aplicación; `SQLSTATE` en el log de PHP. |

## 11.3 El sistema de correo: INTERNET → DNS MX → POSTFIX → DOVECOT → BUZÓN → CLIENTE

```text title="Diagrama"
 ┌─────────────┐ 1. ¿MX de ejemplo2.com?    ┌──────────────────────────────┐
 │ Servidor    │ ─────────────────────────► │ DNS: MX 10 mail.ejemplo2.com │
 │ remitente   │ ◄───────────────────────── │      A  203.0.113.10         │
 │ (Gmail)     │                            └──────────────────────────────┘
 └──────┬──────┘
        │ 2. TCP 25 (STARTTLS) · MAIL FROM / RCPT TO:<soporte@ejemplo2.com>
        ▼
 ┌──────────────────────────────── SERVIDOR ───────────────────────────────────┐
 │  POSTFIX smtpd (25)                                                         │
 │   3. Restricciones: HELO válido, remitente con dominio real, ¿destino mío?  │
 │   4. virtual_domains: ejemplo2.com ✓  · vmailbox: soporte@ ✓                │
 │   5. Milter OpenDKIM: verifica la firma DKIM del remitente (entrante)       │
 │   6. Cola → qmgr → lmtp                                                     │
 │        ▼ LMTP por /var/spool/postfix/private/dovecot-lmtp                   │
 │  DOVECOT lmtp                                                               │
 │   7. userdb (/etc/dovecot/users): uid 5000, home /var/mail/vhosts/...       │
 │   8. Guarda en /var/mail/vhosts/ejemplo2.com/soporte/Maildir/new/           │
 │        ▲                                                                    │
 │  DOVECOT imap (993)                                                         │
 │   10. Autentica a soporte@ejemplo2.com (passdb) y sirve los mensajes        │
 └────────▲────────────────────────────────────────────────────────────────────┘
          │ 9. IMAP 993 (TLS) usuario soporte@ejemplo2.com
 ┌────────┴────────┐
 │ CLIENTE DE CORREO│  (Thunderbird, móvil)
 └──────────────────┘
```

Y en sentido inverso, cuando `soporte@ejemplo2.com` envía:

```text title="Diagrama"
 CLIENTE ──587 STARTTLS / 465 TLS──► POSTFIX submission/smtps
     1. AUTH PLAIN → pregunta a DOVECOT por /var/spool/postfix/private/auth → OK
     2. sender_login: ¿soporte@ puede usar MAIL FROM soporte@ejemplo2.com? → sí
     3. Milter OpenDKIM (ORIGINATING): firma con la clave de ejemplo2.com
     4. Cola → smtp → consulta MX del destino → TCP 25 → servidor de destino
        (el destino comprueba PTR, SPF, DKIM y DMARC)
```

## 11.4 Cómo se relacionan ambos sistemas: la web que envía correos

Las aplicaciones web necesitan enviar correo constantemente: formularios de contacto, confirmaciones de registro, recuperación de contraseñas, avisos de pedidos. Hay dos formas de hacerlo en nuestra infraestructura:

| Método | Cómo funciona | Ventajas | Inconvenientes |
|---|---|---|---|
| **`sendmail` local** (función `mail()` de PHP) | PHP ejecuta `/usr/sbin/sendmail` (que pertenece a Postfix); el mensaje entra por `pickup`, sin red ni contraseñas. | Sencillo, sin credenciales en el código, firmado por DKIM (`non_smtpd_milters`). | Solo funciona en el mismo servidor. |
| SMTP autenticado (librerías como PHPMailer o Symfony Mailer) | La aplicación se conecta al puerto 587 con usuario y contraseña de una cuenta de correo. | Funciona desde cualquier servidor; control fino. | Hay que guardar otra contraseña en la configuración. |

Usaremos el primero, que ya dejamos preparado: en el pool de cada web (PASO 5.3) fijamos `sendmail_path = /usr/sbin/sendmail -t -i -f noreply@ejemplo1.com`, de modo que el **remitente del sobre** de todo lo que envíe la web 1 es `noreply@ejemplo1.com` (alias de `administracion@`, que recibirá rebotes), aunque el código PHP intente otra cosa.

```text title="Diagrama"
 Visitante rellena el formulario en https://www.ejemplo1.com/contacto.php
     │
     ▼
 PHP-FPM pool web1 ── mail() ──► /usr/sbin/sendmail -t -i -f noreply@ejemplo1.com
                                      │ (cola maildrop de Postfix)
                                      ▼
                          POSTFIX pickup → cleanup → milter OpenDKIM (firma d=ejemplo1.com)
                                      │
                                      ▼ el destinatario es info@ejemplo1.com (dominio virtual)
                          qmgr → lmtp → DOVECOT → buzón de info@ejemplo1.com
```

### PASO 11.1 – Crear el formulario de contacto de ejemplo1.com

**Objetivo:** una página que guarda los mensajes de los visitantes en la base de datos y los envía por correo a `info@ejemplo1.com`.

Primero crea la tabla para guardar los mensajes (así no se pierden aunque falle el correo):

```bash title="Terminal"
sudo mariadb -e "CREATE TABLE web1_db.mensajes (
  id INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
  nombre VARCHAR(100) NOT NULL,
  email VARCHAR(150) NOT NULL,
  mensaje TEXT NOT NULL,
  enviado TINYINT(1) NOT NULL DEFAULT 0,
  creado_en TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;"
sudo mariadb -e "CREATE TABLE web2_db.mensajes LIKE web1_db.mensajes;"
```

Como `web1_user` tiene `SELECT, INSERT, UPDATE, DELETE` sobre todas las tablas de `web1_db`, ya puede usar la nueva tabla sin cambiar permisos.

**Comando:** 

```bash title="Terminal"
sudo -u web1 nano /var/www/ejemplo1.com/public_html/contacto.php
```

**Archivo:** `/var/www/ejemplo1.com/public_html/contacto.php` (nuevo, contenido completo):

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
$enviado = false;
$datos = ['nombre' => '', 'email' => '', 'mensaje' => ''];

if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    if (!hash_equals($_SESSION['csrf'], (string)($_POST['csrf'] ?? ''))) {
        http_response_code(400);
        exit('Petición no válida.');
    }
    // Campo trampa invisible: los robots lo rellenan, las personas no
    if (($_POST['web'] ?? '') !== '') {
        exit('Gracias.');
    }

    $datos = [
        'nombre'  => trim((string)($_POST['nombre'] ?? '')),
        'email'   => trim((string)($_POST['email'] ?? '')),
        'mensaje' => trim((string)($_POST['mensaje'] ?? '')),
    ];

    if ($datos['nombre'] === '' || mb_strlen($datos['nombre']) > 100 || preg_match('/[\r\n]/', $datos['nombre'])) {
        $errores[] = 'Indica un nombre válido (máximo 100 caracteres).';
    }
    if (!filter_var($datos['email'], FILTER_VALIDATE_EMAIL)) {
        $errores[] = 'Indica un email válido.';
    }
    if ($datos['mensaje'] === '' || mb_strlen($datos['mensaje']) > 5000) {
        $errores[] = 'Escribe un mensaje (máximo 5000 caracteres).';
    }

    if (!$errores) {
        try {
            $pdo = obtenerConexion();
            // 1. Guardar en la base de datos (consulta preparada)
            $stmt = $pdo->prepare('INSERT INTO mensajes (nombre, email, mensaje) VALUES (:nombre, :email, :mensaje)');
            $stmt->execute($datos);
            $id = (int)$pdo->lastInsertId();

            // 2. Enviar por correo al buzón de la empresa
            $destino = $config['app']['correo_contacto'];
            $dominio = $config['app']['nombre'];
            $asunto  = '=?UTF-8?B?' . base64_encode("Nuevo mensaje web #$id de {$datos['nombre']}") . '?=';
            $cuerpo  = "Mensaje recibido desde el formulario de $dominio\n\n"
                     . "Nombre: {$datos['nombre']}\n"
                     . "Email:  {$datos['email']}\n"
                     . "Fecha:  " . date('Y-m-d H:i:s') . "\n\n"
                     . $datos['mensaje'] . "\n";
            $cabeceras = [
                'From'                      => "Formulario web <noreply@$dominio>",
                'Reply-To'                  => $datos['email'],
                'MIME-Version'              => '1.0',
                'Content-Type'              => 'text/plain; charset=UTF-8',
                'Content-Transfer-Encoding' => '8bit',
                'X-Mailer'                  => 'Formulario ' . $dominio,
            ];

            if (mail($destino, $asunto, $cuerpo, $cabeceras)) {
                $pdo->prepare('UPDATE mensajes SET enviado = 1 WHERE id = :id')->execute(['id' => $id]);
                $enviado = true;
            } else {
                error_log("Contacto: mail() devolvió false para el mensaje #$id");
                $errores[] = 'Hemos guardado tu mensaje, pero no se ha podido enviar el aviso. Lo revisaremos.';
            }
        } catch (PDOException $ex) {
            error_log('Contacto: error de base de datos: ' . $ex->getMessage());
            $errores[] = 'Se ha producido un error interno. Inténtalo más tarde.';
        }
    }
}
?>
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>Contacto · <?= e($config['app']['nombre']) ?></title>
    <link rel="stylesheet" href="/estilo.css">
</head>
<body>
<main>
    <h1>Contacto · <?= e($config['app']['nombre']) ?></h1>
    <?php if ($enviado): ?>
        <p class="ok">Gracias, <?= e($datos['nombre']) ?>. Hemos recibido tu mensaje y te responderemos pronto.</p>
    <?php else: ?>
        <?php foreach ($errores as $err): ?><p class="error"><?= e($err) ?></p><?php endforeach; ?>
        <form method="post" action="/contacto.php">
            <input type="hidden" name="csrf" value="<?= e($_SESSION['csrf']) ?>">
            <div style="position:absolute;left:-9999px" aria-hidden="true">
                <label>No rellenar <input name="web" tabindex="-1" autocomplete="off"></label>
            </div>
            <label>Nombre <input name="nombre" required maxlength="100" value="<?= e($datos['nombre']) ?>"></label>
            <label>Email <input name="email" type="email" required maxlength="150" value="<?= e($datos['email']) ?>"></label>
            <label>Mensaje <textarea name="mensaje" rows="6" required maxlength="5000"><?= e($datos['mensaje']) ?></textarea></label>
            <button type="submit">Enviar</button>
        </form>
    <?php endif; ?>
    <p><a href="/">Volver a la agenda</a></p>
</main>
</body>
</html>
```

Análisis de las decisiones de seguridad:

| Elemento | Por qué |
|---|---|
| `preg_match('/[\r\n]/', nombre)` | Impide la **inyección de cabeceras**: si un atacante pudiera meter saltos de línea en un dato que acaba en una cabecera, podría añadir destinatarios (`Bcc:`) y usar tu formulario para enviar spam. |
| `filter_var(FILTER_VALIDATE_EMAIL)` | El email del visitante va en `Reply-To`; debe ser una dirección válida y sin saltos de línea. |
| `From: noreply@ejemplo1.com` | El remitente visible es siempre **tu** dominio, nunca el email del visitante. Si pusieras el del visitante (por ejemplo `@gmail.com`), fallaría DMARC porque tu servidor no está autorizado a enviar en nombre de gmail.com. |
| `Reply-To: email del visitante` | Al pulsar «Responder» en el buzón, la respuesta va al visitante. |
| Asunto codificado `=?UTF-8?B?...?=` | Permite tildes y eñes en el asunto de forma estándar. |
| Guardar antes de enviar | Si falla el correo, el mensaje no se pierde. |
| Campo trampa `web` | Filtro sencillo contra robots de spam. |
| `mail()` con array de cabeceras | Desde PHP 7.2 se aceptan como array, más limpio y seguro que construir el texto a mano. |

```bash title="Terminal"
sudo chmod 640 /var/www/ejemplo1.com/public_html/contacto.php
```

Despliega lo mismo en ejemplo2.com (su `config.php` ya indica `soporte@ejemplo2.com` como destino y su pool usa `noreply@ejemplo2.com` como remitente del sobre):

```bash title="Terminal"
sudo install -o web2 -g www-data -m 640 /var/www/ejemplo1.com/public_html/contacto.php /var/www/ejemplo2.com/public_html/contacto.php
```

### PASO 11.2 – Probar el envío desde la web y seguir el correo de extremo a extremo

**Objetivo:** verificar la integración completa web → base de datos → correo → buzón.

**Comando:** abre el log en un terminal:

```bash title="Terminal"
sudo tail -f /var/log/mail.log
```

Y en el navegador, entra en `https://www.ejemplo1.com/contacto.php`, rellena el formulario y envíalo.

Resultado esperado en el log:

```text title="Salida esperada"
postfix/pickup[9301]: A1B2C3E110: uid=998 from=<noreply@ejemplo1.com>
postfix/cleanup[9310]: A1B2C3E110: message-id=<20261109041201.A1B2C3E110@mail.ejemplo1.com>
opendkim[8120]: A1B2C3E110: DKIM-Signature field added (s=mail, d=ejemplo1.com)
postfix/qmgr[9302]: A1B2C3E110: from=<noreply@ejemplo1.com>, size=1180, nrcpt=1 (queue active)
postfix/lmtp[9312]: A1B2C3E110: to=<info@ejemplo1.com>, relay=mail.ejemplo1.com[private/dovecot-lmtp], status=sent (250 2.0.0 <info@ejemplo1.com> ... Saved)
```

Interpretación línea a línea:

1. `pickup ... uid=998 from=<noreply@ejemplo1.com>`: el correo lo generó el usuario Linux 998 (`web1`) con el remitente de sobre que fijamos en su pool.
2. `opendkim ... DKIM-Signature field added (s=mail, d=ejemplo1.com)`: OpenDKIM lo firmó (el correo local pasa por `non_smtpd_milters`).
3. `lmtp ... to=<info@ejemplo1.com> ... Saved`: Dovecot lo guardó en el buzón.

Comprobación final:

```bash title="Terminal"
sudo mariadb -e "SELECT id, nombre, email, enviado, creado_en FROM web1_db.mensajes;"
sudo doveadm search -u info@ejemplo1.com mailbox INBOX header X-Mailer "Formulario"
```

En Thunderbird, el mensaje aparece en la bandeja de `info@ejemplo1.com`; al pulsar «Responder», el destinatario es el email del visitante.

**Errores habituales:** 

| Síntoma | Causa | Solución |
|---|---|---|
| `mail() devolvió false` en el log de PHP | `sendmail_path` incorrecto o Postfix detenido. | `ls -l /usr/sbin/sendmail`, `systemctl status postfix@-`. |
| `warning: mail_queue_enter: create file maildrop/...: Permission denied` | Permisos de la cola de Postfix alterados. | `sudo postfix set-permissions` y `sudo postfix check`. |
| El correo no aparece en el log | `disable_functions` incluye `mail` o hay un error de PHP antes. | Revisa el log de PHP de la web. |
| Llega sin firma DKIM | Falta `non_smtpd_milters`. | PASO 9.9. |

## 11.5 Dependencias entre servicios

Saber qué depende de qué te permite diagnosticar más rápido y reiniciar en el orden correcto:

| Servicio | Depende de | Si se detiene... |
|---|---|---|
| MariaDB | Disco | Las webs muestran errores de base de datos; lo estático sigue funcionando. |
| PHP-FPM | MariaDB (para las apps), Postfix (para `mail()`) | Las páginas `.php` dan 503; lo estático funciona. |
| Apache | PHP-FPM, certificados | Ninguna web responde. Certbot no puede renovar por webroot. |
| Dovecot | Certificado, `/etc/dovecot/users` | Los usuarios no leen correo; Postfix no puede entregar (los mensajes esperan en cola) ni autenticar envíos. |
| OpenDKIM | Claves | Con `milter_default_action = accept` el correo sale **sin firmar** (puede ir a spam). |
| Postfix | Dovecot (entrega y SASL), OpenDKIM | No se recibe ni envía correo; los servidores remitentes reintentarán durante días. |
| Certbot | Apache (webroot en el puerto 80), DNS | Los certificados caducan a los 90 días. |

## 11.6 Ejercicios prácticos del Módulo 11

1. Detén Dovecot, envía el formulario de contacto, observa con `mailq` que el correo queda en cola y arranca Dovecot de nuevo. ¿Se pierde el mensaje?
2. Modifica `contacto.php` para enviar además un acuse de recibo al visitante. ¿Qué remitente y qué `Reply-To` usarías? ¿Qué riesgo tiene enviar correos a direcciones que escribe cualquiera?
3. Comprueba en Gmail que un correo enviado por el formulario de ejemplo2.com pasa SPF, DKIM y DMARC con el dominio ejemplo2.com.
4. Dibuja de memoria los dos diagramas de este módulo e indica en qué archivo se configura cada flecha.
5. Instala PHPMailer en ejemplo2.com (con Composer, en una carpeta fuera de `public_html`) y envía el formulario por SMTP autenticado al puerto 587 usando una cuenta de correo. Compara las dos formas de envío en el log.
