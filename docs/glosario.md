# Glosario técnico

| Término | Definición |
|---|---|
| **A (registro)** | Registro DNS que asocia un nombre a una dirección IPv4. |
| **ACL (permisos)** | Lista de control de acceso: permisos adicionales para usuarios o grupos concretos sobre un archivo, más allá de propietario, grupo y otros (`setfacl`, `getfacl`). |
| **AAAA (registro)** | Registro DNS que asocia un nombre a una dirección IPv6. |
| **ACME** | Protocolo con el que Let's Encrypt verifica el control de un dominio y emite certificados. |
| **Alias (correo)** | Dirección sin buzón propio que reenvía a una o varias direcciones. |
| **APT** | Gestor de paquetes de Debian y Ubuntu (`apt install`, `apt update`). |
| **Backscatter** | Rebotes enviados a remitentes falsificados cuando un servidor acepta correo para direcciones inexistentes y luego lo devuelve. |
| **CA (Autoridad de Certificación)** | Entidad de confianza que firma certificados digitales. |
| **Certbot** | Cliente oficial de Let's Encrypt para obtener y renovar certificados. |
| **Chroot** | Técnica que encierra un proceso en una carpeta, impidiéndole ver el resto del sistema. Postfix la usa en `/var/spool/postfix`. |
| **CIDR** | Notación de rangos de IP con barra (`192.168.1.0/24`). |
| **CNAME** | Registro DNS que convierte un nombre en alias de otro nombre. |
| **Cron** | Programador de tareas periódicas de Linux. |
| **CRUD** | *Create, Read, Update, Delete*: las cuatro operaciones básicas sobre datos. |
| **CSRF** | Ataque que engaña al navegador de un usuario para enviar una petición no deseada a otra web; se evita con tokens. |
| **Daemon (demonio)** | Programa que se ejecuta en segundo plano prestando un servicio. |
| **DKIM** | Firma criptográfica de los correos salientes, verificable con una clave pública publicada en el DNS. |
| **DMARC** | Política DNS que indica qué hacer con correos que no superan SPF/DKIM alineados con el remitente visible, y dónde enviar informes. |
| **DNS** | Sistema que traduce nombres de dominio a direcciones IP y otros datos. |
| **DocumentRoot** | Carpeta cuyo contenido publica un VirtualHost de Apache. |
| **Dovecot** | Servidor IMAP/POP3; en este manual también entrega el correo (LMTP) y autentica (SASL). |
| **Envelope (sobre)** | Remitente y destinatarios usados en la transmisión SMTP (`MAIL FROM`, `RCPT TO`), distintos de las cabeceras visibles. |
| **Fail2ban** | Herramienta que bloquea IP tras detectar fallos de autenticación repetidos en los logs. |
| **FCrDNS** | Coincidencia entre el DNS inverso de una IP y el DNS directo de ese nombre. |
| **FQDN** | Nombre de dominio completo (`mail.ejemplo1.com`). |
| **FTP** | Protocolo de transferencia de archivos con una conexión de control (puerto 21) y conexiones de datos aparte. Sin cifrar es inseguro. |
| **FTPS** | FTP protegido con TLS. En modo explícito, el cliente pide el cifrado con `AUTH TLS` en el puerto 21. |
| **Hash** | Resultado de una función criptográfica de un solo sentido; se usa para guardar contraseñas sin almacenarlas en claro. |
| **HSTS** | Cabecera que obliga al navegador a usar siempre HTTPS con un sitio. |
| **IMAP** | Protocolo para consultar el correo manteniéndolo sincronizado en el servidor. |
| **InnoDB** | Motor de almacenamiento transaccional de MariaDB. |
| **Inyección SQL** | Ataque que introduce código SQL a través de datos no filtrados; se evita con consultas preparadas. |
| **Let's Encrypt** | Autoridad de certificación gratuita y automatizada. |
| **LMTP** | Protocolo de entrega local de correo, usado entre Postfix y Dovecot. |
| **Log** | Registro de eventos de un programa. |
| **Maildir** | Formato de buzón en el que cada mensaje es un archivo independiente. |
| **MariaDB** | Sistema gestor de bases de datos relacional, compatible con MySQL. |
| **MDA / MTA / MUA / MSA** | Agente de entrega, de transferencia, de usuario y de envío de correo (ver Módulo 6). |
| **Milter** | Protocolo para que Postfix pase los correos a filtros externos (OpenDKIM, antispam). |
| **MPM** | Módulo de Apache que define cómo atiende peticiones concurrentes (`event`, `prefork`, `worker`). |
| **Modo pasivo (FTP)** | Forma de abrir la conexión de datos FTP en la que el cliente se conecta a un puerto del rango que indica el servidor. Funciona detrás de NAT. |
| **MX (registro)** | Registro DNS que indica el servidor que recibe el correo de un dominio. |
| **open_basedir** | Directiva de PHP que restringe las carpetas a las que puede acceder el código. |
| **OpenDKIM** | Servicio que firma y verifica DKIM. |
| **PDO** | Interfaz de PHP para acceder a bases de datos de forma uniforme. |
| **PHP-FPM** | Gestor de procesos PHP que atiende peticiones FastCGI, con *pools* independientes. |
| **Pool (PHP-FPM)** | Grupo de procesos PHP con usuario, socket y configuración propios. |
| **POP3** | Protocolo para descargar el correo del servidor al dispositivo. |
| **Postfix** | Servidor SMTP (MTA) utilizado en este manual. |
| **Puerto** | Número que identifica un servicio dentro de un equipo (0–65535). |
| **PTR** | Registro de DNS inverso: de IP a nombre. |
| **Relay abierto** | Servidor de correo que reenvía mensajes de cualquiera a cualquiera; es una vulnerabilidad grave. |
| **SAN** | Campo de un certificado con la lista de nombres que cubre. |
| **SASL** | Marco de autenticación usado por SMTP para comprobar usuario y contraseña. |
| **Selector DKIM** | Nombre que identifica una clave DKIM (`mail` en `mail._domainkey`). |
| **SFTP** | Transferencia de archivos dentro de una conexión SSH (puerto 22). No tiene relación con FTP salvo el propósito. |
| **SMTP** | Protocolo de envío y transporte de correo electrónico. |
| **SNI** | Extensión de TLS que indica el nombre del sitio al iniciar la conexión, permitiendo varios certificados en una IP. |
| **Socket Unix** | Archivo especial para comunicar procesos del mismo equipo sin pasar por la red. |
| **SPF** | Registro DNS que enumera las IP autorizadas a enviar correo de un dominio. |
| **SSH** | Protocolo de acceso remoto cifrado. |
| **STARTTLS** | Comando que convierte una conexión en texto plano en cifrada. |
| **sudo** | Comando para ejecutar órdenes con privilegios de administrador. |
| **systemd / systemctl** | Gestor de servicios de Linux y su herramienta de control. |
| **TCP / UDP** | Protocolos de transporte: con conexión y garantías (TCP) o sin ellas (UDP). |
| **TLS** | Protocolo criptográfico que cifra y autentica las comunicaciones (sucesor de SSL). |
| **TTL** | Tiempo durante el que una respuesta DNS puede guardarse en caché. |
| **UFW** | Interfaz sencilla del cortafuegos de Linux. |
| **Usuario virtual** | Cuenta de correo que no es una cuenta del sistema operativo. |
| **VirtualHost** | Bloque de configuración de Apache que define un sitio web. |
| **vmail** | Usuario técnico de Linux propietario de todos los buzones virtuales. |
| **vsftpd** | Servidor FTP de Linux orientado a la seguridad; en este manual sirve FTPS con usuarios enjaulados en su web. |
| **XSS** | Ataque que inyecta JavaScript en una página; se evita escapando la salida con `htmlspecialchars`. |
