# Introducción

!!! abstract "Descargar el manual completo"
    También disponible como libro: [PDF](descargas/Manual_Administracion_Servidores_Linux.pdf) · [Word (.docx)](descargas/Manual_Administracion_Servidores_Linux.docx)

## A quién va dirigido este manual

Este manual está escrito para una persona que nunca ha administrado un servidor y quiere aprender a hacerlo de verdad. No basta con instalar paquetes: el objetivo es entender qué se hace, por qué se hace, qué pasa por dentro y cómo comprobar que funciona.

Al terminar, serás capaz de montar, desde un servidor vacío, una infraestructura completa con:

- Dos sitios web independientes (`ejemplo1.com` y `ejemplo2.com`) servidos por Apache mediante Virtual Hosting.
- Una aplicación PHP en cada sitio, ejecutada con PHP-FPM y un usuario del sistema propio para cada web.
- Una base de datos MariaDB independiente para cada aplicación (`web1_db` y `web2_db`), cada una con su propio usuario y permisos limitados.
- Un servidor de correo completo: Postfix (SMTP) para enviar y recibir, y Dovecot (IMAP/POP3) para consultar los buzones.
- Cuentas de correo virtuales como `info@ejemplo1.com`, `administracion@ejemplo1.com` y `soporte@ejemplo2.com`.
- Firma DKIM, políticas SPF y DMARC, y DNS inverso para que tus correos no acaben en la carpeta de spam.
- Certificados SSL/TLS gratuitos de Let's Encrypt para la web y el correo, con renovación automática.
- Cortafuegos UFW, protección contra fuerza bruta con Fail2ban, copias de seguridad automáticas y procedimientos de diagnóstico.
- Como complemento, un servidor FTP cifrado (FTPS) con vsftpd para que el responsable de cada web suba sus archivos, encerrado en su propia carpeta.

## Cómo está organizado

El manual tiene 14 módulos que se apoyan unos en otros, más un módulo complementario (el 15) dedicado al servidor FTP. Cada módulo empieza con sus objetivos y termina con ejercicios prácticos. Las operaciones están divididas en **PASOS** que siempre siguen la misma estructura:

| Apartado | Qué contiene |
|---|---|
| Objetivo | Qué vamos a conseguir con este paso. |
| Explicación | Qué estamos haciendo, por qué y qué ocurre internamente. |
| Comando | El comando completo, listo para copiar y pegar. |
| Análisis del comando | Qué significa cada parte y cada parámetro. |
| Resultado esperado | Lo que debería aparecer en el terminal. |
| Comprobación | Cómo verificar que el paso ha funcionado antes de seguir. |
| Errores habituales | Qué puede fallar y cómo solucionarlo. |
| Siguiente paso | Qué haremos a continuación. |

!!! warning "Atención"
    **Regla de oro del manual:** no pases nunca al paso siguiente sin haber hecho la comprobación del paso actual. Casi todos los problemas difíciles de un servidor son en realidad tres problemas fáciles que se han acumulado sin que nadie los detectara.

## Convenciones tipográficas

Los bloques de código llevan una etiqueta que indica qué son:

```bash title="Terminal"
# TERMINAL: comandos que se escriben en la consola del servidor (o del equipo que se indique).
# Las líneas que empiezan por # son comentarios: no hacen nada, solo explican.
sudo systemctl status apache2
```

```text title="Archivo de configuración"
# ARCHIVO DE CONFIGURACIÓN: contenido de un archivo. Encima se indica siempre su ruta exacta.
ServerName www.ejemplo1.com
```

```text title="Salida esperada"
SALIDA ESPERADA: lo que el terminal debería mostrarte. Puede variar ligeramente
(fechas, números de proceso, versiones menores), pero la estructura será la misma.
```

Además:

- El indicador `sysadmin@mail:~$` significa «terminal del servidor, como usuario normal `sysadmin`». Si ves `root@mail:~#` estás como superusuario. Nunca copies el indicador, solo lo que va detrás.
- `sudo` delante de un comando significa «ejecútalo con privilegios de administrador». Te pedirá tu contraseña de usuario (no la de root).
- Las palabras entre `<` y `>` (por ejemplo `<CONTRASEÑA_SEGURA>`) son marcadores: sustitúyelas por tu valor real, **sin** los signos `<` y `>`.

!!! note "Nota"
    Este manual no contiene contraseñas reales. Todos los valores como `Cambia.Esta.Clave.01` son ejemplos y **debes** sustituirlos por contraseñas largas y únicas generadas por ti (más adelante verás cómo generarlas con `openssl rand`).

## Plataforma de referencia y versiones

Para que todos los comandos, rutas y archivos sean exactos, el manual usa una única plataforma de referencia:

| Componente | Versión de referencia (Ubuntu Server 24.04 LTS) |
|---|---|
| Sistema operativo | Ubuntu Server 24.04 LTS «Noble Numbat», 64 bits (amd64 o arm64) |
| Servidor web | Apache HTTP Server 2.4.58 o superior (paquete `apache2`) |
| PHP | PHP 8.3 con PHP-FPM (paquetes `php8.3-*`) |
| Base de datos | MariaDB 10.11 LTS (paquete `mariadb-server`) |
| SMTP | Postfix 3.8 |
| IMAP / POP3 | Dovecot 2.3.21 |
| Firma DKIM | OpenDKIM 2.11 |
| Certificados | Certbot 2.x (paquete `certbot` + `python3-certbot-apache`) |
| Cortafuegos | UFW 0.36 |
| Anti fuerza bruta | Fail2ban 1.0 |
| FTP (módulo complementario) | vsftpd 3.0.5 |

Ubuntu 24.04 LTS tiene soporte estándar hasta abril de 2029, utiliza Dovecot 2.3 (cuya sintaxis de configuración está muy documentada) y es la plataforma que la mayoría de proveedores de servidores ofrecen como imagen preinstalada.

!!! warning "Atención"
    **Si usas otra versión del sistema, lee esto.** Debian 12 «bookworm» es casi idéntica a Ubuntu 24.04 en todo lo que hace este manual; la diferencia principal es que trae PHP 8.2, así que donde ponga `8.3` deberás escribir `8.2` (por ejemplo `/etc/php/8.2/fpm/`). En cambio, **Debian 13 «trixie» y Ubuntu 26.04 incluyen Dovecot 2.4**, cuya sintaxis de configuración cambió de forma importante (por ejemplo `mail_location` se divide en `mail_driver` y `mail_path`, y el bloque `passdb` usa otro formato). Los módulos 7 y 8 están escritos para Dovecot 2.3. Comprueba siempre tu versión con `dovecot --version` antes de copiar su configuración.

## Antes de empezar: lo que necesitas saber de tu entorno

Un administrador nunca empieza a instalar sin conocer el terreno. Rellena esta ficha antes de continuar; la usarás durante todo el manual.

| Dato | Cómo averiguarlo | Valor en el manual | Tu valor |
|---|---|---|---|
| Sistema operativo y versión | `lsb_release -a` o `cat /etc/os-release` | Ubuntu 24.04 LTS | |
| Arquitectura | `uname -m` (x86_64 = amd64; aarch64 = arm64) | x86_64 | |
| IP pública IPv4 | Panel del proveedor o `curl -4 ifconfig.me` | 203.0.113.10 | |
| IP pública IPv6 (si tienes) | Panel del proveedor o `ip -6 addr` | 2001:db8:10::10 | |
| Dominio 1 | El que hayas registrado | ejemplo1.com | |
| Dominio 2 | El que hayas registrado | ejemplo2.com | |
| Acceso | SSH con usuario y contraseña, o consola web | SSH | |
| Proveedor DNS | Donde gestionas los registros (registrador, Cloudflare...) | — | |
| ¿Puedes configurar el DNS inverso (PTR)? | Panel del proveedor del servidor (no del dominio) | Sí | |
| ¿El puerto 25 saliente está abierto? | Documentación del proveedor; prueba del Módulo 6 | Sí | |
| RAM y disco | `free -h` y `df -h` | 2 GB / 40 GB | |

!!! note "Nota"
    Las direcciones `203.0.113.10` y `2001:db8:10::10` pertenecen a rangos reservados para documentación (RFC 5737 y RFC 3849). No existen en Internet: sustitúyelas siempre por las tuyas. Lo mismo ocurre con `ejemplo1.com` y `ejemplo2.com`.

### Restricciones de proveedores que debes conocer

- **Puerto 25 bloqueado.** Muchas nubes (AWS, Google Cloud, Azure, y la mayoría de conexiones domésticas) bloquean por defecto el tráfico saliente al puerto 25 para frenar el spam. Sin él tu servidor podrá **recibir** correo, pero no **entregarlo** a Gmail, Outlook, etc. Algunos proveedores lo desbloquean bajo petición; otros nunca. Si es tu caso, tendrás que usar un *relay* (servidor intermediario) de envío. Lo verás en el Módulo 6.
- **DNS inverso (PTR).** Lo configura el dueño de la IP (tu proveedor del servidor), no tu registrador de dominios. Sin un PTR correcto, gran parte de tus correos será rechazada.
- **IP doméstica o dinámica.** Puedes seguir todo el manual en una máquina virtual de tu casa para aprender, pero un servidor de correo real necesita una IP pública fija con buena reputación. Las IP domésticas suelen estar en listas negras (por ejemplo la PBL de Spamhaus).
- **RAM.** 1 GB es el mínimo absoluto; 2 GB es lo recomendable para Apache + PHP + MariaDB + correo.

!!! tip "Consejo"
    **Si estás aprendiendo en un laboratorio local** (VirtualBox, VMware, Proxmox, Hyper-V) sin IP pública ni dominios reales, puedes seguir todos los módulos. Las diferencias son tres: (1) en lugar de configurar el DNS público, añadirás los nombres al archivo `hosts` de tu equipo; (2) Let's Encrypt no podrá emitir certificados, así que usarás el certificado autofirmado que Ubuntu genera automáticamente (`/etc/ssl/certs/ssl-cert-snakeoil.pem`); (3) no podrás intercambiar correo con Internet, pero sí entre tus propias cuentas. Cada módulo indica cómo adaptarlo.

## Los datos del proyecto guía

Durante todo el manual usaremos estos valores. Cuando adaptes el manual a tu caso, mantén la coherencia: si cambias un nombre, cámbialo en todas partes.

| Elemento | ejemplo1.com | ejemplo2.com |
|---|---|---|
| Nombre web principal | www.ejemplo1.com | www.ejemplo2.com |
| Alias | ejemplo1.com | ejemplo2.com |
| Usuario Linux del sitio | web1 | web2 |
| Carpeta del sitio | /var/www/ejemplo1.com | /var/www/ejemplo2.com |
| Carpeta pública (DocumentRoot) | /var/www/ejemplo1.com/public_html | /var/www/ejemplo2.com/public_html |
| Archivo VirtualHost | /etc/apache2/sites-available/ejemplo1.com.conf | /etc/apache2/sites-available/ejemplo2.com.conf |
| Pool PHP-FPM | /etc/php/8.3/fpm/pool.d/web1.conf | /etc/php/8.3/fpm/pool.d/web2.conf |
| Base de datos | web1_db | web2_db |
| Usuario de base de datos | web1_user | web2_user |
| Cuentas de correo | info@, administracion@, postmaster@ (alias) | soporte@, postmaster@ (alias) |
| Servidor de correo (MX) | mail.ejemplo1.com | mail.ejemplo2.com (misma IP) |
| Acceso FTP (Módulo 15) | ftp.ejemplo1.com, usuario web1 | ftp.ejemplo2.com, usuario web2 |

El servidor se llamará `mail.ejemplo1.com`. Es su nombre «oficial» (el que aparecerá en el DNS inverso y en los saludos SMTP). El usuario administrador del sistema será `sysadmin`.
