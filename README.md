# 🟢 Greenbone Toolbox - Automatización GVM

Herramienta de mantenimiento automatizado para entornos Greenbone / GVM basada en PostgreSQL + feeds SCAP/CERT/NVT.

---

# 📦 Instalación

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
# Instalación

vim /usr/bin/greenbone-toolbox

--------------------------------------------------
...
código de greenbone-toolbox
...
--------------------------------------------------

chmod +x /usr/bin/greenbone-toolbox

Ahora puedo ejecutar el script directamente: greenbone-toolbox

# crontab -e

--------------------------------------------------
...
00 6 * * * /usr/sbin/ntpdate -s hora.roa.es

02 00 1 * * * /usr/bin/greenbone-toolbox 4 >> /var/log/gvm-feed.log 2>&1

00 21 * * * /usr/bin/greenbone-toolbox 3 >> /var/log/greenbone-system.log 2>&1

#04 14 * * * /usr/bin/greenbone-toolbox 3 >> /var/log/greenbone-system.log 2>&1

00 23 * * 5 /usr/bin/greenbone-toolbox 1 >> /var/log/greenbone-system.log 2>&1
--------------------------------------------------

# Lógica

En Greenbone/GVM el sistema funciona en dos capas:

- Feeds (disco)
  - /scap-data → CVEs, OVAL, CPEs
  - /cert-data → CERT advisories
  - /plugins → NVTs (NASL scripts)

- Base de datos (gvmd PostgreSQL)
  - hosts, tasks, targets
  - reports, results
  - vulnerabilidades indexadas

## Flujo del sistema

Feeds (SCAP / CERT / NVT)
        ↓
greenbone-feed-sync
        ↓
gvmd --rebuild
        ↓
PostgreSQL (gvmd DB)
        ↓
Escaneos → Results → Reports → Vulnerabilidades

## Lógica del toolbox

- Modo 1: mantenimiento completo
- Modo 2: sync feeds + rebuild DB
- Modo 3: VACUUM FULL PostgreSQL
- Modo 4: reinicio de servicios
- Modo 5: limpieza física de feeds

## Objetivo

ACTUALIZAR → LIMPIAR → OPTIMIZAR → RECONSTRUIR → REINICIAR
