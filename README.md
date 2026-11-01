# Test
Desarrollo_Propios

codigo 

#!/usr/bin/env bash

set -Eeuo pipefail

# =====================================================
# 1. CONFIGURACION SQL SERVER
# =====================================================
SQL_SERVER="${SQL_SERVER:-MI_SERVIDOR_SQL}"
SQL_USER="${SQL_USER:-MI_USUARIO}"
SQL_PASS="${SQL_PASS:?Debes definir SQL_PASS}"
SQL_DB="${SQL_DB:-pcbs}"

# IMPORTANTE:
# Completa el JOIN y las condiciones con la consulta SQL
# exacta que utilizas actualmente.
SQL_QUERY="${SQL_QUERY:-SET NOCOUNT ON; SELECT ISNULL(COUNT(*), 0) AS TOTAL FROM pcbs.dbo.vnp_cotizacion A}"

# =====================================================
# 2. CONFIGURACION DYNATRACE
# =====================================================

# Endpoint local de OneAgent (no requiere token)
DT_METRIC_URL="http://localhost:14499/metrics/ingest"

# URL de tu entorno Dynatrace SaaS
DT_EVENTS_URL="${DT_EVENTS_URL:?Define DT_EVENTS_URL}"

# Token con permiso events.ingest
DT_API_TOKEN="${DT_API_TOKEN:?Define DT_API_TOKEN}"

# Entidad existente donde se asociara el problema.
# Usa el nombre REAL de un host monitorizado.
DT_ENTITY_HOST="${DT_ENTITY_HOST:?Define DT_ENTITY_HOST}"

# Nombre de la metrica
METRIC_KEY="sqlserver_PCBS.SEGUROS_AUTOS_MERCOSUR.TOTAL_2"

# =====================================================
# 3. EJECUTAR CONSULTA SQL SERVER
# =====================================================

echo "Ejecutando consulta SQL Server..."

if ! SQL_OUTPUT=$(
    sqlcmd \
        -S "$SQL_SERVER" \
        -U "$SQL_USER" \
        -P "$SQL_PASS" \
        -d "$SQL_DB" \
        -h -1 \
        -W \
        -b \
        -Q "$SQL_QUERY" 2>&1
); then
    echo "ERROR: Fallo la consulta SQL Server."
    echo "$SQL_OUTPUT" >&2
    exit 1
fi

# Obtener el ultimo resultado numerico de la consulta
TOTAL=$(
    printf '%s\n' "$SQL_OUTPUT" |
    awk '
        /^[[:space:]]*[0-9]+[[:space:]]*$/ {
            gsub(/[[:space:]]/, "")
            value = $0
        }
        END {
            if (value == "") exit 1
            print value
        }
    '
) || {
    echo "ERROR: No se pudo obtener un total numerico."
    exit 1
}

echo "Valor detectado: $TOTAL"

# =====================================================
# 4. ENVIAR METRICA A DYNATRACE
# =====================================================

METRIC_PAYLOAD=$(printf '%s %s' "$METRIC_KEY" "$TOTAL")

METRIC_HTTP_CODE=$(
    curl -sS \
        -o /tmp/dt_metric_response.txt \
        -w '%{http_code}' \
        -X POST "$DT_METRIC_URL" \
        -H "Content-Type: text/plain; charset=utf-8" \
        --data-binary "$METRIC_PAYLOAD"
) || {
    echo "ERROR: No fue posible conectar con OneAgent."
    exit 1
}

if [[ "$METRIC_HTTP_CODE" != "202" ]]; then
    echo "ERROR: Ingestion de metrica HTTP $METRIC_HTTP_CODE"
    cat /tmp/dt_metric_response.txt
    exit 1
fi

echo "Metrica enviada correctamente a Dynatrace."

# =====================================================
# 5. REGLA DE NEGOCIO: TOTAL IGUAL A CERO
# =====================================================

if [[ "$TOTAL" == "0" ]]; then

    echo "ALERTA: El total de cotizaciones es cero."

    # Crear un evento de error visible en Problems
    EVENT_PAYLOAD=$(
        jq -n \
            --arg title "ALERTA NEGOCIO: Total de cotizaciones igual a cero" \
            --arg host "$DT_ENTITY_HOST" \
            --arg total "$TOTAL" \
            '{
                eventType: "ERROR_EVENT",
                title: $title,
                entitySelector:
                    ("type(HOST),entityName.equals(" + $host + ")"),
                timeout: 15,
                properties: {
                    "metric.name":
                        "sqlserver_PCBS.SEGUROS_AUTOS_MERCOSUR.TOTAL_2",
                    "business.rule": "TOTAL_IGUAL_CERO",
                    "business.value": $total,
                    "source": "Bash_SQL_Server"
                }
            }'
    )

    EVENT_HTTP_CODE=$(
        curl -sS \
            -o /tmp/dt_event_response.json \
            -w '%{http_code}' \
            -X POST "$DT_EVENTS_URL" \
            -H "Authorization: Api-Token $DT_API_TOKEN" \
            -H "Content-Type: application/json" \
            --data-binary "$EVENT_PAYLOAD"
    ) || {
        echo "ERROR: No fue posible enviar el evento a Dynatrace."
        exit 1
    }

    if [[ "$EVENT_HTTP_CODE" == "201" ]] &&
       jq -e '.eventIngestResults[] | select(.status == "OK")' \
           /tmp/dt_event_response.json >/dev/null; then
        echo "Evento de alerta aceptado por Dynatrace."
    else
        echo "ERROR: Dynatrace no confirmo la creacion del evento."
        cat /tmp/dt_event_response.json
        exit 1
    fi

else
    echo "OK: El total es mayor que cero. No se genera alerta."
fi

echo "Proceso finalizado."
