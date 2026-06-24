#!/bin/bash
# script para automatizar mantenimiento aplicación de Greenbone
YELLOW='\033[1;33m'
NC='\033[0m'
RED='\033[1;31m'

DB_NAME="gvmd"
DB_USER="postgres"
DAYS="15"

# Permitir ejecutar opción directamente por parámetro (para cron)
if [ -n "$1" ]; then
    opcion="$1"
 else
     clear
     cat <<'EOF'

  __________                           ___.                          ___________           .__ ___.
 /    _____/______   ____   ____   ____\_ |__   ____   ____   ____   \__    ___/___   ____ |  |\_ |__   _______  ___
/     \  __\_  __ \_/ __ \_/ __ \ /    \| __ \ /  _ \ /    \_/ __ \    |    | /  _ \ /  _ \|  | | __ \ /  _ \  \/  /
\      \_\  \  | \/\  ___/\  ___/|   |  \ \_\ (  <_> )   |  \  ___/    |    |(  <_> |  <_> )  |_| \_\ (  <_> >    <
 \________  /__|    \___  >\___  >___|  /___  /\____/|___|  /\___  >   |____| \____/ \____/|____/___  /\____/__/\_ \
          \/            \/     \/     \/    \/            \/     \/                                 \/            \/
                                            Script de mantenimiento para Greenbone

EOF
    #descripción breve
          # echo "En GVM, los feeds se descargan en disco como fuente temporal (scap-data/, cert-data/ y plugins/) y luego se importan a PostgreSQL, donde se indexan y relacionan con escaneos, CVEs, vulnerabilidades, tareas y reportes."

    #descripción extensa
    echo "En Greenbone/GVM hay dos sitios donde se descarga la información, los registros de la db apuntan a los feeds descargados:"
    echo "  - Feeds descargados (ficheros en disco): fuente temporal."
    echo "          + scap-data/ → CVEs, CPEs, OVAL, definiciones SCAP"
    echo "          + cert-data/ → Alertas y bulletins CERT."
    echo "          + plugins/   → NVTs (scripts NASL), nasl-data. (en versiones antiguas se referencia como nvt/)"
    echo "  - Base de datos PostgreSQL (gvmd DB): cada registro actua como “punteros” y metadatos que permiten buscar y relacionar vulnerabilidades con tareas/escaneos de los archivos físicos."
    echo "          + Se indexan, relacionandose como los resultados de tus escaneos CVEs ↔ Hosts ↔ Vulnerabilidades encontradas."
    echo "          + También ahí se guardan tasks, targets, resultados de escaneo, reportes, usuarios, permisos, etc."
    echo ""

   #descripción extensa
    echo -e "${YELLOW}===================================================================================================================="
    echo " Menu de mantenimiento GVM"
    echo "===================================================================================================================="
    echo "1) DELETE Modo mantenimiento automático (DELETE de feeds físicos, reports y DB > ${DAYS} días)"
    echo "2) Borrar archivos físicos de los feeds (/scap-data/*, /cert-data/* y /openvas/plugins/*)"
    echo "3) Limpiar registros vacíos y optimizar DB (VACUUM FULL)"
    echo "4) Reiniciar todos los servicios y sincronizadores de Greenbone (systemctls y greenbone-*)"
        echo "5) Borrar feeds físicos y reconstruye la cache de feeds en la base de datos"
    echo "6) TRUNCATE * reports (alerts, results, reports, report_hosts, report_host_details, report_format_*)"
    echo "7) TRUNCATE * hosts (hosts, host_details, host_oss, host_identifiers)"
    echo "8) TRUNCATE * tasks (tasks, task_preferences, task_files, task_alerts)"
    echo -e "9) TRUNCATE * targets (targets, targets_login_data)"${NC}
    echo -e "${RED}10) RESET reinicio de la base de datos (borro la db para volverla a generar vacía)" ${NC}
    read -p "Selecciona una opción (1-9): " opcion
fi

case $opcion in
  1)
    echo "Modo mantenimiento automático (elementos con más de ${DAYS} días)..."

    # 1 Borrar partes físicas de los feeds con más de ${DAYS} días
    echo "Borrando partes físicas de los feeds con más de ${DAYS} días..."
    systemctl stop gvmd ospd-openvas
    find /var/lib/gvm/scap-data/* /var/lib/gvm/cert-data/* /var/lib/openvas/plugins/* -mtime +${DAYS} -exec rm -rf {} \;
    systemctl start gvmd ospd-openvas
    echo "Archivos físicos de los feeds con más de ${DAYS} días eliminados (/var/lib/gvm/scap-data/*, /var/lib/gvm/cert-data/* y /var/lib/openvas/plugins/*)."

    # 2 Borrar reports con más de ${DAYS} días
    echo "Borrando reports con más de ${DAYS} días..."
    sudo -u $DB_USER psql -d $DB_NAME -c "
-- 1 Borrar resultados de reports antiguos (trash y activos)
DELETE FROM results_trash WHERE report IN (SELECT id FROM reports WHERE creation_time < EXTRACT(EPOCH FROM (NOW() - INTERVAL '${DAYS} days')));

-- 2 Borrar detalles de hosts asociados a reports antiguos
DELETE FROM report_host_details
WHERE report_host IN (
    SELECT id FROM report_hosts
    WHERE report IN (
        SELECT id FROM reports WHERE creation_time < EXTRACT(EPOCH FROM (NOW() - INTERVAL '${DAYS} days'))
    )
);

-- 3 Borrar report_hosts de reports antiguos
DELETE FROM report_hosts
WHERE report IN (
    SELECT id FROM reports WHERE creation_time < EXTRACT(EPOCH FROM (NOW() - INTERVAL '${DAYS} days'))
);

-- 4 Borrar resultados de reports antiguos (trash y activos)
DELETE FROM results_trash
WHERE report IN (
    SELECT id FROM reports WHERE creation_time < EXTRACT(EPOCH FROM (NOW() - INTERVAL '${DAYS} days'))
);

DELETE FROM results
WHERE report IN (
    SELECT id FROM reports WHERE creation_time < EXTRACT(EPOCH FROM (NOW() - INTERVAL '${DAYS} days'))
);

-- 5 Borrar alerts de reports antiguos
DELETE FROM alert_condition_data
WHERE alert IN (SELECT id FROM alerts WHERE creation_time < EXTRACT(EPOCH FROM (NOW() - INTERVAL '${DAYS} days')));

DELETE FROM alert_event_data
WHERE alert IN (SELECT id FROM alerts WHERE creation_time < EXTRACT(EPOCH FROM (NOW() - INTERVAL '${DAYS} days')));

DELETE FROM alert_method_data
WHERE alert IN (SELECT id FROM alerts WHERE creation_time < EXTRACT(EPOCH FROM (NOW() - INTERVAL '${DAYS} days')));

DELETE FROM alerts
WHERE creation_time < EXTRACT(EPOCH FROM (NOW() - INTERVAL '${DAYS} days'));

-- 6 Finalmente, borrar los reports
DELETE FROM reports
WHERE creation_time < EXTRACT(EPOCH FROM (NOW() - INTERVAL '${DAYS} days'));
"
    echo "Registros borrados de más de ${DAYS} días (reports, report_hosts, report_host_details, results, alerts)."

    # 3 Limpiar registros vacíos y optimizar DB
    echo "Limpiando registros vacíos y optimizando DB... (VACUUM FULL)"
    sudo -u postgres psql -d "$DB_NAME" -U "$DB_USER" -c 'VACUUM FULL;'
    echo "VACUUM FULL realizado."

    # 4 Reinicio servicios
    systemctl restart postgresql
    systemctl restart gvmd
    systemctl restart gsad
    systemctl restart ospd-openvas
    systemctl restart openvasd
    ;;

  2)
    # 1 Borro archivos físicos
    echo "Borrando archivos físicos de los feeds..."
    systemctl stop gvmd ospd-openvas
    # [x]: /var/lib/gvm/scap-data/ - CVEs, CPEs, OVAL. 55
    # [x]: /var/lib/gvm/cert-data/ - CERT-Bund, DFN-CERT... 56
    # [x]: /var/lib/gvm/plugins/ - NVTs (scripts NASL), nasl-data
    rm -rf /var/lib/gvm/scap-data/* /var/lib/gvm/cert-data/* /var/lib/openvas/plugins/*
    systemctl start gvmd ospd-openvas
    echo "Archivos eliminados (/var/lib/gvm/scap-data/*, /var/lib/gvm/cert-data/* y /var/lib/openvas/plugins/*)."

    # 2 Reinicio servicios
    systemctl restart postgresql
    systemctl restart gvmd
    systemctl restart gsad
    systemctl restart ospd-openvas
    systemctl restart openvasd
    ;;

  3)
    echo "Limpiando registros vacíos y optimizando DB..."
    #sudo -u postgres psql -d "$DB_NAME" -U "$DB_USER" -c 'VACUUM FULL;'
    sudo -u postgres psql -d "$DB_NAME" -c "VACUUM FULL;"
    echo "Hecho."
    ;;

  4)
    # descargo los feeds en archivos físicos
    #greenbone-nvt-sync
    #greenbone-scapdata-sync
    #greenbone-certdata-sync
    #greenbone-feed-sync
    sudo runuser -u gvm -- greenbone-feed-sync --type all
    # enlazo los feeds de los archivos a la db
    sudo runuser -u gvm -- gvmd --rebuild
    echo "Sincronizadores reiniciados (greenbone-feed-sync, greenbone-nvt-sync, greenbone-scapdata-sync, greenbone-certdata-sync) ."

    systemctl restart postgresql
    systemctl restart gvmd
    systemctl restart gsad
    systemctl restart ospd-openvas
    systemctl restart openvasd
    echo "Servicios reiniciados (postgresql, gvmd, gsad, ospd-openvas y openvasd)."
    ;;

  5)
        # paro servicios
    echo -e "${YELLOW}paro servicios" ${NC}
        sudo systemctl stop gvmd
        sudo systemctl stop ospd-openvas

        # borro feeds físicos
        echo -e "${YELLOW}elimino feeds físicos" ${NC}
        sudo rm -rf /var/lib/gvm/nvt/*
        sudo rm -rf /var/lib/gvm/scap-data/*
        sudo rm -rf /var/lib/gvm/cert-data/*

        # Eliminar lock huérfano
        echo -e "${YELLOW}elimino lock huérfano" ${NC}
        sudo rm -f /var/lib/gvm/feed-update.lock

        # levanto servicios
        echo -e "${YELLOW}levanto servicios" ${NC}
        sudo systemctl start gvmd
        sudo systemctl start ospd-openvas

        # Reconstruir cache y vaciar la DB de feeds
        echo -e "${YELLOW}Reconstruyo cache y vacio la base de datos de feeds" ${NC}
    sudo runuser -u gvm -- greenbone-feed-sync --type all
    sudo runuser -u gvm -- gvmd --rebuild --verbose
    ;;

  6)
    echo "Borrando reports..."
    sudo -u $DB_USER psql -d $DB_NAME -c "
    TRUNCATE alerts, results, reports, report_hosts, report_host_details, report_format_params, report_format_param_options, report_formats RESTART IDENTITY CASCADE;
    "
    echo "Hecho."
    ;;

  7)
    echo "Borrando hosts..."
    sudo -u $DB_USER psql -d $DB_NAME -c "
    TRUNCATE hosts, host_details, host_oss, host_identifiers RESTART IDENTITY CASCADE;
    "
    echo "Hecho."
    ;;

  8)
    echo "Borrando tasks..."
    sudo -u $DB_USER psql -d $DB_NAME -c "
    TRUNCATE tasks, task_preferences, task_files, task_alerts RESTART IDENTITY CASCADE;
    "
    echo "Hecho."
    ;;

  9)
    echo "Borrando targets..."
    sudo -u $DB_USER psql -d $DB_NAME -c "
    TRUNCATE targets, targets_login_data RESTART IDENTITY CASCADE;
    "
    echo "Hecho."
    ;;

  10)
    echo "Borrando base de datos completa de greenbone..."
    echo -e "${RED}🚨 Al borrar la base de datos se borrarán todos los datos (incluidos los feeds) y se iniciará con una instalación limpia."
    echo -e "${RED}🚨 Se perderán usuarios, configuraciones y resultados previos." ${NC}
    read -p "¿Estás seguro de que quieres eliminar la base de datos? Esto no se puede deshacer (s/n): " confirm
    if [[ "$confirm" =~ ^[Ss]$ ]]; then
        echo -e "${YELLOW}Paro el servicio de gvmd..."${NC}
        systemctl stop gvmd

        echo -e "${YELLOW}Eliminando archivo físicos de los feeds descargados..."${NC}
        sudo rm -rf /var/lib/openvas/plugins/*
        sudo rm -rf /var/lib/gvm/scap-data/*
        sudo rm -rf /var/lib/gvm/cert-data/*

        echo -e "${YELLOW}Eliminado cache del proyecto..."${NC}
        sudo rm -rf /var/lib/openvas/plugins/*
        sudo rm -rf /var/cache/openvas/*

        echo -e "${YELLOW}Eliminando la base de datos..."${NC}
        sudo -u postgres dropdb $DB_NAME

        echo -e "${YELLOW}No creo el usuario para la base de datos porque utilizo el mismo creado anteriormente ($DB_NAME):"${NC}
        echo -e "${YELLOW}\e[9m sudo -u postgres psql -c \"CREATE ROLE gvm WITH LOGIN PASSWORD 'tu_contraseña';\" \e[0m"${NC}
        echo -e "${YELLOW}Creo la base de datos nueva..."${NC}
        sudo -u postgres createdb -O gvm $DB_NAME
        #sudo -u postgres createdb $DB_NAME
        #sudo -u postgres psql -c "ALTER DATABASE gvmd OWNER TO gvm;"
        #sudo -u postgres psql -d gvmd -c 'create role dba with superuser noinherit;'
        #sudo -u postgres psql -d gvmd -c 'grant dba to gvmd;'

        echo -e "${YELLOW}Inicializo la estructura de la base de datos (tablas, columnas etc)..."${NC}
        sudo runuser -u gvm -- $DB_NAME

        echo -e "${YELLOW}Descargo feeds como archivos físicos..."${NC}
        sudo runuser -u gvm -- greenbone-feed-sync --type all

        echo -e "${YELLOW}Construyo la base de datos a través de los feeds descargados..."${NC}
        sudo runuser -u gvm -- $DB_NAME --rebuild

        echo -e "${YELLOW}Creo el usuario admin para la interface web admin:Rengereg%768 (recuerda cambiar la contraseña)..."${NC}
        gvmd --create-user=admin --password=Rengereg%768
        echo -e "${YELLOW}\e[9m gvmd --user=admin --new-password=otracontraseña\" \e[0m"${NC}

        echo -e "${YELLOW}Modificaciones actualizadas..."${NC}
        echo -e "${YELLOW}Inicializo todos los servicios de nuevo."${NC}
        systemctl start gvmd
        systemctl restart postgresql gvmd gsad ospd-openvas openvasd

        echo -e "${RED}Entra en la interface web a través del protocolo 9392 y dirigete a 'Administration > Feed Status', desde donde verás la actualización de los feeds."${NC}

    else
        echo "Operación cancelada."
    fi
    ;;

  *)
    echo "Opción inválida"
    ;;
esac
