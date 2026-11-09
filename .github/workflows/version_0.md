Opción 1: Crear un evento cuando el valor sea 0

Tu bloque es correcto, pero mejoraría el entitySelector:

Shell
EVENT_PAYLOAD=$(
jq -n \
--arg title "ALERTA NEGOCIO: Total de cotizaciones igual a cero" \
--arg host "$DT_ENTITY_HOST" \
--arg total "$TOTAL" \
'{
eventType: "ERROR_EVENT",
title: $title,
entitySelector: ("type(HOST),entityName.equals(\"" + $host + "\")"),
timeout: 15,
properties: {
"metric.name":"sqlserver_PCBS.SEGUROS_AUTOS_MERCOSUR.TOTAL_2",
"business.rule":"TOTAL_IGUAL_CERO",
"business.value":$total,
"source":"Bash_SQL_Server"
}
}'
)

Valida además que:

Shell
echo "$EVENT_PAYLOAD" | jq .

genere JSON válido.

Opción 2 (Recomendada): Crear el problema desde una métrica

Tu script ya envía la métrica:

Shell
sqlserver_PCBS.SEGUROS_AUTOS_MERCOSUR.TOTAL_2 0

Lo mejor es:

1. Enviar siempre la métrica
Shell
METRIC_KEY="sqlserver_PCBS.SEGUROS_AUTOS_MERCOSUR.TOTAL_2"
2. Crear una Metric Event en Dynatrace

Ir a:

Plain Text
Settings
→ Anomaly Detection
→ Metric Events
→ Add metric event

Configurar:

Campo	Valor
Metric	sqlserver_PCBS.SEGUROS_AUTOS_MERCOSUR.TOTAL_2
Threshold	Lower than
Value	1
Samples	1 de 1
Severity	Custom Alert
Event title	Total de cotizaciones igual a cero

De esta forma:

Plain Text
TOTAL = 10 -> OK
TOTAL = 5 -> OK
TOTAL = 1 -> OK
TOTAL = 0 -> Problem automático

Sin necesidad de llamar al API de eventos.

Opción 3: Forzar un evento DAVIS de tipo CUSTOM_ALERT

En lugar de:

JSON
"eventType":"ERROR_EVENT"

usar:

JSON
"eventType":"CUSTOM_ALERT"

Ejemplo:

Shell
EVENT_PAYLOAD=$(
jq -n \
'{
eventType: "CUSTOM_ALERT",
title: "ALERTA NEGOCIO: Total de cotizaciones igual a cero",
timeout: 60,
entitySelector: "type(HOST),entityName.equals(\"HOST-PCBS\")",
properties: {
"business.value": "0",
"service": "COTIZACIONES"
}
}'
)

En varias implementaciones de Dynatrace esto genera una correlación más visible para Davis AI.

Algo que también corregiría

La extracción de TOTAL podría fallar si sqlcmd devuelve sólo 0 sin espacios.

Yo usaría:

Shell
TOTAL=$(echo "$SQL_OUTPUT" | grep -Eo '^[0-9]+$' | tail -n1)
 

o

Shell
TOTAL=$(echo "$SQL_OUTPUT" | tr -d '[:space:]')

si tu consulta devuelve únicamente el contador.

Mi recomendación

Como administrador de Dynatrace, la solución más robusta es:

Mantener tu envío de métrica personalizada.
Crear un Metric Event con condición < 1.
Dejar que Davis AI genere automáticamente el Problem.
