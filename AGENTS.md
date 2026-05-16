# Instrucciones del Proyecto

Este repositorio documenta y opera el VPS DevUs.

Antes de responder o ejecutar cambios relacionados con este proyecto:

1. Leer `MIGRACION_DEVUS_VPS.md`.
2. Usar ese archivo como bitacora principal.
3. Actualizarlo despues de cambios relevantes con fecha, servicio afectado, comandos/configuracion, validacion y pendientes.
4. No commitear backups `.tar`, credenciales, llaves privadas ni secretos.

Para acceso al VPS, preferir IPv4:

```bash
ssh -4 -i ~/.ssh/devus_vps root@193.46.199.28
```

