# Módulo 4. Instalación y configuración de MariaDB

## Objetivos del módulo

- Comprender qué es un sistema gestor de bases de datos relacional y cómo funciona MariaDB.
- Instalar MariaDB, protegerla y dejarla arrancando automáticamente.
- Trabajar en la consola SQL: crear bases de datos, tablas, usuarios y permisos.
- Crear una base de datos y un usuario independientes para cada VirtualHost.
- Cambiar contraseñas, hacer copias de seguridad, restaurarlas y comprobar conexiones.

## 4.1 Qué es MariaDB

**MariaDB** es un sistema gestor de bases de datos relacional (SGBD), nacido en 2009 como continuación libre de MySQL por parte de sus creadores originales. Es compatible con MySQL en la práctica totalidad de casos: los mismos comandos, el mismo protocolo y las mismas librerías de conexión. Ubuntu 24.04 incluye MariaDB 10.11, una versión de soporte a largo plazo.

Conceptos básicos:

| Concepto | Qué es | Ejemplo |
|---|---|---|
| Servidor | El programa `mariadbd`, que guarda los datos y atiende conexiones. | Servicio `mariadb`. |
| Base de datos | Un contenedor de tablas, normalmente una por aplicación. | `web1_db` |
| Tabla | Datos organizados en filas y columnas. | `contactos` |
| Columna | Un campo con un tipo de dato. | `email VARCHAR(150)` |
| Fila (registro) | Un elemento concreto. | (1, 'Ana', 'ana@...') |
| Clave primaria | Columna que identifica de forma única cada fila. | `id` |
| Usuario | Cuenta que se conecta, identificada por nombre **y** origen. | `'web1_user'@'localhost'` |
| Privilegio | Lo que un usuario puede hacer sobre qué objetos. | `SELECT` en `web1_db.*` |
| SQL | Lenguaje con el que se habla con la base de datos. | `SELECT * FROM contactos;` |

!!! warning "Atención"
    En MariaDB un usuario se identifica por **nombre y host** a la vez. `'web1_user'@'localhost'` y `'web1_user'@'%'` son dos usuarios distintos, con contraseñas y permisos distintos. `localhost` significa «conectándose desde el propio servidor»; `%` significa «desde cualquier sitio». En este manual solo usaremos `localhost`.

Cómo se conectará cada aplicación:

```text title="Diagrama"
  Pool PHP-FPM web1 (usuario Linux web1)                 Pool PHP-FPM web2 (usuario Linux web2)
           │  usuario BD: web1_user                                │  usuario BD: web2_user
           │                                                       │
           └──────────────► /run/mysqld/mysqld.sock ◄──────────────┘
                                     │
                         ┌───────────▼────────────┐
                         │   MariaDB (mariadbd)    │  escucha solo en 127.0.0.1:3306
                         │                         │  y en el socket Unix
                         │  web1_db  ◄── solo web1_user tiene permisos
                         │  web2_db  ◄── solo web2_user tiene permisos
                         │  mysql    ◄── usuarios y privilegios (solo root)
                         └─────────────────────────┘
```

### PASO 4.1 – Instalar MariaDB

**Objetivo:** instalar el servidor y el cliente de MariaDB.

**Comando:** 

```bash title="Terminal"
sudo apt install -y mariadb-server mariadb-client
```

**Análisis del comando:** `mariadb-server` instala el servidor (`mariadbd`) y sus herramientas; `mariadb-client` el cliente de consola `mariadb` y utilidades como `mariadb-dump`.

**Resultado esperado:** termina con `Created symlink ... mariadb.service` y sin errores.

**Comprobación:** 

```bash title="Terminal"
mariadb --version
systemctl status mariadb --no-pager
systemctl is-enabled mariadb
```

```text title="Salida esperada"
mariadb  Ver 15.2 Distrib 10.11.13-MariaDB, for debian-linux-gnu (x86_64) using  EditLine wrapper
● mariadb.service - MariaDB 10.11.13 database server
     Loaded: loaded (/usr/lib/systemd/system/mariadb.service; enabled; preset: enabled)
     Active: active (running) since ...
enabled
```

Si no estuviera habilitado: `sudo systemctl enable --now mariadb`.

**Siguiente paso:** comprobar dónde escucha.

### PASO 4.2 – Verificar que MariaDB no es accesible desde Internet

**Objetivo:** confirmar que MariaDB solo escucha en la interfaz local.

**Explicación:** una base de datos expuesta a Internet es uno de los fallos de seguridad más graves y más explotados. Ubuntu configura MariaDB para escuchar solo en `127.0.0.1`. Lo comprobamos.

**Comando:** 

```bash title="Terminal"
sudo ss -tlnp | grep -E 'mariadbd|mysqld'
grep -E '^\s*bind-address' /etc/mysql/mariadb.conf.d/50-server.cnf
```

**Análisis del comando:** `ss -tlnp` lista los sockets TCP (`t`) en escucha (`l`), en formato numérico (`n`) y con el proceso (`p`). El `grep` busca la directiva `bind-address` en el archivo de configuración del servidor.

**Resultado esperado:** 

```text title="Salida esperada"
LISTEN 0  80  127.0.0.1:3306  0.0.0.0:*  users:(("mariadbd",pid=4120,fd=21))
bind-address            = 127.0.0.1
```

**Comprobación:** la dirección debe ser `127.0.0.1:3306`, **nunca** `0.0.0.0:3306` ni `*:3306`.

**Errores habituales:** si aparece `0.0.0.0`, edita `/etc/mysql/mariadb.conf.d/50-server.cnf`, deja la línea `bind-address = 127.0.0.1` dentro de la sección `[mysqld]` y reinicia con `sudo systemctl restart mariadb`.

!!! tip "Consejo"
    La estructura de configuración de MariaDB en Ubuntu: `/etc/mysql/my.cnf` incluye `/etc/mysql/conf.d/` y `/etc/mysql/mariadb.conf.d/`. El archivo que más tocarás es `50-server.cnf` (sección `[mysqld]`). Para tus ajustes propios es buena práctica crear un archivo nuevo, por ejemplo `/etc/mysql/mariadb.conf.d/99-local.cnf`, en lugar de modificar los del paquete.

**Siguiente paso:** proteger la instalación.

### PASO 4.3 – Proteger la instalación con mariadb-secure-installation

**Objetivo:** eliminar los elementos inseguros de una instalación nueva.

**Explicación:** en Ubuntu, el usuario `root` de MariaDB se autentica por **unix_socket**: solo puede entrar quien sea root en Linux (con `sudo`), sin contraseña de MariaDB. Es un método muy seguro y lo mantendremos. El asistente elimina usuarios anónimos, el acceso remoto de root y la base de datos de pruebas.

**Comando:** 

```bash title="Terminal"
sudo mariadb-secure-installation
```

Responde así:

| Pregunta | Respuesta | Por qué |
|---|---|---|
| Enter current password for root (enter for none) | Intro | No hay contraseña: root usa unix_socket. |
| Switch to unix_socket authentication [Y/n] | n | Ya lo usa. Responder `n` deja la configuración tal cual. |
| Change the root password? [Y/n] | n | Con unix_socket no hace falta contraseña. |
| Remove anonymous users? [Y/n] | Y | Los usuarios anónimos permiten entrar sin nombre. |
| Disallow root login remotely? [Y/n] | Y | Root solo desde el propio servidor. |
| Remove test database and access to it? [Y/n] | Y | La base `test` es accesible por cualquiera. |
| Reload privilege tables now? [Y/n] | Y | Aplica los cambios. |

**Resultado esperado:** termina con `Thanks for using MariaDB!`.

**Comprobación:** entra como root sin contraseña (gracias a sudo) e intenta entrar como root **sin** sudo:

```bash title="Terminal"
sudo mariadb -e "SELECT user, host, plugin FROM mysql.user;"
mariadb -u root
```

```text title="Salida esperada"
+-------------+-----------+-----------------------+
| User        | Host      | plugin                |
+-------------+-----------+-----------------------+
| mariadb.sys | localhost | mysql_native_password |
| root        | localhost | mysql_native_password |
| mysql       | localhost | mysql_native_password |
+-------------+-----------+-----------------------+
ERROR 1698 (28000): Access denied for user 'root'@'localhost'
```

El error final es **la prueba de que funciona**: sin ser root en Linux, no se puede entrar como root en MariaDB. (La columna `plugin` muestra el método principal; root tiene además `unix_socket` como alternativa, que es la que se usa con sudo.)

**Siguiente paso:** conocer la consola.

## 4.2 La consola de MariaDB

### PASO 4.4 – Acceder a la consola y orientarse

**Objetivo:** entrar a la consola SQL y aprender sus comandos básicos.

**Comando:** 

```bash title="Terminal"
sudo mariadb
```

Verás el indicador `MariaDB [(none)]>`. `(none)` indica que no has seleccionado ninguna base de datos. Prueba:

```sql title="SQL"
SELECT VERSION();
SHOW DATABASES;
SELECT CURRENT_USER();
STATUS;
```

Reglas de la consola:

- Toda instrucción SQL termina en **punto y coma** `;`. Si pulsas Intro sin él, verás `->`: MariaDB espera que termines. Escribe `;` e Intro.
- Las palabras clave (`SELECT`, `CREATE`) pueden ir en mayúsculas o minúsculas; por convención se escriben en mayúsculas.
- Los nombres de bases de datos y tablas **distinguen mayúsculas** en Linux.
- Los textos van entre comillas simples: `'texto'`.
- Para salir: `EXIT;` o **Ctrl+D**. Para cancelar una instrucción a medio escribir: **Ctrl+C**.

Resultado esperado de `SHOW DATABASES;`:

```text title="Salida esperada"
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
4 rows in set (0.001 sec)
```

Estas bases son internas: `mysql` guarda usuarios y privilegios; las demás ofrecen información del servidor. No las modifiques directamente.

**Siguiente paso:** generar contraseñas seguras.

## 4.3 Bases de datos y usuarios independientes para cada VirtualHost

### PASO 4.5 – Generar contraseñas seguras para los usuarios de base de datos

**Objetivo:** crear dos contraseñas aleatorias y robustas.

**Comando (en el terminal de Linux, **fuera** de la consola de MariaDB):** 

```bash title="Terminal"
openssl rand -base64 24
openssl rand -base64 24
```

**Análisis del comando:** `openssl rand` genera bytes aleatorios criptográficamente seguros; `-base64 24` genera 24 bytes y los codifica en 32 caracteres legibles.

**Resultado esperado:** dos líneas como `p3Q9xV2r...` (cada vez distintas). Guárdalas en un gestor de contraseñas: la primera será la de `web1_user` y la segunda la de `web2_user`.

!!! warning "Atención"
    En este manual las representamos como `<CLAVE_WEB1>` y `<CLAVE_WEB2>`. Sustitúyelas siempre por las tuyas. Evita contraseñas con comillas simples `'` o barras invertidas `\`: complican su uso dentro de SQL y PHP.

**Siguiente paso:** crear las bases de datos.

### PASO 4.6 – Crear las bases de datos web1_db y web2_db

**Objetivo:** crear una base de datos independiente para cada sitio.

**Comando (dentro de `sudo mariadb`):** 

```sql title="SQL"
CREATE DATABASE web1_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE DATABASE web2_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
SHOW DATABASES;
```

**Análisis del comando:** 

- `CREATE DATABASE web1_db`: crea la base de datos.
- `CHARACTER SET utf8mb4`: codificación de caracteres. `utf8mb4` admite todos los caracteres Unicode, incluidas tildes, eñes y emojis. (El antiguo `utf8` de MySQL solo admite 3 bytes por carácter y falla con emojis.)
- `COLLATE utf8mb4_unicode_ci`: reglas de comparación y ordenación. `ci` = *case insensitive*: `'Ana'` y `'ana'` se consideran iguales al buscar.

**Resultado esperado:** `Query OK, 1 row affected` por cada una, y ambas aparecen en `SHOW DATABASES`.

**Errores habituales:** `ERROR 1007 (HY000): Can't create database 'web1_db'; database exists`. Ya existía; puedes usar `CREATE DATABASE IF NOT EXISTS ...` para evitar el error.

**Siguiente paso:** crear los usuarios.

### PASO 4.7 – Crear los usuarios de base de datos

**Objetivo:** crear `web1_user` y `web2_user`, que solo podrán conectarse desde el propio servidor.

**Comando (dentro de `sudo mariadb`):** 

```sql title="SQL"
CREATE USER 'web1_user'@'localhost' IDENTIFIED BY '<CLAVE_WEB1>';
CREATE USER 'web2_user'@'localhost' IDENTIFIED BY '<CLAVE_WEB2>';
SELECT user, host FROM mysql.user WHERE user LIKE 'web%';
```

**Análisis del comando:** 

- `CREATE USER 'web1_user'@'localhost'`: crea el usuario `web1_user` que solo puede conectarse desde `localhost` (socket Unix o 127.0.0.1).
- `IDENTIFIED BY '...'`: su contraseña. MariaDB no la guarda en claro, sino un *hash*.

**Resultado esperado:** 

```text title="Salida esperada"
+-----------+-----------+
| User      | Host      |
+-----------+-----------+
| web1_user | localhost |
| web2_user | localhost |
+-----------+-----------+
```

**Siguiente paso:** asignar permisos limitados.

### PASO 4.8 – Asignar permisos específicos a cada usuario

**Objetivo:** que `web1_user` solo pueda trabajar con `web1_db` y `web2_user` solo con `web2_db`, y con los privilegios mínimos necesarios.

**Explicación:** una aplicación web normal solo necesita leer, insertar, modificar y borrar filas. No necesita borrar tablas enteras, crear usuarios ni tocar otras bases de datos. Si un atacante encontrara una inyección SQL en la web 1, con estos permisos no podría ni ver la base de datos de la web 2.

Distinguimos dos niveles de permisos:

| Privilegio | Permite | ¿Lo damos? |
|---|---|---|
| `SELECT` | Leer filas. | Sí |
| `INSERT` | Añadir filas. | Sí |
| `UPDATE` | Modificar filas. | Sí |
| `DELETE` | Borrar filas. | Sí |
| `CREATE`, `ALTER`, `INDEX`, `DROP` | Crear, modificar y borrar tablas e índices. | Solo si la aplicación instala o actualiza su propio esquema (WordPress, por ejemplo, los necesita). En nuestro ejemplo, no: creamos las tablas como administradores. |
| `CREATE TEMPORARY TABLES`, `LOCK TABLES` | Tablas temporales y bloqueos. | Solo si la aplicación lo requiere. |
| `ALL PRIVILEGES` | Todo sobre la base indicada. | No. |
| `GRANT OPTION`, `SUPER`, `FILE`, `PROCESS` | Administración del servidor. | **Nunca** a una aplicación. |

**Comando (dentro de `sudo mariadb`):** 

```sql title="SQL"
GRANT SELECT, INSERT, UPDATE, DELETE ON web1_db.* TO 'web1_user'@'localhost';
GRANT SELECT, INSERT, UPDATE, DELETE ON web2_db.* TO 'web2_user'@'localhost';
FLUSH PRIVILEGES;
SHOW GRANTS FOR 'web1_user'@'localhost';
SHOW GRANTS FOR 'web2_user'@'localhost';
```

**Análisis del comando:** 

- `GRANT privilegios ON web1_db.* TO usuario`: concede esos privilegios sobre todas las tablas (`*`) de `web1_db`.
- `FLUSH PRIVILEGES`: recarga las tablas de permisos. Con `GRANT` no es estrictamente necesario, pero no hace daño y es una buena costumbre.
- `SHOW GRANTS`: muestra los permisos efectivos.

**Resultado esperado:** 

```text title="Salida esperada"
+------------------------------------------------------------------------------------------------+
| Grants for web1_user@localhost                                                                 |
+------------------------------------------------------------------------------------------------+
| GRANT USAGE ON *.* TO `web1_user`@`localhost` IDENTIFIED BY PASSWORD '*A1B2C3...'              |
| GRANT SELECT, INSERT, UPDATE, DELETE ON `web1_db`.* TO `web1_user`@`localhost`                 |
+------------------------------------------------------------------------------------------------+
```

`USAGE ON *.*` significa «puede conectarse, pero sin privilegios globales». Es correcto.

**Siguiente paso:** crear las tablas de las aplicaciones.

### PASO 4.9 – Crear la estructura de tablas de cada aplicación

**Objetivo:** crear la tabla `contactos` que usará la aplicación PHP del Módulo 5.

**Explicación:** nuestra aplicación de ejemplo será una agenda de contactos con un pequeño formulario. Cada base de datos tendrá la misma estructura, pero datos completamente independientes.

**Comando (dentro de `sudo mariadb`):** 

```sql title="SQL"
USE web1_db;
CREATE TABLE contactos (
    id         INT UNSIGNED NOT NULL AUTO_INCREMENT,
    nombre     VARCHAR(100) NOT NULL,
    email      VARCHAR(150) NOT NULL,
    telefono   VARCHAR(30)  NULL,
    notas      TEXT         NULL,
    creado_en  TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (id),
    UNIQUE KEY uk_email (email)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

INSERT INTO contactos (nombre, email, telefono, notas)
VALUES ('Ana Pérez', 'ana@ejemplo1.com', '600111222', 'Contacto de prueba de la web 1');

USE web2_db;
CREATE TABLE contactos LIKE web1_db.contactos;
INSERT INTO contactos (nombre, email, telefono, notas)
VALUES ('Luis Gómez', 'luis@ejemplo2.com', '600333444', 'Contacto de prueba de la web 2');
```

**Análisis del comando:** 

| Elemento | Significado |
|---|---|
| `USE web1_db;` | Selecciona la base de datos sobre la que trabajarán las instrucciones siguientes. |
| `INT UNSIGNED` | Número entero sin signo (0 a 4.294.967.295). |
| `NOT NULL` | El campo es obligatorio. |
| `AUTO_INCREMENT` | MariaDB asigna automáticamente 1, 2, 3... |
| `VARCHAR(100)` | Texto de longitud variable, hasta 100 caracteres. |
| `TEXT` | Texto largo (hasta 64 KB). |
| `TIMESTAMP ... DEFAULT CURRENT_TIMESTAMP` | Fecha y hora, rellenada automáticamente al insertar. |
| `PRIMARY KEY (id)` | `id` identifica cada fila de forma única. |
| `UNIQUE KEY uk_email (email)` | No puede haber dos contactos con el mismo email. |
| `ENGINE=InnoDB` | Motor de almacenamiento con transacciones y recuperación ante fallos (el predeterminado). |
| `CREATE TABLE contactos LIKE web1_db.contactos` | Copia la estructura (no los datos) de otra tabla. |

**Comprobación:** 

```sql title="SQL"
SELECT * FROM web1_db.contactos;
SELECT * FROM web2_db.contactos;
DESCRIBE web1_db.contactos;
EXIT;
```

Cada consulta debe devolver su propio contacto.

**Siguiente paso:** comprobar que los permisos funcionan.

### PASO 4.10 – Comprobar las conexiones y el aislamiento entre usuarios

**Objetivo:** demostrar que cada usuario entra en su base de datos y **no** puede acceder a la otra.

**Comando (en el terminal de Linux):** 

```bash title="Terminal"
mariadb -u web1_user -p web1_db -e "SELECT nombre, email FROM contactos;"
mariadb -u web1_user -p web2_db -e "SELECT * FROM contactos;"
mariadb -u web1_user -p web1_db -e "DROP TABLE contactos;"
```

**Análisis del comando:** 

- `-u web1_user`: usuario con el que conectar.
- `-p`: pide la contraseña de forma interactiva (no la escribas en la línea de comandos: quedaría en el historial).
- `web1_db`: base de datos a usar.
- `-e "..."`: ejecuta la instrucción y sale.

**Resultado esperado:** 

```text title="Salida esperada"
+-----------+------------------+
| nombre    | email            |
+-----------+------------------+
| Ana Pérez | ana@ejemplo1.com |
+-----------+------------------+
ERROR 1044 (42000): Access denied for user 'web1_user'@'localhost' to database 'web2_db'
ERROR 1142 (42000): DROP command denied to user 'web1_user'@'localhost' for table `web1_db`.`contactos`
```

Las dos denegaciones son exactamente lo que queremos: aislamiento entre webs y protección contra borrados de tablas.

**Errores habituales:** 

- `ERROR 1045 (28000): Access denied for user 'web1_user'@'localhost' (using password: YES)`: contraseña incorrecta o usuario creado con otro host. Revisa con `SELECT user,host FROM mysql.user;`.
- `ERROR 2002 (HY000): Can't connect to local server through socket '/run/mysqld/mysqld.sock'`: MariaDB no está funcionando (`systemctl status mariadb`).

**Siguiente paso:** aprender a cambiar contraseñas.

### PASO 4.11 – Modificar contraseñas

**Objetivo:** cambiar la contraseña de un usuario de base de datos (por ejemplo, si sospechas que se ha filtrado).

**Comando (dentro de `sudo mariadb`):** 

```sql title="SQL"
ALTER USER 'web1_user'@'localhost' IDENTIFIED BY '<NUEVA_CLAVE_WEB1>';
FLUSH PRIVILEGES;
```

**Análisis del comando:** `ALTER USER ... IDENTIFIED BY` sustituye la contraseña. A partir de ese momento, la aplicación dejará de conectar hasta que actualices su archivo de configuración (Módulo 5, `config.php`).

**Comprobación:** `mariadb -u web1_user -p web1_db -e "SELECT 1;"` con la nueva clave debe devolver `1`.

Otras operaciones de administración de usuarios:

```sql title="SQL"
-- Quitar un privilegio concreto
REVOKE DELETE ON web1_db.* FROM 'web1_user'@'localhost';
-- Volver a concederlo
GRANT DELETE ON web1_db.* TO 'web1_user'@'localhost';
-- Bloquear temporalmente una cuenta sin borrarla
ALTER USER 'web1_user'@'localhost' ACCOUNT LOCK;
ALTER USER 'web1_user'@'localhost' ACCOUNT UNLOCK;
-- Borrar un usuario
DROP USER 'usuario_obsoleto'@'localhost';
-- Ver qué usuarios están conectados ahora mismo
SHOW PROCESSLIST;
```

**Siguiente paso:** copias de seguridad.

## 4.4 Copias de seguridad y restauración

Una base de datos sin copia de seguridad es una base de datos que vas a perder. La herramienta estándar es **mariadb-dump** (antes `mysqldump`), que genera un archivo de texto con todas las instrucciones SQL necesarias para reconstruir la base de datos.

### PASO 4.12 – Hacer una copia de seguridad manual

**Objetivo:** exportar `web1_db` a un archivo comprimido.

**Comando:** 

```bash title="Terminal"
sudo mkdir -p /var/backups/mariadb
sudo chmod 700 /var/backups/mariadb
sudo mariadb-dump --single-transaction --routines --triggers --events --databases web1_db | gzip | sudo tee /var/backups/mariadb/web1_db_$(date +%F_%H%M).sql.gz > /dev/null
ls -lh /var/backups/mariadb/
```

**Análisis del comando:** 

- `mariadb-dump`: exporta la base de datos (lo ejecutamos con sudo para usar la cuenta root por unix_socket).
- `--single-transaction`: hace la copia como una «foto» coherente sin bloquear las tablas InnoDB; la web sigue funcionando mientras tanto.
- `--routines --triggers --events`: incluye procedimientos almacenados, disparadores y eventos programados.
- `--databases web1_db`: incluye en el archivo la instrucción `CREATE DATABASE` y `USE`, de modo que la restauración recrea la base si no existe.
- `| gzip`: comprime (los volcados SQL se reducen mucho).
- `$(date +%F_%H%M)`: inserta la fecha y hora actuales en el nombre, por ejemplo `2026-11-08_2310`.
- `chmod 700` en la carpeta: las copias contienen todos tus datos, solo root debe leerlas.

**Resultado esperado:** 

```text title="Salida esperada"
-rw-r--r-- 1 root root 1,2K nov  8 23:10 web1_db_2026-11-08_2310.sql.gz
```

**Comprobación:** mira el contenido sin descomprimir el archivo:

```bash title="Terminal"
sudo zcat /var/backups/mariadb/web1_db_*.sql.gz | head -40
```

Verás instrucciones `CREATE DATABASE`, `CREATE TABLE` e `INSERT INTO`.

Para copiar **todas** las bases de datos de una vez:

```bash title="Terminal"
sudo mariadb-dump --single-transaction --routines --triggers --events --all-databases | gzip | sudo tee /var/backups/mariadb/todas_$(date +%F).sql.gz > /dev/null
```

**Siguiente paso:** restaurar.

### PASO 4.13 – Restaurar una base de datos

**Objetivo:** recuperar `web1_db` desde la copia, simulando una pérdida de datos.

**Comando:** primero simulamos el desastre borrando la base de datos (**solo en tu laboratorio o tras verificar la copia**):

```bash title="Terminal"
sudo mariadb -e "DROP DATABASE web1_db;"
sudo mariadb -e "SHOW DATABASES;"
```

Ahora restauramos:

```bash title="Terminal"
sudo sh -c 'zcat /var/backups/mariadb/web1_db_2026-11-08_2310.sql.gz | mariadb'
```

**Análisis del comando:** `zcat` descomprime y envía el SQL a la consola `mariadb`, que ejecuta las instrucciones una tras otra. Como el volcado se hizo con `--databases`, el propio archivo contiene `CREATE DATABASE` y `USE web1_db`. Usamos `sh -c '...'` para que toda la tubería se ejecute con privilegios de root. Sustituye el nombre del archivo por el tuyo.

**Comprobación:** 

```bash title="Terminal"
sudo mariadb -e "SELECT * FROM web1_db.contactos;"
mariadb -u web1_user -p web1_db -e "SELECT COUNT(*) FROM contactos;"
```

Los datos han vuelto **y** el usuario `web1_user` sigue pudiendo acceder (los usuarios y permisos viven en la base `mysql`, que no hemos borrado).

!!! warning "Atención"
    Si restauras un volcado hecho **sin** `--databases` (solo con el nombre de la base), el archivo no contiene `CREATE DATABASE`. En ese caso crea primero la base vacía y restaura indicándola: `sudo mariadb -e "CREATE DATABASE web1_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"` y después `zcat archivo.sql.gz | sudo mariadb web1_db`.

!!! tip "Consejo"
    Una copia que nunca has restaurado es solo una esperanza. Programa una prueba de restauración en un servidor de pruebas al menos una vez al mes. La automatización diaria de las copias se configura en el Módulo 12.

**Errores habituales:** `ERROR 1049 (42000): Unknown database 'web1_db'` al restaurar: el volcado no incluía `CREATE DATABASE` (ver la nota anterior).

**Siguiente paso:** Módulo 5, conectar PHP con MariaDB.

## 4.5 Referencia rápida de SQL

| Operación | Sintaxis |
|---|---|
| Crear base de datos | `CREATE DATABASE nombre CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;` |
| Borrar base de datos | `DROP DATABASE nombre;` |
| Crear usuario | `CREATE USER 'u'@'localhost' IDENTIFIED BY 'clave';` |
| Conceder permisos | `GRANT SELECT, INSERT, UPDATE, DELETE ON bd.* TO 'u'@'localhost';` |
| Ver permisos | `SHOW GRANTS FOR 'u'@'localhost';` |
| Retirar permisos | `REVOKE DELETE ON bd.* FROM 'u'@'localhost';` |
| Cambiar contraseña | `ALTER USER 'u'@'localhost' IDENTIFIED BY 'nueva';` |
| Borrar usuario | `DROP USER 'u'@'localhost';` |
| Listar tablas | `SHOW TABLES;` |
| Estructura de una tabla | `DESCRIBE tabla;` |
| Insertar | `INSERT INTO t (c1, c2) VALUES ('a', 'b');` |
| Leer | `SELECT c1, c2 FROM t WHERE id = 1 ORDER BY c1 LIMIT 10;` |
| Modificar | `UPDATE t SET c1 = 'x' WHERE id = 1;` |
| Borrar filas | `DELETE FROM t WHERE id = 1;` |

!!! warning "Atención"
    Un `UPDATE` o `DELETE` **sin** `WHERE` afecta a **todas** las filas de la tabla. Antes de ejecutar uno, ejecuta el `SELECT` equivalente con el mismo `WHERE` para ver qué filas vas a tocar.

## 4.6 Ejercicios prácticos del Módulo 4

1. Crea una tabla `productos` (id, nombre, precio DECIMAL(10,2), stock INT) en `web2_db`, inserta tres productos y practica `SELECT` con `WHERE`, `ORDER BY` y `LIMIT`.
2. Intenta conectar como `web2_user` a `web1_db` y documenta el error.
3. Quita a `web1_user` el permiso `DELETE`, intenta borrar un contacto y vuelve a concederlo.
4. Haz una copia de `web2_db`, borra un registro, restaura la copia y verifica que el registro ha vuelto.
5. Explica por qué es más seguro `'web1_user'@'localhost'` que `'web1_user'@'%'` y por qué no damos `ALL PRIVILEGES` a las aplicaciones.
