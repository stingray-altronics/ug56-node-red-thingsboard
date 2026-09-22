# UG56 → ThingsBoard: registro de dispositivos y downlinks genéricos

Flujo v2 para Node-RED dentro del UG56, LNS Milesight embebido y ThingsBoard CE 3.5.0. MQTT estándar, uplink genérico, control de RO/DO del LT-22222-L e intervalo de reporte de LT-22222-L y AM102. Requiere Node-RED 2.1 o posterior, sin paquetes adicionales.

Importar [UG56-ThingsBoard-mqtt-flow.json](UG56-ThingsBoard-mqtt-flow.json). La plantilla pública tiene registro vacío, host ThingsBoard de ejemplo y ninguna credencial ni identificador real. Comienza en **simulación**.

## Migración y conexiones

1. Exportar su flujo instalado y deshabilitar su pestaña antes de desplegar esta versión, para evitar comandos y suscripciones duplicados.
2. Importar el JSON. La pestaña **Open Lab UG56 MQTT - Registry** tiene IDs nuevos y no sustituye silenciosamente el flujo anterior. Volver a introducir las credenciales en sus nuevos nodos de conexión.
3. Configurar **EDITAR CREDENCIALES - LNS local UG56**: `127.0.0.1:1883`, sin TLS, con usuario y contraseña de su broker. Loopback funciona porque Node-RED se ejecuta en el UG56.
4. Configurar **EDITAR TOKEN - ThingsBoard CE 3.5.0** con host real, puerto `8883`, TLS y Access Token del dispositivo marcado como gateway en ThingsBoard como usuario MQTT. Dejar contraseña vacía.
5. Sustituir `LT_DEVEUI`, `AM_DEVEUI`, `LNS_DOWNLINK_APP_ID` y `LT_DOWNLINK_FPORT` por entradas en `DEVICE_REGISTRY`. Esas variables antiguas ya no se utilizan.
6. Con `ENABLE_DOWNLINK=false`, desplegar y verificar telemetría y RPC en simulación. Habilitar downlinks solo después de comprobar destino, bytes, clase y FPort. En el taller bastan los indicadores incorporados del LT; no se conectan cargas externas.

La plantilla pública activa **Verify server certificate**, sin certificados de cliente ni claves privadas. Si su instalación requiere desactivarlo, desmarcar esa opción en el nodo TLS: se mantiene el cifrado, pero se omite comprobar la identidad del servidor.

ThingsBoard usa QoS 1; comandos MQTT locales, QoS 0. Publicaciones con Retain desactivado. `confirmed:false` es una opción LoRaWAN independiente del QoS. La respuesta del flujo no confirma aceptación por el broker, LNS ni dispositivo.

## Variables: dónde están

Hacer doble clic en la **pestaña** del flujo → **Environment variables** → editar → **Done → Deploy**. No son nodos Function. Las funciones las leen mediante `env.get(...)`.

| Variable | Tipo | Finalidad |
|---|---|---|
| `DEVICE_REGISTRY` | JSON | Una entrada por DevEUI para registro inicial y autorización de RPC. |
| `LNS_UPLINK_TOPIC` | Texto | Inicialmente `application/2/device/+/rx`. |
| `ENABLE_DOWNLINK` | Booleano | `false`: simular; `true`: publicar comandos. |
| `STATE_MAX_AGE_S` | Número | Antigüedad máxima de estado RO/DO para get; inicial 900 s. |
| `MIN_COMMAND_GAP_S` | Número | Pausa mínima entre escrituras al mismo dispositivo; inicial 2 s. |

No confundir intervalo de reporte, antigüedad del estado y timeout RPC. Por ejemplo: LT reportando cada 300 s, antigüedad máxima 900 s, timeout del widget 10000 ms. El flujo responde al preparar el envío, sin esperar actuación ni uplink.

## Registro escalable

Ejemplo con identificadores ficticios; reemplazarlos por sus equipos:

```json
{
  "0011223344556601": {
    "model": "LT22222", "applicationId": "2", "fPort": 2
  },
  "0011223344556602": {
    "model": "LT22222", "applicationId": "7", "fPort": 2,
    "stateMaxAgeS": 1800
  },
  "0011223344556603": {
    "model": "AM102", "applicationId": "2", "fPort": 85
  },
  "0011223344556604": {
    "model": "AM102", "applicationId": "7", "fPort": 85
  }
}
```

- Clave: DevEUI de 16 caracteres hexadecimales, normalizado a minúsculas. Se rechazan duplicados con distinta capitalización.
- `model`: exactamente `LT22222` o `AM102`. Selecciona el helper; otros modelos pueden seguir enviando uplink fuera del registro.
- `applicationId`: ID de la aplicación del equipo en el LNS, no AppEUI/JoinEUI ni FPort.
- `fPort`: entero 1–223 para comandos. Verificar 2 en el LT según firmware; AM102 usa 85 por defecto.
- `stateMaxAgeS`: opcional, sustituye la variable general para ese LT. Si se omite, se aplica `STATE_MAX_AGE_S`.

Para añadir equipos: registrar primero OTAA y codec en el LNS, añadir una entrada al JSON, desplegar y pulsar **Registrar dispositivos conocidos**. Crear los alias/widgets de los nuevos hijos en ThingsBoard. No hacen falta variables por dispositivo ni copias del flujo.

Para varias aplicaciones, usar `application/+/device/+/rx`, o filtrar las aplicaciones autorizadas antes del conversor. Añadir una aplicación al registro **no amplía automáticamente la suscripción MQTT**. Los downlinks sí usan el ID de cada entrada.

Los métodos existentes get/setRO1, RO2, DO1 y DO2 no cambian. El destino del widget debe ser el hijo ThingsBoard cuyo nombre es el DevEUI en minúsculas, no el gateway lógico.

## Arquitectura

```text
MQTT LNS → Uplink genérico → Estados LT por dispositivo → MQTT ThingsBoard

                                      ┌→ Helper LT: control RO/DO ─┐
MQTT ThingsBoard → Registro y método ──┤                           ├→ Emisor genérico → MQTT LNS
                                      └→ Helper: intervalo ───────┘
```

Los helpers generan los bytes del protocolo. El emisor común recibe hex, valida, consulta el registro y publica en `application/<applicationId>/device/<DevEUI>/tx`, con la envoltura siguiente (`data` es Base64):

```json
{"confirmed":false,"fport":2,"data":"AwER"}
```

Dejar vacío el topic de **Downlink LNS local** para usar `msg.topic`. El emisor no codifica comandos de fabricante. No se expone un método raw al dashboard.

## Control LT y estado

| Acción | Método | Parámetro JSON | Hex generado |
|---|---|---|---|
| Cerrar / abrir RO1 | `setRO1` | `true` / `false` | `030111` / `030011` |
| Cerrar / abrir RO2 | `setRO2` | `true` / `false` | `031101` / `031100` |
| Activar / liberar DO1 | `setDO1` | `true` / `false` | `02011111` / `02001111` |
| Activar / liberar DO2 | `setDO2` | `true` / `false` | `02110111` / `02110011` |

Booleanos sin comillas; también se admiten números 0/1. Los bytes `11` conservan salidas vecinas. LED RO encendido indica relé cerrado; LED DO encendido indica salida activa L. Comparar LED y uplink posterior.

`getRO1`, `getRO2`, `getDO1` y `getDO2` devuelven booleanos del último estado recibido. Se rechaza estado ausente o antiguo. La memoria es independiente por dispositivo **y por salida**: un reporte de RO1 no renueva la antigüedad de RO2. Uplinks atrasados no sustituyen estados más recientes. Enviar una orden no cambia el estado almacenado.

## Intervalo de reporte desde ThingsBoard

1. Añadir un widget que permita enviar método RPC y parámetros JSON, dirigido al hijo LT o AM102.
2. Método: **`setReportingInterval`**. Parámetros: **`300`**, sin comillas ni objeto envolvente. Significa 300 segundos = 5 minutos.
3. Usar petición de dos vías y timeout 10000 ms. No configurar `getReportingInterval`: esta revisión no ofrece lectura verificada del intervalo.
4. Probar en simulación y revisar **Comando preparado / simulacion**. Habilitar downlinks y enviar una solicitud nueva.

ThingsBoard construye la envoltura Gateway API; el widget solo define método y parámetros:

```json
{
  "device": "0011223344556603",
  "data": {"id": 42, "method": "setReportingInterval", "params": 300}
}
```

| Modelo | Rango admitido por este flujo | Codificación | Ejemplo 300 s |
|---|---|---|---|
| LT22222 | 30–86400 s | `01` + 3 bytes, big endian | `0100012c` |
| AM102 | 60–64800 s | `ff03` + 2 bytes, little endian | `ff032c01` |

El rango LT de 30 s a 24 h es una política de esta plantilla, no el rango completo del campo de tres bytes. AM102 se limita al rango publicado de 1–1080 minutos. Solo se aceptan segundos enteros; se rechazan texto, booleanos, fracciones y valores fuera de rango.

AM102 Clase A recibe tras un uplink y puede conservar el intervalo anterior hasta entonces. No confundir reporte con retransmisión de históricos ni refresco de pantalla. Comprobar el cambio por ToolBox/consola o varios uplinks periódicos posteriores, distinguiendo eventos e históricos. El flujo no ajusta automáticamente `STATE_MAX_AGE_S`.

## Respuestas RPC y diagnóstico

En simulación: `accepted:false`, `stage:simulation`, `executed:false` y hex preparado; no publica al LNS. Activar la variable no reproduce comandos anteriores.

Una escritura enviada responde en `v1/gateway/rpc`:

```json
{
  "device":"0011223344556603", "id":42,
  "data":{
    "accepted":true, "stage":"submitted_to_mqtt_node",
    "executed":false, "requestedIntervalSeconds":300
  }
}
```

Confirma preparación y entrega al nodo MQTT, no ejecución física. Errores de registro, método, parámetro, estado o frecuencia responden con `accepted:false` y `error`. Mensajes sin destino/ID utilizables se muestran en Catch; los ecos de respuestas se ignoran.

Para un timeout, activar **RPC recibido (activar para diagnostico)** y **Respuesta RPC (activar para diagnostico)**. Comprobar destino hijo, token del gateway, suscripción `v1/gateway/rpc`, registro y errores Catch. La respuesta no espera una confirmación por radio.

Se deduplican escrituras por DevEUI + ID durante 120 s. Una repetición devuelve la respuesta anterior sin reenviar; el mismo ID con distinto comando o ruta se rechaza. La pausa de 2 s se aplica por dispositivo. Reiniciar Node-RED borra la memoria; no se garantiza entrega exactamente una vez.

## Uplink e históricos

Todo DevEUI válido con `object` decodificado puede enviar telemetría, aunque no esté en el registro. Se registra en ThingsBoard al primer uplink. El registro sí es necesario para RPC y el adaptador de estados LT.

Números, booleanos y cadenas se conservan. Objetos/listas anidados, incluido `object.history`, se publican como texto JSON. No se reconstruye la serie histórica con sus timestamps. El conversor no sustituye al codec. Los dispositivos descubiertos se vuelven a registrar tras reconexión; con contexto en memoria, se redescubren después de reiniciar.

## Validación y fuentes

**28 grupos de pruebas locales** sobre los Function y conexiones del JSON exportado: cuatro dispositivos, aplicaciones distintas, ocho comandos LT, intervalos/límites, simulación, respuestas, estado por salida, duplicados, errores, muestras MQTT v2 y exportación sin credenciales. Falta el ensayo con el UG56 y ThingsBoard reales; no se ha enviado ningún comando al hardware durante estas pruebas.

- [Dragino LT-22222-L: comandos e indicadores](https://wiki.dragino.com/docs/LoRaWAN-End-Node/io-controllers-sensor-nodes/lt-22222-l/)
- [Milesight AM102: comandos](https://www.milesight.com/products/docs/en/am102/protocol/am100/am10x-downlink.html)
- [Milesight AM102: intervalo de reporte](https://www.milesight.com/products/docs/en/am102/steps/am100/general-settings.html)
- [ThingsBoard Gateway RPC](https://thingsboard.io/docs/reference/gateway-api/rpc/)
- [Node-RED: variables de entorno](https://nodered.org/docs/user-guide/environment-variables)

## Licencia

Flujo e instrucciones bajo [licencia MIT](LICENSE). Copyright (c) 2026 stingray-altronics.
