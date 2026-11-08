# Administración de Servidores Linux

Manual de formación para montar **desde cero** un servidor Linux (Ubuntu Server 24.04 LTS) que aloja varias webs con Apache y PHP-FPM, bases de datos MariaDB independientes y un servidor de correo completo con Postfix, Dovecot, OpenDKIM, SPF, DMARC y certificados de Let's Encrypt.

- **Leer en la web:** https://davidzabalaasir-create.github.io/manual-servidores-linux/
- **Descargar:** [PDF](docs/descargas/Manual_Administracion_Servidores_Linux.pdf) · [Word](docs/descargas/Manual_Administracion_Servidores_Linux.docx)

## Contenido

| Módulo | Tema |
|---|---|
| 1 | Fundamentos de infraestructura (IP, DNS, puertos, SSH, usuarios y permisos) |
| 2 | Apache |
| 3 | Virtual Hosting y HTTPS |
| 4 | MariaDB |
| 5 | PHP-FPM y aplicaciones conectadas a MariaDB |
| 6 | Postfix (SMTP) |
| 7 | Dovecot (IMAP/POP3) |
| 8 | Cuentas de correo virtuales |
| 9 | DNS del correo: MX, SPF, DKIM, DMARC y PTR |
| 10 | Certificados SSL/TLS |
| 11 | Integración completa de los servicios |
| 12 | Seguridad y mantenimiento |
| 13 | Pruebas y diagnóstico |
| 14 | Proyecto final |

## Ver el manual en tu ordenador

Necesitas Python 3.

```bash
pip install -r requirements.txt
mkdocs serve
```

Abre http://127.0.0.1:8000. Los cambios en `docs/` se ven al instante.

## Publicarlo en GitHub Pages

1. Crea en GitHub un repositorio vacío llamado `manual-servidores-linux`.
2. En `mkdocs.yml` comprueba que `site_url` y `repo_url` usan tu usuario de GitHub.
3. Sube el proyecto:

   ```bash
   git init -b main
   git add .
   git commit -m "Primera versión del manual"
   git remote add origin https://github.com/davidzabalaasir-create/manual-servidores-linux.git
   git push -u origin main
   ```

4. La acción `.github/workflows/publicar.yml` compila la web y la publica en la rama `gh-pages`. Cuando termine (pestaña **Actions**), ve a **Settings → Pages**, en **Source** elige **Deploy from a branch**, rama `gh-pages`, carpeta `/ (root)`, y guarda.
5. En uno o dos minutos el manual estará en `https://davidzabalaasir-create.github.io/manual-servidores-linux/`.

A partir de ahí, cada `git push` a `main` vuelve a publicar la web automáticamente.

## Estructura

```text
manual-servidores-linux/
├── mkdocs.yml                  configuración del sitio y menú
├── requirements.txt            dependencias (MkDocs Material)
├── docs/                       una página Markdown por módulo
│   └── descargas/              PDF y Word del manual
└── .github/workflows/          publicación automática
```
