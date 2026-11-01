Antes de ejecutarlo: el SELECT del ejemplo es ilustrativo y debes reemplazarlo por tu consulta completa, incluido el LEFT JOIN de tu código original. El usuario SQL debe tener permisos de lectura sobre las tablas involucradas.

Además, el servidor donde se ejecuta Bash debe tener instalados sqlcmd, curl, jq y acceso al OneAgent local. Si tu entorno Dynatrace no es SaaS, la URL de eventos puede ser distinta.
2. Cómo configurar Dynatrace paso a paso
Paso 1. Obtener el token de Dynatrace


1. Ingresa a tu entorno Dynatrace.
2. Busca la sección de tokens de acceso personal (Access tokens).
3. Crea un token con el permiso events.ingest.
4. Copia el token y guárdalo en un lugar seguro.

Documentación oficial: API de eventos de Dynatrace. 
POST an event — Dynatrace Docs

Paso 2. Identificar la URL de tu entorno
Si utilizas Dynatrace SaaS, la URL de eventos tiene esta estructura:

Paso 2. Identificar la URL de tu entorno
Si utilizas Dynatrace SaaS, la URL de eventos tiene esta estructura:


Reemplaza TU_ENTORNO por el identificador real de tu entorno.

Configura las variables en el servidor Linux:



https://TU_ENTORNO.live.dynatrace.com/api/v2/events/ingest

export SQL_SERVER="tu-servidor-sql"
export SQL_USER="tu-usuario-sql"
export SQL_PASS="tu-clave-sql"
export SQL_DB="pcbs"

export DT_EVENTS_URL="https://TU_ENTORNO.live.dynatrace.com/api/v2/events/ingest"
export DT_API_TOKEN="TU_TOKEN_DYNATRACE"
export DT_ENTITY_HOST="NOMBRE_REAL_DEL_HOST"

El nombre de host debe corresponder a una entidad existente en Dynatrace. El token es un secreto: no lo incluyas directamente en el archivo Bash ni lo publiques en registros.
Paso 3. Identificar la entidad donde aparecerá la alerta
En Dynatrace:
1. Ingresa a Infrastructure & Operations → Hosts (el nombre de la sección puede variar según tu versión).
2. Busca el servidor Linux donde corre el script y OneAgent.
3. Copia el nombre exacto del host.
4. Asigna ese nombre a DT_ENTITY_HOST.
Por ejemplo, si el host se llama srv-linux-pcbs-prod, configura:
export DT_ENTITY_HOST="srv-linux-pcbs-prod"


La alerta quedará asociada a ese host. Si prefieres asociarla a un servicio u otra entidad, debemos adaptar el entitySelector.
Paso 4. Guardar y ejecutar el Bash
En Linux, ejecuta:
chmod 750 monitor_cotizacion_dynatrace.sh
./monitor_cotizacion_dynatrace.sh


El comportamiento esperado es el siguiente:
Caso A: el resultado es mayor que cero

Ejemplo: TOTAL = 25
- Envía la métrica a Dynatrace.
- No genera un nuevo evento de error.



Caso B: el resultado es cero

Ejemplo: TOTAL = 0
- Envía la métrica con valor 0.
- Envía un evento ERROR_EVENT.
- Dynatrace acepta el evento y, si la entidad y el tipo de evento son válidos, genera el problema correspondiente.



Antes de ejecutarlo en producción, prueba ambos escenarios en un ambiente de pruebas. El código de respuesta HTTP 201 y el estado OK confirman que Dynatrace aceptó el evento, pero debes comprobar también su aparición en Problems.
Paso 5. Verificar la alerta en Problems









1. Abrir Problems

Ingresa a la aplicación Problems de Dynatrace.











2. Buscar el problema

Busca el título:

ALERTA NEGOCIO: Total de cotizaciones igual a cero


3. Revisar el detalle

Comprueba que el problema esté asociado al host esperado y que la propiedad business.value tenga valor 0.



Si no aparece, revisa la respuesta de /tmp/dt_event_response.json, el selector de entidad, el permiso del token y los registros de la ejecución. La documentación de la API explica los tipos de eventos, sus tiempos de espera y los estados de ingestión. 
POST an event — Dynatrace Docs


En este ejemplo, el evento tiene un tiempo de espera de 15 minutos. Si la consulta vuelve a dar cero y el script se ejecuta periódicamente, el evento puede refrescarse; si deja de enviarse, el problema puede cerrarse al vencer ese tiempo. El código no cierra el problema inmediatamente cuando el total vuelve a ser mayor que cero.
3. Opcional: crear la alerta mediante una regla de métricas
Si quieres que Dynatrace evalúe continuamente el valor de la métrica y gestione la alerta a través de su motor de detección, puedes utilizar esta alternativa:
1. Ejecuta el script para que la métrica se ingiera al menos una vez.
2. En Dynatrace, entra en Settings → Anomaly detection → Metric events.
3. Selecciona Add metric event.
4. Busca la métrica sqlserver_PCBS.SEGUROS_AUTOS_MERCOSUR.TOTAL_2.
5. Configura una condición de valor por debajo de 1, dado que el resultado es un conteo entero no negativo.
6. Define el título de la alerta, la severidad y la entidad correspondiente.
7. Guarda la regla y verifica su comportamiento con valores de prueba.
Esta opción permite crear problemas mediante la detección de métricas, sin que el Bash tenga que enviar un ERROR_EVENT adicional. La disponibilidad exacta de los campos depende de la versión de Dynatrace. 
Dynatrace Docs+1


Documentación oficial: Extender Dynatrace con métricas personalizadas.

Mi recomendación: utiliza una sola estrategia de alertamiento para evitar duplicados: el ERROR_EVENT directo del Bash o la regla de métrica de Dynatrace. Para tu caso, comenzaría con el evento directo si necesitas generar el problema específicamente cuando la consulta devuelve cero.

Pensar

