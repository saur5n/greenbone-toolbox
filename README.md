# 🟢 Greenbone Toolbox - Automatización GVM

Herramienta de mantenimiento automatizado para entornos Greenbone / GVM basada en PostgreSQL + feeds SCAP/CERT/NVT.

---

### 📦 Instalación

```bash
vim /usr/bin/greenbone-toolbox
--------------------------------------------------
...
código de greenbone-toolbox
...
--------------------------------------------------

chmod +x /usr/bin/greenbone-toolbox

Ahora puedo ejecutar el script directamente: greenbone-toolbox
````


### 📦 crontab -e (automatización)
````
# Optimización de base de datos PostgreSQL (VACUUM FULL)
# Frecuencia: diaria
00 21 * * * /usr/bin/greenbone-toolbox 3 >> /var/log/greenbone-system.log 2>&1
# Mantenimiento completo: limpieza feeds + reports antiguos + optimización + reinicio servicios (>15 días)
# Frecuencia: semanal (viernes)
00 23 * * 5 /usr/bin/greenbone-toolbox 1 >> /var/log/greenbone-system.log 2>&1
# Reinicia servicios GVM + sincronizadores (gvmd, ospd-openvas, gsad, openvasd)
# Frecuencia: mensual (día 1 de cada mes)
02 00 1 * * * /usr/bin/greenbone-toolbox 4 >> /var/log/gvm-feed.log 2>&1
````



## Objetivo

ACTUALIZAR → LIMPIAR → OPTIMIZAR → RECONSTRUIR → REINICIAR
