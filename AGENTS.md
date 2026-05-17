# Instrucciones del Proyecto

Este repositorio documenta y opera el VPS DevUs.

Antes de responder o ejecutar cambios relacionados con este proyecto:

1. Leer `MIGRACION_DEVUS_VPS.md`.
2. Usar ese archivo como bitacora principal.
3. Actualizarlo solo despues de cambios importantes o funcionales con fecha, servicio afectado, comandos/configuracion, validacion y pendientes. No registrar verificaciones rutinarias, lecturas de archivos, consultas sin cambios o acciones menores que no alteren la operacion.
4. No commitear backups `.tar`, credenciales, llaves privadas ni secretos.

Para acceso al VPS, preferir IPv4:

```bash
ssh -4 -i ~/.ssh/devus_vps root@193.46.199.28
```

## Proyecto independiente lex_garantia

La carpeta `lex_garantia/` esta ignorada por este repositorio porque sera un proyecto independiente, con su propio `AGENTS.md`, `.git`, dependencias y ciclo de despliegue.

Cuando una nueva instancia trabaje dentro de `lex_garantia/`, debe leer primero `lex_garantia/AGENTS.md`. Ese proyecto sera una app Next.js y debera generarse/desplegarse en el VPS sin mezclar su historial Git, dependencias, artefactos de build ni secretos con este repositorio operativo del VPS.
