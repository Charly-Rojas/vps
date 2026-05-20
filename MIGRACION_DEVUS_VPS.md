# Migracion DevUs a VPS Hostinger con Virtualmin

Fecha de documentacion: 2026-05-16

## Regla operativa para futuras sesiones

Antes de hacer cambios relacionados con este VPS, dominios, DNS, correo, Virtualmin, WordPress, Node.js u otras apps alojadas aqui, leer este documento.

Despues de cualquier cambio importante o funcional, actualizar este mismo documento con:

```text
Fecha
Cambio realizado
Dominio o servicio afectado
Comandos/configuracion importante
Validacion realizada
Pendientes o riesgos
```

Este archivo debe tratarse como la bitacora principal de operacion del VPS.

No es necesario registrar absolutamente todo. Evitar entradas por verificaciones rutinarias, lecturas de archivos, consultas sin cambios, pruebas exploratorias menores o acciones que no alteren la operacion. La bitacora debe mantenerse enfocada en cambios con impacto funcional, configuracion importante, migraciones, incidentes, decisiones operativas y pendientes reales.

## Resumen ejecutivo

Se migro la operacion principal desde un plan reseller de HostGator hacia un VPS en Hostinger con Webmin/Virtualmin.

El VPS nuevo es:

```text
IP publica: 193.46.199.28
Hostname usado: server.devus.mx
Panel: Webmin + Virtualmin
Sistema: AlmaLinux
Acceso SSH: ssh -4 -i ~/.ssh/devus_vps root@193.46.199.28
```

La estrategia fue:

1. Descargar/usar backups completos de cPanel.
2. Revisar integridad y contenido de los `.tar.gz`.
3. Escanear los backups con ClamAV antes de migrar.
4. Migrar cada dominio con `virtualmin migrate-domain --type cpanel`.
5. Validar sitio, base de datos, usuarios de correo y DNS.
6. Cambiar DNS publico a Cloudflare.
7. Resolver envio de correo saliente usando SMTP2GO para dominios verificados.

## Dominios migrados o revisados

Dominios migrados al VPS:

```text
dyr.com.mx
devus.mx
fortiguardia.com
frozenandfire.com
innovajb.com
quieroteamup.com
sjpabogados.com
```

Dominios revisados en Cloudflare / Virtualmin:

```text
creativobusiness.com
lexgarantia.com
```

Dominio omitido por decision:

```text
compteq.mx
```

## Proceso base usado para migrar desde cPanel

Para cada backup de cPanel se siguio este flujo:

1. Crear carpeta temporal de escaneo.

```bash
mkdir -p /tmp/dominio_scan
```

2. Extraer el backup completo.

```bash
tar -xzf backup-dominio.tar.gz -C /tmp/dominio_scan
```

3. Actualizar firmas de ClamAV si estaba disponible.

```bash
freshclam
```

4. Escanear el contenido.

```bash
clamscan -r --infected --log=/tmp/dominio_clamscan.log /tmp/dominio_scan
```

5. Si el escaneo salia limpio, subir el backup al VPS.

```bash
rsync -avh --progress -e "ssh -i ~/.ssh/devus_vps" backup-dominio.tar.gz root@193.46.199.28:/root/migrations/cpanel/
```

6. Validar hash local y remoto.

```bash
sha256sum backup-dominio.tar.gz
ssh -4 -i ~/.ssh/devus_vps root@193.46.199.28 'sha256sum /root/migrations/cpanel/backup-dominio.tar.gz'
```

7. Migrar con Virtualmin.

```bash
virtualmin migrate-domain \
  --source /root/migrations/cpanel/backup-dominio.tar.gz \
  --type cpanel \
  --domain dominio.com \
  --user usuario \
  --webmin
```

8. Validar sitio y servicios.

```bash
virtualmin list-domains
virtualmin list-users --domain dominio.com
virtualmin list-databases --domain dominio.com
curl -I https://dominio.com
```

## Caso especial: dyr.com.mx

En `dyr.com.mx` se detecto una alerta de ClamAV:

```text
Win.Trojan.Suspect-34
```

El archivo infectado estaba dentro de correo spam de una cuenta migrada, no dentro de WordPress. Se elimino del backup sanitizado y se genero una copia limpia:

```text
backup-5.13.2026_23-17-18_dyr.sanitized.tar.gz
```

Despues se rescaneo, salio limpio y se migro a Virtualmin.

Tambien se deshabilitaron plugins sospechosos o rotos renombrandolos en el VPS:

```text
content-optimizer-v2.disabled
wp-performance-tools.disabled
```

## WordPress

En los dominios WordPress se valido:

```text
document root
wp-config.php
conexion a base de datos
respuesta HTTP/HTTPS
plugins sospechosos o incompatibles
```

Bases de datos relevantes migradas:

```text
dyr.com.mx: dyr_wp711 o equivalente migrado
fortiguardia.com: fortigua_wp
frozenandfire.com: frozenan_wp90
sjpabogados.com: sjpaboga_wp945
quieroteamup.com: teamup_wp
devus.mx: multiples bases, incluyendo devusmx_wp512 y otras apps
innovajb.com: innovajb_u865970393_innovajb
```

`innovajb.com` no es WordPress; es una aplicacion PHP personalizada.

## DNS y Cloudflare

La gestion publica de DNS quedo en Cloudflare.

Regla general aplicada:

```text
Sitio web raiz: A hacia 193.46.199.28, proxied si es web
www: CNAME hacia raiz, proxied si es web
mail: A hacia 193.46.199.28, DNS only
MX: hacia mail.dominio.com
SPF: autorizar VPS y relay SMTP
DKIM: mantener/agregar registros generados por Virtualmin o relay
DMARC: agregar politica inicial
```

Importante:

Los registros de correo no deben estar proxied en Cloudflare:

```text
mail
MX
autodiscover si se usa para correo
webmail si se quiere acceso directo al servidor
```

Para servicios de correo, usar siempre `DNS only`.

## Estado DNS revisado

Se revisaron exportaciones de Cloudflare para estos dominios:

```text
creativobusiness.com
devus.mx
dyr.com.mx
fortiguardia.com
frozenandfire.com
innovajb.com
lexgarantia.com
quieroteamup.com
sjpabogados.com
```

Hallazgos principales:

- `devus.mx`, `dyr.com.mx`, `frozenandfire.com`, `innovajb.com`, `sjpabogados.com` quedaron correctos para web y correo.
- `fortiguardia.com` tenia `mail` proxied; debia estar DNS only.
- `quieroteamup.com` tenia `mail` proxied; debia estar DNS only.
- `lexgarantia.com` no quedo considerado como migrado/listo en Virtualmin. No tratarlo como cerrado hasta migrarlo y confirmar correo.

## Correo entrante

Se crearon y probaron buzones de prueba:

```text
prueba@creativobusiness.com
prueba@devus.mx
prueba@dyr.com.mx
prueba@fortiguardia.com
prueba@frozenandfire.com
prueba@innovajb.com
prueba@quieroteamup.com
prueba@sjpabogados.com
```

Las contrasenas de estos buzones se guardaron en el VPS:

```text
/root/migrations/prueba-mailboxes-20260516-182157.txt
```

No incluir esas contrasenas en documentacion publica.

## Problema de correo saliente directo

Inicialmente el VPS intentaba enviar correo directamente a Gmail/Outlook por puerto `25`.

Se detecto este rebote de Gmail:

```text
550-5.7.1 The IP you're using to send mail is not authorized to send email directly to our servers
```

La causa inicial fue que Postfix estaba enviando por IPv6, pero SPF solo autorizaba IPv4.

Se aplico:

```bash
postconf -e "inet_protocols = ipv4" "smtp_address_preference = ipv4"
systemctl restart postfix
```

Despues, algunos correos si salieron, pero otros quedaron con timeouts al conectar a Gmail/Outlook por puerto `25`:

```text
connect to gmail-smtp-in.l.google.com:25: Connection timed out
connect to hotmail-com.olc.protection.outlook.com:25: Connection timed out
```

Conclusion:

Enviar correo directo desde VPS por puerto `25` no es suficientemente confiable para produccion. La solucion correcta es usar un relay SMTP autenticado.

## SMTP2GO

Se configuro SMTP2GO para los dominios verificados en la cuenta:

```text
devus.mx
dyr.com.mx
fortiguardia.com
quieroteamup.com
sjpabogados.com
```

Configuracion usada:

```text
Servidor SMTP: mail.smtp2go.com
Puerto: 2525
TLS: disponible en el mismo puerto
Autenticacion: usuario SMTP2GO
```

La contrasena SMTP no debe documentarse en texto plano.

En Postfix se configuro salida por SMTP2GO solo para los dominios verificados, mediante mapas por remitente:

```text
/etc/postfix/sender_relay_smtp2go.regexp
/etc/postfix/sasl_passwd_smtp2go.regexp
/etc/postfix/tls_policy_smtp2go.regexp
```

Parametros relevantes en Postfix:

```text
sender_dependent_relayhost_maps = regexp:/etc/postfix/sender_relay_smtp2go.regexp
smtp_sender_dependent_authentication = yes
smtp_sasl_auth_enable = yes
smtp_sasl_password_maps = regexp:/etc/postfix/sasl_passwd_smtp2go.regexp
smtp_sasl_security_options = noanonymous
smtp_sasl_tls_security_options = noanonymous
smtp_tls_policy_maps = regexp:/etc/postfix/tls_policy_smtp2go.regexp
```

Tambien se dejo envio mas conservador:

```text
smtp_destination_concurrency_limit = 1
smtp_destination_rate_delay = 5s
smtp_connect_timeout = 15s
```

Validacion:

Se enviaron pruebas desde `prueba@dominio` hacia Gmail y Hotmail para los 5 dominios verificados. Todas salieron por:

```text
relay=mail.smtp2go.com:2525
status=sent
```

La cola quedo vacia:

```text
Mail queue is empty
```

## Dominios aun no cubiertos por SMTP2GO

Estos dominios no estan cubiertos por SMTP2GO en la cuenta gratis actual:

```text
creativobusiness.com
frozenandfire.com
innovajb.com
```

Opciones:

1. Subir de plan en SMTP2GO y verificar esos dominios.
2. Usar otro relay para esos dominios.
3. Dejarlos enviando directo desde el VPS, aceptando riesgo de timeouts y menor entregabilidad.

Recomendacion: consolidar todos los dominios en un solo relay si el correo es importante.

## Siguientes pasos recomendados

### 1. Revisar recepcion real en Gmail/Hotmail

Aunque Postfix y SMTP2GO reporten `status=sent`, revisar:

```text
Inbox
Spam
Promociones
Correo no deseado
```

En:

```text
01carlos.rojas@gmail.com
charly_rojas_jimenez@hotmail.com
```

### 2. Verificar panel de SMTP2GO

En SMTP2GO revisar:

```text
Reports
Activity
Bounces
Spam complaints
Suppression list
```

### 3. Completar dominios faltantes en SMTP2GO

Agregar o mover:

```text
creativobusiness.com
frozenandfire.com
innovajb.com
```

Despues actualizar:

```text
/etc/postfix/sender_relay_smtp2go.regexp
/etc/postfix/sasl_passwd_smtp2go.regexp
```

Y recargar Postfix:

```bash
systemctl reload postfix
```

### 4. No cancelar HostGator hasta confirmar correo

Antes de cancelar HostGator, confirmar:

```text
Todos los sitios responden por HTTPS
Todos los formularios de contacto envian correo
Todos los buzones importantes reciben y envian
No hay correos pendientes en cola
Backups completos ya estan fuera de HostGator
DNS en Cloudflare apunta al VPS nuevo
```

Comando util:

```bash
mailq
```

### 5. Backups del VPS

Configurar backups periodicos de:

```text
/home
/etc
/var/lib/mysql o dumps de MariaDB/MySQL
/etc/postfix
/etc/webmin
/etc/virtualmin
```

Recomendacion minima:

```text
Backup diario local o remoto
Retencion de 7 dias
Backup semanal externo
Prueba de restauracion mensual
```

## Recomendaciones para futuras instancias WordPress

### Seguridad base

1. Mantener WordPress, plugins y themes actualizados.
2. Eliminar themes/plugins no usados.
3. Evitar plugins abandonados.
4. Instalar solo desde fuentes confiables.
5. Usar usuarios administradores unicos, no `admin`.
6. Activar 2FA si el proyecto lo justifica.
7. Limitar intentos de login.
8. Mantener backups antes de actualizaciones grandes.

### Permisos recomendados

```text
Directorios: 755
Archivos: 644
wp-config.php: 600 o 640 si es compatible
```

Evitar permisos `777`.

### Configuracion WordPress

En `wp-config.php`, revisar:

```php
define('DISALLOW_FILE_EDIT', true);
```

Tambien configurar correctamente:

```text
WP_HOME
WP_SITEURL
DB_NAME
DB_USER
DB_PASSWORD
DB_HOST
```

### Cache y rendimiento

Usar una sola capa principal de cache. Evitar instalar varios plugins que hagan lo mismo.

Opciones razonables:

```text
LiteSpeed Cache si el servidor usa LiteSpeed
WP Rocket si se quiere solucion pagada
Cache de Cloudflare para estaticos
OPcache en PHP
```

### Formularios de contacto

No depender de `mail()` directo de PHP para formularios criticos.

Recomendacion:

```text
Configurar SMTP autenticado
Usar el relay SMTP2GO o proveedor equivalente
Probar formularios despues de cada migracion
```

### Migraciones futuras WordPress

Checklist:

```text
Escanear backup
Migrar archivos
Migrar base de datos
Ajustar wp-config.php
Revisar URLs en base de datos si cambia dominio
Probar con hosts local antes de cambiar DNS
Revisar logs PHP
Probar formularios
Revisar HTTPS
Actualizar DNS
Monitorear 24-48 horas
```

## Recomendaciones para nuevos proyectos Node.js u otras tecnologias

Virtualmin puede alojar proyectos no WordPress, pero conviene estandarizar despliegues.

### Estructura sugerida

Para un proyecto Node.js:

```text
/home/usuario/app
/home/usuario/app/releases
/home/usuario/app/current
/home/usuario/app/shared
/home/usuario/logs
```

### Usuario por proyecto

Mantener un usuario de sistema por proyecto o dominio.

No correr aplicaciones como `root`.

### Variables de entorno

Guardar configuracion sensible fuera del repositorio:

```text
/home/usuario/app/shared/.env
```

Permisos:

```bash
chmod 600 /home/usuario/app/shared/.env
```

### Node.js con systemd

Ejemplo de servicio:

```ini
[Unit]
Description=App Node.js dominio.com
After=network.target

[Service]
Type=simple
User=usuario
WorkingDirectory=/home/usuario/app/current
EnvironmentFile=/home/usuario/app/shared/.env
ExecStart=/usr/bin/node server.js
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Comandos:

```bash
systemctl daemon-reload
systemctl enable app-dominio
systemctl start app-dominio
systemctl status app-dominio
```

### Reverse proxy

Usar Apache/Nginx como reverse proxy hacia el puerto local de la app.

Ejemplo conceptual:

```text
https://dominio.com -> Apache/Nginx -> http://127.0.0.1:3000
```

No exponer puertos Node.js directamente a internet salvo que sea necesario.

### Logs

Revisar:

```bash
journalctl -u app-dominio -f
```

Y configurar rotacion si se escriben logs a archivo.

### Deploys

Para proyectos nuevos:

```text
Usar Git
No editar produccion manualmente salvo emergencia
Separar staging y produccion si el proyecto lo amerita
Mantener README de despliegue
Documentar variables de entorno
```

### Bases de datos

Usar usuarios de base de datos con permisos minimos.

No reutilizar credenciales entre proyectos.

### SSL

Usar Let's Encrypt desde Virtualmin cuando sea posible.

Verificar:

```bash
curl -I https://dominio.com
```

### Correo en apps Node/PHP/Python

No enviar directo por puerto `25`.

Usar:

```text
SMTP2GO
Brevo
Mailgun
Amazon SES
Google Workspace
Microsoft 365
```

Y configurar SPF/DKIM/DMARC del proveedor.

## Comandos utiles de operacion

### Estado de Virtualmin

```bash
virtualmin list-domains
virtualmin list-users --domain dominio.com
virtualmin list-databases --domain dominio.com
```

### Estado de correo

```bash
mailq
journalctl -u postfix -f
postconf -n
```

### Probar envio local

```bash
printf "Subject: prueba\n\nmensaje\n" | sendmail -f prueba@dominio.com destino@gmail.com
```

### Revisar HTTP/HTTPS

```bash
curl -I http://dominio.com
curl -I https://dominio.com
```

### Revisar espacio

```bash
df -h
du -sh /home/*
```

### Revisar servicios

```bash
systemctl status postfix
systemctl status mariadb
systemctl status httpd
systemctl status nginx
```

## Recomendacion final

El VPS ya puede operar los sitios migrados, pero la parte critica a cerrar antes de cancelar HostGator es el correo.

Prioridad:

1. Confirmar recepcion real de pruebas en Gmail/Hotmail.
2. Completar relay SMTP para dominios faltantes.
3. Probar formularios web de cada WordPress.
4. Confirmar backups externos.
5. Mantener HostGator activo unos dias despues del cambio final por seguridad operativa.

## Bitacora

### 2026-05-16 - Inicializacion de repositorio Git

Cambio realizado:

```text
Se preparo este directorio como repositorio Git para documentacion operativa del VPS.
```

Archivos agregados:

```text
README.md
AGENTS.md
.gitignore
MIGRACION_DEVUS_VPS.md
```

Configuracion importante:

```text
Los backups .tar, .tar.gz, .tgz y variantes quedan ignorados por Git.
Tambien se ignoran archivos comunes de secretos como .env, llaves y archivos con password/secret en el nombre.
```

Validacion realizada:

```text
Repositorio inicializado en Git.
Rama principal renombrada a main.
Remote configurado como git@github.com:Charly-Rojas/vps.git.
Push exitoso a origin/main.
Backups .tar.gz verificados como ignorados por .gitignore.
```

### 2026-05-16 - Verificacion de acceso SSH al VPS

Cambio realizado:

```text
Se verifico acceso SSH IPv4 al VPS usando la llave ~/.ssh/devus_vps.
Se ajustaron permisos locales de la llave privada de 0644 a 0600 porque OpenSSH la rechazaba por permisos demasiado abiertos.
```

Servicio afectado:

```text
Acceso SSH administrativo al VPS 193.46.199.28.
```

Comandos/configuracion importante:

```bash
chmod 600 ~/.ssh/devus_vps
ssh -4 -i ~/.ssh/devus_vps root@193.46.199.28 'hostname -f; whoami; date -Is; uptime -p'
```

Validacion realizada:

```text
host=server.devus.mx
user=root
date=2026-05-16T21:06:27+00:00
uptime=up 3 days, 1 hour, 41 minutes
```

Pendientes o riesgos:

```text
Sin pendientes para acceso SSH. No se hicieron cambios en el servidor durante esta verificacion.
```

### 2026-05-16 - Usuarios administrativos charly y xime

Cambio realizado:

```text
Se crearon los usuarios locales charly y xime en el VPS y se habilito sudo sin contraseña para ambos.
```

Servicio afectado:

```text
Acceso administrativo del sistema operativo en server.devus.mx.
```

Comandos/configuracion importante:

```bash
useradd -m -s /bin/bash charly
useradd -m -s /bin/bash xime
usermod -aG wheel charly
usermod -aG wheel xime
cat > /etc/sudoers.d/devus-admins
chmod 0440 /etc/sudoers.d/devus-admins
visudo -cf /etc/sudoers.d/devus-admins
```

Contenido de `/etc/sudoers.d/devus-admins`:

```text
charly ALL=(ALL) NOPASSWD: ALL
xime ALL=(ALL) NOPASSWD: ALL
```

Validacion realizada:

```text
/etc/sudoers.d/devus-admins: parsed OK
charly pertenece a wheel y sudo -n funciona correctamente.
xime pertenece a wheel y sudo -n funciona correctamente.
```

Pendientes o riesgos:

```text
No se configuraron contraseñas ni llaves SSH especificas para estos usuarios en esta accion.
```

### 2026-05-16 - Llaves SSH para charly y xime

Cambio realizado:

```text
Se generaron pares de llaves SSH ed25519 para los usuarios charly y xime.
Se copiaron las llaves al directorio del proyecto y a ~/.ssh.
Se instalaron las llaves publicas en authorized_keys de cada usuario en el VPS.
Se amplio .gitignore para evitar que llaves con nombres devus_vps_* o id_* se agreguen a Git.
```

Servicio afectado:

```text
Acceso SSH de usuarios administrativos charly y xime en server.devus.mx.
```

Comandos/configuracion importante:

```bash
ssh-keygen -t ed25519 -a 100 -N '' -C 'charly@server.devus.mx' -f devus_vps_charly
ssh-keygen -t ed25519 -a 100 -N '' -C 'xime@server.devus.mx' -f devus_vps_xime
cp devus_vps_charly* ~/.ssh/
cp devus_vps_xime* ~/.ssh/
chmod 600 devus_vps_charly devus_vps_xime ~/.ssh/devus_vps_charly ~/.ssh/devus_vps_xime
chmod 644 devus_vps_charly.pub devus_vps_xime.pub ~/.ssh/devus_vps_charly.pub ~/.ssh/devus_vps_xime.pub
```

Archivos locales:

```text
devus_vps_charly
devus_vps_charly.pub
devus_vps_xime
devus_vps_xime.pub
~/.ssh/devus_vps_charly
~/.ssh/devus_vps_charly.pub
~/.ssh/devus_vps_xime
~/.ssh/devus_vps_xime.pub
```

Validacion realizada:

```text
Login SSH directo con devus_vps_charly: user=charly host=server.devus.mx sudo=ok
Login SSH directo con devus_vps_xime: user=xime host=server.devus.mx sudo=ok
git status muestra las llaves locales como ignoradas.
```

Pendientes o riesgos:

```text
Las llaves privadas fueron creadas sin passphrase por solicitud operativa implicita. Deben mantenerse fuera de Git y compartirse solo por un canal seguro si otra persona las necesita.
```

### 2026-05-16 - Certificados Let's Encrypt en origen para dominios Cloudflare

Cambio realizado:

```text
Se reemplazaron certificados self-signed del origen por certificados Let's Encrypt en Virtualmin para los dominios que estaban detras de Cloudflare.
Se solicitaron certificados solo para dominio raiz, www y mail, evitando admin/webmail para no repetir fallos previos por hostnames no resueltos o no validados.
Virtualmin copio los certificados a la configuracion web y los aplico tambien a Webmin/Usermin, Dovecot y Postfix para cada dominio.
```

Dominios afectados:

```text
creativobusiness.com
dyr.com.mx
fortiguardia.com
frozenandfire.com
sjpabogados.com
quieroteamup.com
```

Comandos/configuracion importante:

```bash
virtualmin generate-acme-cert \
  --domain dominio.com \
  --host dominio.com \
  --host www.dominio.com \
  --host mail.dominio.com \
  --web \
  --renew \
  --email-error \
  --allow-subset \
  --skip-dns-check
```

Validacion realizada:

```text
Se valido contra la IP del origen 193.46.199.28 usando SNI que cada dominio sirve certificado emitido por Let's Encrypt.
Se valido SMTP STARTTLS en mail.dominio:587 para cada dominio afectado y todos presentan certificados Let's Encrypt.
Virtualmin muestra SSL provider renewal habilitado para los dominios.
```

Vencimientos:

```text
creativobusiness.com: 2026-08-14
dyr.com.mx: 2026-08-14
fortiguardia.com: 2026-08-14
frozenandfire.com: 2026-08-14
sjpabogados.com: 2026-08-14
quieroteamup.com: 2026-08-14
```

Pendientes o riesgos:

```text
No se incluyeron admin.dominio ni webmail.dominio en estos certificados. Si se desea usarlos publicamente con HTTPS valido, primero confirmar DNS y alcance real de esos hostnames.
La renovacion automatica queda gestionada por Virtualmin; no se configuro un timer separado de certbot para evitar renovar sin copiar certificados a servicios.
```

### 2026-05-17 - Publicacion Lex Garantia en VPS

Cambio realizado:

```text
Se creo y publico la app Next.js de Lex Garantia en el VPS con dos entornos separados:
produccion en lexgarantia.com desde rama main y dev en dev-env.lexgarantia.com desde rama dev.
Se configuraron Virtualmin, Apache reverse proxy, SSL, systemd, variables de entorno separadas y buzon contacto@lexgarantia.com.
```

Dominio o servicio afectado:

```text
lexgarantia.com
dev-env.lexgarantia.com
Apache/Virtualmin
systemd
Postfix/Dovecot/OpenDKIM
```

Comandos/configuracion importante:

```bash
virtualmin create-domain --domain lexgarantia.com --user lexgarantia --web --ssl --mail --unix --dir --logrotate --spam --proxy http://127.0.0.1:3100/
virtualmin create-domain --domain dev-env.lexgarantia.com --user lexgdev --web --ssl --unix --dir --logrotate --proxy http://127.0.0.1:3101/
dnf -y install nodejs git
virtualmin create-user --domain lexgarantia.com --user contacto --passfile <archivo-temporal> --real "Contacto Lex Garantia" --no-creation-mail
systemctl enable --now lex-garantia-prod.service lex-garantia-dev.service
```

Rutas y servicios:

```text
Produccion:
  usuario: lexgarantia
  app: /home/lexgarantia/apps/lex_garantia_prod/current
  env: /home/lexgarantia/apps/lex_garantia_prod/shared/.env
  servicio: lex-garantia-prod.service
  puerto: 127.0.0.1:3100

Dev:
  usuario: lexgdev
  app: /home/lexgdev/apps/lex_garantia_dev/current
  env: /home/lexgdev/apps/lex_garantia_dev/shared/.env
  servicio: lex-garantia-dev.service
  puerto: 127.0.0.1:3101

Buzon:
  contacto@lexgarantia.com
  password guardado solo en /root/migrations/lexgarantia-contact-mailbox-20260517-024520.txt
```

Validacion realizada:

```text
GitHub:
  ramas publicadas: main, dev, feature/project-initialization, feature/bootstrap-docs

Build:
  npm ci y npm run build exitosos en prod y dev dentro del VPS.

Servicios:
  lex-garantia-prod.service activo
  lex-garantia-dev.service activo
  127.0.0.1:3100 responde 200
  127.0.0.1:3101 responde 200

Web:
  https://lexgarantia.com responde 200 por Cloudflare
  https://www.lexgarantia.com responde 200 por Cloudflare
  https://dev-env.lexgarantia.com responde 200 por Cloudflare
  robots.txt de produccion permite indexacion
  robots.txt de dev contiene Disallow: /
  sitemap.xml generado en ambos entornos

Correo:
  contacto@lexgarantia.com creado.
  App autentica SMTP contra Postfix local con SMTP_HOST=127.0.0.1 y SMTP_TLS_SERVERNAME=server.devus.mx.
  Prueba local al buzon contacto entregada.
  Prueba del formulario en produccion exitosa.
  Confirmacion externa a Gmail aceptada por Gmail.
  Cola Postfix vacia.
```

Pendientes o riesgos:

```text
Cloudflare requiere ajustes manuales para correo:
  mail debe ser A 193.46.199.28 en DNS only.
  MX debe apuntar a mail.lexgarantia.com, no al apex proxied.
  Agregar/ajustar SPF, DKIM y DMARC.

SMTP2GO rechazo lexgarantia.com porque el dominio no esta verificado en su panel.
Se retiro el mapeo de relay para lexgarantia.com; actualmente la salida usa Postfix directo.
Para produccion estable, verificar lexgarantia.com en SMTP2GO o configurar otro proveedor SMTP transaccional autenticado.
```

### 2026-05-17 - Ajuste visual y consentimiento de contacto en Lex Garantia

Cambio realizado:

```text
Se publico un ajuste de UI del sitio Lex Garantia en produccion y dev.
Los botones primarios ahora mantienen texto blanco.
El boton flotante de WhatsApp quedo como boton circular verde solo con icono blanco.
El formulario de contacto ahora exige aceptar el aviso de privacidad antes de enviar.
Se actualizo el texto introductorio del formulario de contacto.
```

Dominio o servicio afectado:

```text
lexgarantia.com
dev-env.lexgarantia.com
lex-garantia-prod.service
lex-garantia-dev.service
```

Comandos/configuracion importante:

```bash
git fetch origin main
git checkout -B main origin/main
npm ci --no-audit --no-fund
npm run build
systemctl restart lex-garantia-prod.service

git fetch origin dev
git checkout -B dev origin/dev
npm ci --no-audit --no-fund
npm run build
systemctl restart lex-garantia-dev.service
```

Validacion realizada:

```text
Commit desplegado en prod y dev: bbe1347.
npm run lint exitoso en local.
npm run build exitoso en local.
npm run build exitoso en VPS para prod y dev.
Servicios systemd activos.
https://lexgarantia.com responde 200.
https://www.lexgarantia.com responde 200.
https://dev-env.lexgarantia.com responde 200.
Playwright en produccion confirmo texto blanco en botones primarios.
Playwright en produccion confirmo boton WhatsApp sin texto visible, fondo verde y radio circular.
Playwright en produccion confirmo texto nuevo y checkbox obligatorio del aviso de privacidad.
```

Pendientes o riesgos:

```text
Persisten pendientes previos de correo: ajustar DNS en Cloudflare y verificar lexgarantia.com en SMTP2GO o proveedor transaccional definitivo.
```

### 2026-05-20 - Roundcube como webmail por dominio en mail.<dominio>

Cambio realizado:

```text
Se instalo Roundcube 1.6.15 desde EPEL en el VPS como webmail compartido para los dominios alojados.
Se creo base de datos MariaDB roundcube con usuario dedicado y se cargo el schema inicial.
Se configuro /etc/roundcubemail/config.inc.php apuntando a IMAP/SMTP localhost via TLS sobre puertos 143/587.
Se creo pool dedicado PHP-FPM roundcube en /run/php-fpm/roundcube.sock.
Se generaron vhosts Apache mail.<dominio> en /etc/httpd/conf.d/00-roundcube-mail-vhosts.conf usando los certs Let's Encrypt existentes de cada dominio (todos cubren mail.<dominio> en SAN).
Cada vhost http redirige a https; el handler PHP usa el socket Roundcube.
```

Dominio o servicio afectado:

```text
mail.creativobusiness.com
mail.devus.mx
mail.dyr.com.mx
mail.fortiguardia.com
mail.frozenandfire.com
mail.innovajb.com
mail.lexgarantia.com
mail.sjpabogados.com
mail.quieroteamup.com
Apache (httpd)
PHP-FPM
MariaDB (base nueva roundcube)
```

Comandos/configuracion importante:

```bash
dnf -y install roundcubemail
mysql -uroot -e "CREATE DATABASE roundcube CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"
mysql -uroot -e "CREATE USER 'roundcube'@'localhost' IDENTIFIED BY '<pass>'; GRANT ALL ON roundcube.* TO 'roundcube'@'localhost';"
mysql -uroot roundcube < /usr/share/roundcubemail/SQL/mysql.initial.sql
systemctl restart php-fpm
systemctl reload httpd
```

Archivos de referencia en VPS:

```text
/root/migrations/setup-roundcube.sh
/root/migrations/roundcube-credentials-20260520.txt   (modo 600, root only)
/etc/roundcubemail/config.inc.php                     (modo 640, root:apache)
/etc/php-fpm.d/roundcube.conf
/etc/httpd/conf.d/00-roundcube-mail-vhosts.conf
/var/lib/roundcubemail/    /var/log/roundcubemail/
```

Validacion realizada:

```text
httpd -t: Syntax OK
Los 9 vhosts mail.<dominio> responden HTTPS 200 con titulo "DevUs Webmail :: Welcome to DevUs Webmail".
Login real exitoso para prueba@quieroteamup.com: POST a /?_task=login devolvio 302 hacia ?_task=mail con sesion nueva.
Cert Let's Encrypt de cada dominio cubre mail.<dominio> en SAN, sin necesidad de emitir certs adicionales.
PHP-FPM pool roundcube activo en /run/php-fpm/roundcube.sock como apache:apache.
config.inc.php tiene enable_installer=false y desactiva verify_peer en TLS local a Dovecot/Postfix (cert self-signed interno).
```

Acceso para usuarios:

```text
URL por dominio: https://mail.<dominio>
Usuario: direccion de correo completa
Pass: la del buzon
```

Pendientes o riesgos:

```text
mail.lexgarantia.com responde via VPS solo si se pone DNS only en Cloudflare; hoy el A esta proxied y resuelve a IPs de Cloudflare.
Ademas el MX de lexgarantia.com apunta a Cloudflare Email Routing (_dc-mx.a70f41ee9d25.lexgarantia.com), asi que el correo entrante no llega a Dovecot del VPS; los buzones IMAP del VPS no recibiran nada externo hasta que el MX cambie a mail.lexgarantia.com.
Existen ServerAlias mail.<dominio> en los vhosts principales de Virtualmin; los vhosts dedicados de Roundcube ganan match por ServerName exacto y se cargan desde /etc/httpd/conf.d/00-... (despues de httpd.conf). Si Virtualmin regenera y reordena vhosts, revisar que sigan ganando.
Roundcube se actualizara con dnf update; backup recomendado de /etc/roundcubemail/ y dump de la base roundcube antes de cada actualizacion mayor.
No se instalo plugin password ni se configuro cambio de contrasena desde Roundcube; los usuarios deben pedir cambio por admin.
```

### 2026-05-20 - Redirect puerto 2096 (cPanel webmail) hacia mail.<dominio>

Cambio realizado:

```text
Se hizo que Apache escuche en :2096 con SSL y devuelva 301 hacia https://mail.<dominio>/ para los 9 dominios.
Esto cubre bookmarks antiguos tipo https://dominio.com:2096 que en cPanel apuntaban al webmail.
Se abrio 2096/tcp en firewalld. Cloudflare ya proxia ese puerto, asi que el 523 "origin unreachable" desaparece al haber un origen escuchando.
```

Dominio o servicio afectado:

```text
Apache (httpd) en puerto 2096
firewalld
mail.<dominio> de los 9 dominios cubiertos por Roundcube
```

Comandos/configuracion importante:

```bash
# Generado por /root/migrations/setup-2096-redirect.sh
# /etc/httpd/conf.d/01-cpanel-2096-redirect.conf:
#   Listen 2096 https
#   <VirtualHost *:2096> por dominio con su cert LE y RedirectMatch 301
firewall-cmd --permanent --add-port=2096/tcp
firewall-cmd --reload
systemctl reload httpd
```

Validacion realizada:

```text
httpd -t: Syntax OK
Curl directo al origen en :2096 devuelve 301 hacia https://mail.<dominio>/ en los 9 dominios.
Curl via Cloudflare a https://<dominio>:2096/ devuelve 301 hacia https://mail.<dominio>/ en los 9 dominios; el error 523 quedo resuelto.
```

Pendientes o riesgos:

```text
Solo se cubrio 2096 (HTTPS webmail). Otros puertos cPanel (2082 cPanel, 2083 cPanel SSL, 2086/2087 WHM, 2095 webmail HTTP) NO redirigen; si surgen bookmarks que los usen, replicar el patron.
El cert Listen 2096 https usa los certs LE de cada dominio; renuevan via Virtualmin igual que el resto, no requieren paso extra.
```
