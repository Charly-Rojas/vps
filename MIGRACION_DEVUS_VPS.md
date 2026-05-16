# Migracion DevUs a VPS Hostinger con Virtualmin

Fecha de documentacion: 2026-05-16

## Regla operativa para futuras sesiones

Antes de hacer cambios relacionados con este VPS, dominios, DNS, correo, Virtualmin, WordPress, Node.js u otras apps alojadas aqui, leer este documento.

Despues de cualquier cambio relevante, actualizar este mismo documento con:

```text
Fecha
Cambio realizado
Dominio o servicio afectado
Comandos/configuracion importante
Validacion realizada
Pendientes o riesgos
```

Este archivo debe tratarse como la bitacora principal de operacion del VPS.

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
