# IOT Open Lab: UG56 con MQTT local

Versión actual: nodos estándar MQTT in/out, LNS Milesight embebido y Node-RED dentro del UG56. AM102, LT-22222-L, AU915 8–15 y ThingsBoard CE 3.5.0.

## Archivo para importar

Importar [UG56-ThingsBoard-mqtt-flow.json](UG56-ThingsBoard-mqtt-flow.json). Los DevEUI están vacíos:
configurar los equipos reales antes de usar las funciones de control.
Los identificadores de los ejemplos de este repositorio son ficticios.

## Conexiones y puesta en marcha

1. Exportar el flujo instalado y deshabilitar su pestaña antes de desplegar el nuevo. Los identificadores de esta revisión son diferentes.
2. Importar el flujo MQTT. Configurar **EDITAR CREDENCIALES - LNS local UG56**: host `127.0.0.1`, puerto `1883`, MQTT 3.1.1, sin TLS. En Security introducir el usuario y la contraseña configurados en su broker. Las credenciales no se incluyen en el JSON ni en el material compartido.
3. El puerto del broker MQTT local es 1883. Verificar estado conectado. `127.0.0.1` funciona porque Node-RED se ejecuta en el propio UG56.
4. Mantener habilitado el LNS, comprobar su conexión/transmisión hacia ese broker y los codecs. El ID de aplicación depende de cada instalación. Ajustar **LNS_UPLINK_TOPIC** a `application/<APP_ID>/device/+/rx` y **LNS_DOWNLINK_APP_ID** al ID de la aplicación del LT. El valor inicial 2 corresponde únicamente a las muestras. Si ambos equipos pertenecen a la misma aplicación, usar ese mismo ID en ambos ajustes. El `+` ocupa únicamente el segmento DevEUI.
5. Configurar el segundo broker, **EDITAR - ThingsBoard CE 3.5.0**, con host real, puerto, TLS y Access Token del dispositivo gateway como usuario. Mantener separados ambos brokers y sus credenciales. La plantilla ThingsBoard usa TLS/8883; ajustarlo al servidor real.
6. Mantener `ENABLE_DOWNLINK=false`, hacer Deploy y activar **MQTT LNS: mensaje completo**. Revisar `msg.payload.devEUI`, `msg.payload.object`, `msg.payload.time` y `msg.topic`. El nodo MQTT in ya produce JSON: no añadir otra conversión obligatoria.
7. Comprobar registro y telemetría de los hijos ThingsBoard. El Inject **Registrar dispositivos conocidos** repite el registro; también se ejecuta tras reconectar. Configurar el gateway ThingsBoard como gateway.
8. Ensayar RPC en simulación. Revisar bytes y topic. Después de verificar Clase C, cableado y FPort LT, habilitar downlinks y probar una salida por vez. Examinar Packets, actuación y uplink posterior.

Uplink: LNS → broker local → MQTT in → conversor genérico → adaptador LT opcional → MQTT out ThingsBoard.

Control: MQTT in ThingsBoard → función RPC → MQTT out local → broker → LNS → LT.

## Variables de pestaña

Requieren Node-RED 2.1 o posterior.

| Variable | Inicial | Uso |
|---|---|---|
| `LNS_UPLINK_TOPIC` | `application/2/device/+/rx` | Cambiar el ID del topic por el de la instalación |
| `LNS_DOWNLINK_APP_ID` | `2` | Cambiar por el ID de aplicación del LT |
| `LT_DEVEUI` | Vacío: configurar | Equipo controlado |
| `AM_DEVEUI` | Vacío: opcional para registro inicial | Registro inicial AM102 |
| `LT_DOWNLINK_FPORT` | `2` | Puerto de aplicación LoRaWAN del LT |
| `ENABLE_DOWNLINK` | `false` | Simulación |
| `STATE_MAX_AGE_S` | `900` | Antigüedad máxima del estado |

El ID de aplicación LNS y el FPort son conceptos distintos aunque ambos valgan 2 aquí.

## Downlink MQTT

El MQTT out local deja Topic vacío y usa `msg.topic`. Ejemplo con aplicación 2, para cerrar RO1:

```json
{
  "topic": "application/2/device/0011223344556601/tx",
  "payload": {"confirmed": false, "fport": 2, "data": "AwER"},
  "qos": "0",
  "retain": false
}
```

`data` contiene Base64 de `030111`; `fport` se escribe en minúsculas. Los uplinks usan `fPort`: no copiar esa capitalización al downlink. MQTT out serializa el objeto JSON. Se usa QoS 0 local para no pedir reintentos MQTT de comandos; no garantiza entrega. MQTT ThingsBoard conserva QoS 1 y deduplicación RPC en la función.

La propiedad `confirmed=false` pertenece a LoRaWAN, no al QoS MQTT. No se requieren los nodos LoRa Input/Output ni la mejora de payload opcional de su biblioteca. Registrar el firmware y comprobar la integración MQTT antes del taller; actualizar solo si es necesario y siguiendo las notas oficiales.

## Uplink genérico: añadir cualquier modelo

El nodo **Uplink generico: object a ThingsBoard** acepta cualquier DevEUI válido recibido del LNS. No requiere añadir su modelo, nombre de medición ni DevEUI al código. AM_DEVEUI y LT_DEVEUI son registros iniciales del taller; no son una lista de dispositivos permitidos para uplink. LT_DEVEUI sigue limitando los controles RPC y el adaptador de estados.

1. Registrar el nuevo dispositivo en el LNS del UG56 y asignar el codec apropiado para su modelo.
2. Comprobar que MQTT in del LNS entrega el DevEUI y `object` decodificado.
3. Al recibir el primer uplink válido, el flujo envía `v1/gateway/connect` con el DevEUI en minúsculas como nombre y luego publica todos los campos de `object` en `v1/gateway/telemetry`.
4. Seleccionar esas claves en el dashboard; no es necesario modificar el conversor.

Se aceptan mensajes nativos en la raíz o una envoltura JSON en `msg.payload`; alias `deveui`, `devEUI`, `eui`. `object` puede ser objeto o texto JSON. Números, booleanos y cadenas se conservan, incluidos cero y false. Objetos y listas anidados se conservan como texto JSON bajo su clave original; para graficar elementos internos haría falta un mapeo adicional. Se omiten null y valores no publicables. Los metadatos no reemplazan claves del codec.

**Límite:** este nodo formatea datos; no descifra ni decodifica bytes de cualquier fabricante. Si falta `object`, está vacío o es inválido, el mensaje se rechaza con un error de codec. No se adivinan campos de una envoltura aplanada: debe adaptarse a `object` antes de este nodo.

**Opcional LT: estados para RPC** es el único nodo uplink específico de un modelo. Añade los booleanos de RO/DO y guarda el estado para las consultas RPC. Puede puentearse conectando el conversor genérico directamente a la salida MQTT cuando solo se necesite telemetría genérica. El control del LT requiere mantenerlo.

Los dispositivos descubiertos se guardan en contexto de flujo para volver a registrarlos tras una reconexión MQTT. Con el almacenamiento en memoria predeterminado, la lista se pierde al reiniciar Node-RED y se reconstruye con los siguientes uplinks. El registro no crea automáticamente los widgets del dashboard.

## Comandos y RPC

| Acción | Método / parámetro JSON | Hexadecimal | Base64 generado |
|---|---|---|---|
| Abrir RO1 | `setRO1` / `false` | `030011` | `AwAR` |
| Cerrar RO1 | `setRO1` / `true` | `030111` | `AwER` |
| Abrir RO2 | `setRO2` / `false` | `031100` | `AxEA` |
| Cerrar RO2 | `setRO2` / `true` | `031101` | `AxEB` |
| Liberar DO1 | `setDO1` / `false` | `02001111` | `AgAREQ==` |
| Activar DO1 | `setDO1` / `true` | `02011111` | `AgEREQ==` |
| Liberar DO2 | `setDO2` / `false` | `02110011` | `AhEAEQ==` |
| Activar DO2 | `setDO2` / `true` | `02110111` | `AhEBEQ==` |

`true` cierra el relé o activa la salida NPN (L). `false` abre el relé o libera la salida NPN. Los bytes `11` dejan las otras salidas sin cambios.

Los widgets y los métodos get/setRO1, RO2, DO1 y DO2 no cambian. La función solo permite controlar el LT configurado; el uplink admite cualquier dispositivo decodificado de la aplicación suscrita.

## Estado y límites de la validación

La respuesta de escritura indica `accepted=true`, `stage=submitted_to_mqtt_node`, `executed=false`. Confirma entrega a la ruta local, no aceptación por el LNS ni actuación. Revisar Packets, carga y uplink posterior.

Las consultas `getRO1`, `getRO2`, `getDO1` y `getDO2` devuelven el último estado recibido; no realizan un sondeo instantáneo. Las claves son `ro1_active`, `ro2_active`, `do1_active`, `do2_active`. Un estado ausente o más antiguo que `STATE_MAX_AGE_S` se rechaza.

Se mantiene una pausa mínima de dos segundos entre órdenes y protección de ID RPC duplicado durante dos minutos, en memoria. Reiniciar Node-RED borra esa protección. No se garantiza entrega exactamente una vez.

El flujo conserva el timestamp original. Reproducir las muestras no crea mediciones actuales; ajustar la ventana temporal del dashboard.

**29 comprobaciones locales aprobadas**: muestras, bytes, ocho acciones por la misma salida dinámica, variables, validación y estados. No se ejecutó la broker local del UG56 ni se accedió al ThingsBoard del usuario. Completar el ensayo real antes del taller.

## Fuentes

- [Milesight: LoRa Input y codec](https://support.milesight-iot.com/support/solutions/articles/73000535734-how-to-use-decoder-on-node-red)
- [Milesight: downlink MQTT](https://support.milesight-iot.com/support/solutions/articles/73000514234-fail-to-control-device-via-mqtt-downlink-command)
- [Node-RED: variables](https://nodered.org/docs/user-guide/environment-variables)
- Exportación nativa y muestras del usuario.


## Licencia

Este flujo y sus instrucciones se distribuyen bajo la [licencia MIT](LICENSE).
Copyright (c) 2026 Gonzalo Silva.
