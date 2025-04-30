# Implementacion_NODE_RED_Argumentario_y_Guia.md

## 1. Estado actual del sistema

### Arquitectura

- **Sensores LoRaWAN**: EM300-ZDL, EM300-SLD, EM300-MCS, etc.
- **ChirpStack**: Recepción de mensajes y decodificación básica.
- **Mosquitto MQTT Broker**: Distribución de mensajes.
- **Telegraf**: Consumo de MQTT y escritura en InfluxDB.
- **InfluxDB**: Almacenamiento de datos en bucket `iot-datos`.
- **Grafana**: Visualización de métricas en dashboards.

### Configuraciones clave actuales

- Telegraf usa `data_format = "json"`.
- Se capturan campos básicos (`humidity`, `temperature`, `battery`, `rssi`, `snr`).
- No se desanidan objetos JSON (`object`) automáticamente.
- Solo se capturan `tag_keys` explícitos.

---

## 2. Problemas encontrados

### 2.1 MQTT Payload structure

- Tramas recibidas contienen datos anidados en `object`.
- Ejemplo real capturado:

```json
"object": {
  "humidity": 48.5,
  "temperature": 23.5,
  "leakage_status": "normal"
}
```

### 2.2 Limitaciones de Telegraf actual

- No puede "bajar" dentro de `object` usando `data_format = "json"` básico.
- El `fieldpass` no funciona si el campo está anidado.
- `data_format = "json_v2"` no pudo implementarse por incompatibilidades detectadas (error de tipo en configuraciones como `type = "tag"`).
- Resultado: `_leakage_status` no llega como `field` en InfluxDB.

### 2.3 Decoder en ChirpStack

- Aunque se modificó para exponer `_leakage_status`, sigue dentro de `object`.
- ChirpStack encapsula siempre la decodificación bajo `object` por diseño.
- Modificaciones en el decoder JavaScript lograron duplicar el campo como `_leakage_status`, pero Influx no lo detecta si no está fuera del objeto raíz.

### 2.4 InfluxDB y Grafana

- Sin acceso directo a `_leakage_status` u otros estados críticos ("leak", "open/close").
- No se pueden construir paneles ni alarmas directamente.
- Consulta a Influx con:

```flux
from(bucket: "iot-datos")
  |> range(start: -6h)
  |> filter(fn: (r) => r._measurement == "uplink_lora")
  |> filter(fn: (r) => r._field == "_leakage_status")
  |> sort(columns: ["_time"], desc: true)
  |> limit(n: 10)
```

- Resultado: **0 filas devueltas**.

---

## 3. Capturas y ejemplos originales

### MQTT Uplink EM300-ZDL ejemplo:

```json
"object": {
  "_leakage_status": "normal",
  "humidity": 48.5,
  "leakage_status": "normal",
  "temperature": 23.5
}
```

### Decoder personalizado aplicado:

```javascript
function Decode(fPort, bytes) {
  var decoded = milesightSLDDecode(bytes);
  var result = {};
  if (decoded.leakage_status !== undefined) result._leakage_status = decoded.leakage_status;
  if (decoded.humidity !== undefined) result.humidity = decoded.humidity;
  if (decoded.temperature !== undefined) result.temperature = decoded.temperature;
  if (decoded.battery !== undefined) result.battery = decoded.battery;
  return result;
}
```

### Resultado esperado (no logrado sin node-red):

```json
{
  "_leakage_status": "normal",
  "humidity": 48.5,
  "temperature": 23.5,
  "battery": 92
}
```

---

## 4. Argumentos para la implementación de Node-RED

### 4.1 Resolución de problemas actuales

- Desanidar automáticamente el objeto `object`.
- Exponer `leakage_status`, `open_close`, `alarm_status`, etc., al primer nivel.
- Renombrar campos para uso estandarizado (`leakage_status` → `_leakage_status`).
- Asegurar que todos los datos críticos sean visibles y capturables desde Influx.

### 4.2 Beneficios estratégicos

- Control total sobre el formato final enviado a InfluxDB.
- Adaptabilidad a nuevos sensores y tipos de payload sin modificar decoders.
- Generación de alertas en tiempo real (Telegram, Email, Discord, etc.).
- Mejor mantenimiento y extensibilidad del sistema.
- Permite probar y validar estructuras antes de escribirlas en la base de datos.

### 4.3 Futuro crecimiento

- Integrar lógicas más complejas basadas en combinaciones de sensores.
- Posibilidad de hacer enroutamiento condicional (sensor A → Influx, sensor B → PostgreSQL).
- Poder crear logs o trazas auditables para cada tipo de evento.
- Visualizar en tiempo real los datos que están entrando al sistema antes de persistirlos.

---

## 5. Propuesta de flujo Node-RED

```text
[ChirpStack MQTT] → [Node-RED procesamiento] → [MQTT Broker limpio] → [Telegraf] → [InfluxDB] → [Grafana]
```

Node-RED permitirá:
- Aplanar JSON.
- Renombrar o eliminar campos.
- Agregar metainformación.
- Consolidar diferentes decoders bajo un único flujo de control.
- Rastrear problemas o sensores defectuosos en tiempo real.

---

## 6. Guía de actuación para implementación de Node-RED

### 6.1 ¿Dónde debe instalarse Node-RED?

- **Recomendado**: En el **host donde se encuentra Mosquitto** (el broker MQTT), porque Node-RED se suscribirá directamente a los tópicos MQTT de ChirpStack.
- **Alternativas**: Puede estar también en un contenedor separado o máquina virtual, siempre que tenga acceso de red al broker MQTT.

### 6.2 ¿Node-RED sustituye algo?

- ❌ **No reemplaza ChirpStack, Mosquitto, ni Telegraf**.
- ✅ **Se inserta entre Mosquitto y Telegraf como capa de procesamiento y normalización.**

### 6.3 Pasos recomendados

1. **Validar que Mosquitto publica correctamente todos los datos**.
2. **Instalar Node-RED en un host con acceso a Mosquitto**.
3. **Crear flujo básico Node-RED**:
   - `mqtt in` ➔ desanidar payload ➔ renombrar ➔ `mqtt out`
4. **Reconfigurar Telegraf para leer del nuevo tópico MQTT limpio**.
5. **Confirmar que los campos como `_leakage_status` ya aparecen en Influx.**
6. **Crear dashboards y paneles de visualización/alertas en Grafana.**

### 6.4 Validaciones clave en cada paso

- ¿MQTT original está accesible? (usar `mosquitto_sub`)
- ¿Node-RED transforma correctamente el payload? (ver nodos `debug`)
- ¿El nuevo JSON plano contiene los campos esperados? (`_leakage_status`, etc.)
- ¿Telegraf los recibe? (consultas de verificación en InfluxDB)
- ¿Grafana los muestra correctamente? (dashboards funcionales)

---

## 7. Visualización del mensaje final en Node-RED

Para facilitar la validación de los datos que están siendo reenviados por Node-RED hacia el nuevo tópico limpio, se recomienda añadir un nodo `debug` conectado a la salida del nodo `function` (desanidar y aplanar).

### 7.1 Propósito del nodo debug

- Visualizar en tiempo real el objeto final aplanado.
- Confirmar que `_leakage_status`, `humidity`, `temperature` y otros campos están correctamente expuestos al nivel raíz de `msg.payload`.
- Verificar el formato final antes de que Telegraf lo consuma.

### 7.2 Cómo añadirlo

1. En la interfaz de Node-RED, desde la paleta izquierda, arrastra un nodo `debug`.
2. Conéctalo a la salida del nodo `function`.
3. Configura el nodo para mostrar `msg.payload`.
4. Haz clic en `Deploy`.
5. Abre el panel lateral derecho para visualizar los resultados en tiempo real.

Esto facilita identificar cualquier error de estructura o campo inesperado y acelerar el ajuste del flujo antes de pasar a su integración definitiva con Telegraf + InfluxDB.

---

## 8. Ejecución de Node-RED como servicio en segundo plano

Por defecto, si se ejecuta Node-RED desde el terminal con `node-red`, el servicio se detendrá al cerrar la terminal. Para dejarlo funcionando como un servicio del sistema, se recomienda utilizar `pm2`.

### 8.1 Instalar y configurar `pm2`

```bash
sudo npm install -g pm2
pm2 start $(which node-red)
pm2 save
pm2 startup
```

Esto:

- Lanza Node-RED como proceso en segundo plano.
- Garantiza que se reinicie automáticamente tras un reinicio del sistema.
- Permite acceder a la interfaz en `http://localhost:1880` incluso sin sesión de terminal activa.

---

## 9. Diagnóstico visual y solución de errores de conexión MQTT en Node-RED

Durante la implementación del flujo Node-RED, puede aparecer el mensaje de error:

```
"missing broker configuration"
```

Esto indica que alguno de los nodos MQTT (IN o OUT) **no tiene correctamente asignado un broker** o que este **no está bien definido**.

### 9.1 Cómo identificarlo visualmente

- Los nodos `mqtt in` o `mqtt out` muestran un **círculo azul o sin color** en su borde izquierdo.
- Si el broker está correctamente configurado, ese círculo será **🟢 verde**.

### 9.2 Pasos para resolverlo

1. Haz doble clic en el nodo `MQTT IN` o `MQTT OUT`.
2. En el campo **Servidor**, selecciona o crea uno nuevo haciendo clic en el lápiz ✏️.
3. Asegúrate de los siguientes datos:
   - **Servidor**: `localhost`
   - **Puerto**: `1883`
   - **Sin TLS, sin usuario/contraseña** si tu Mosquitto no lo requiere
4. Pulsa **“Añadir”** o **“Actualizar”**, luego **“Hecho”**.
5. Repite para cada nodo MQTT involucrado.
6. Finalmente, haz clic en **“Deploy”** para aplicar los cambios.

Cuando está correctamente conectado, el nodo mostrará el **estado verde** y comenzará a recibir mensajes si el broker MQTT está activo.

---

## 10. Configuración técnica final validada en Node-RED

### 10.1 Estructura del flujo activo

```
[mqtt in] → [Desanidar y aplanar] → [mqtt out]
                           ↘
                        [debug 1]
```

### 10.2 Configuración del nodo MQTT IN

- **Servidor**: `MOSQUITO MQTT LOCAL`
- **Tópico**: `application/+/device/+/event/up`
- **QoS (CdS)**: `2`
- **Salida**: `auto-detectar (objeto JSON, texto o buffer)`

### 10.3 Configuración del broker MQTT (MOSQUITO MQTT LOCAL)

- **Servidor**: `localhost`
- **Puerto**: `1883`
- **Conectar automáticamente**: ✅
- **Usar sesión limpia**: ✅
- **TLS**: ❌ No habilitado

### 10.4 Configuración del nodo MQTT OUT

- **Servidor**: `MOSQUITO MQTT LOCAL`
- **Tópico**: `nodered/uplink/processed`
- **QoS**: por defecto (sin especificar)
- **Retener**: ❌ (no marcado)

### 10.5 Función en el nodo “Desanidar y aplanar”

```javascript
var payload = msg.payload;
var objectData = payload.object || {};
delete payload.object;

for (var key in objectData) {
    payload[key] = objectData[key];
}

msg.payload = payload;
return msg;
```

### 10.6 Ejemplo de salida observada en nodo debug

```json
{
  "applicationID": "1",
  "applicationName": "app-em300",
  "deviceName": "em300-ZDL-Id1-Zone2",
  "deviceProfileName": "profile-EM300-SLD/ZDL-868M",
  "deviceProfileID": "721d249d-8d14-4d2c-9932-0cf06e7b30a3",
  "devEUI": "24e124136d446138",
  "rxInfo": [...],
  "txInfo": {...},
  "adr": true,
  "fCnt": 1909,
  "fPort": 85,
  "data": "A2f0AARoXAUAAA==",
  "_leakage_status": "normal",
  "humidity": 46,
  "leakage_status": "normal",
  "temperature": 24.4
}
```

---

## 11. Validación de publicación MQTT OUT (resultado final esperado)

Al ejecutar:

```bash
mosquitto_sub -t 'nodered/uplink/processed' -v
```

Se observará que cada mensaje publicado por Node-RED contiene ahora el JSON desanidado, ideal para ser capturado directamente por Telegraf:

```json
nodered/uplink/processed {
  "applicationID": "1",
  "deviceName": "em300-SLD-Id1-Zone1",
  "humidity": 63.5,
  "temperature": 17.4,
  "leakage_status": "normal",
  "_leakage_status": "normal",
  "tags": {
    "Marca": "Milesight",
    "Modelo": "EM300-SLD-868M",
    "Ubicacion": "Edificio"
  }
}
```

Esto confirma la funcionalidad completa del flujo desde entrada MQTT hasta salida procesada para InfluxDB.

---

## 12. Consideraciones operativas y de arquitectura

### 12.1 Separación y coexistencia de flujos MQTT

Actualmente, el host núcleo (donde reside ChirpStack y Mosquitto) maneja:

- El flujo MQTT original: `application/+/device/+/event/up`
- El flujo adaptado generado por Node-RED: `nodered/uplink/processed`

Esta coexistencia es **necesaria y adecuada**:

- Permite mantener trazabilidad del mensaje original.
- Ofrece un canal limpio y optimizado para consumo directo por Telegraf.
- Garantiza compatibilidad futura con otros servicios o usos del tópico original.

### 12.2 Node-RED como intermediario lógico

Node-RED no reemplaza ningún componente. Su rol es intermediar entre recepción y destino, cumpliendo funciones como:

- Desanidar campos de payload.
- Homogeneizar estructura.
- Publicar mensajes adaptados en un nuevo canal MQTT.

### 12.3 Impacto en rendimiento

En un entorno como el actual, Node-RED:

- Tiene un impacto de CPU/RAM **muy bajo**.
- Introduce **latencia mínima (milisegundos)**.
- Solo requiere cuidado si el volumen de mensajes por segundo fuera muy alto (miles por segundo).

En condiciones normales y con sensores enviando cada pocos minutos, su impacto es **negligible**.

### 12.4 Recomendaciones clave

- Asegurarse de que Telegraf solo consuma del canal limpio (`nodered/uplink/processed`).
- No redirigir mensajes de vuelta al tópico original (evitar bucles MQTT).
- Usar los tópicos de forma clara y con nombres semánticos.

---

## 13. Node-RED como servicio persistente

### 13.1 ¿Es Node-RED un servicio?

Node-RED no lo es por defecto, pero puede convertirse en uno fácilmente usando `pm2`, que permite:

- Ejecutarlo en segundo plano.
- Que inicie automáticamente con el sistema.
- Conservar la configuración permanentemente.

### 13.2 Activación como servicio

```bash
sudo npm install -g pm2
pm2 start $(which node-red)
pm2 save
pm2 startup
```

### 13.3 ¿Dónde se guarda la configuración y flujos?

Por defecto:

- En el directorio del usuario que ejecuta Node-RED:

```bash
~/.node-red/flows_<hostname>.json
```

- Este archivo contiene todos los flujos y nodos configurados.
- Se actualiza cada vez que haces `Deploy` desde la interfaz.

**Importante**: PM2 garantizará que, incluso tras reinicio del sistema, el servicio de Node-RED vuelva a levantarse con todos los flujos intactos.

---

## 14. Recomendaciones de estructura óptima para visualización y almacenamiento

A continuación se detallan una serie de mejoras recomendadas para dejar los datos publicados por Node-RED en una estructura ideal, tanto para ser almacenados en InfluxDB como para ser visualizados eficientemente en Grafana.

Estas recomendaciones buscan minimizar los ajustes manuales en paneles y facilitar el uso de variables dinámicas en visualizaciones y alertas.

### 14.1 Desanidar completamente `object` y `tags`

Node-RED debe extraer tanto el contenido de `payload.object` como de `payload.tags` y moverlo al nivel raíz del `msg.payload`, eliminando la anidación.

Ejemplo de transformación:

```json
"tags": {
  "Marca": "Milesight",
  "Ubicacion": "Edificio"
}
```

➡️ Se convierte en:

```json
"Marca": "Milesight",
"Ubicacion": "Edificio"
```

Esto permite que Telegraf los registre como `tag_keys` fácilmente.

### 14.2 Usar `deviceName` como identificador humano en Grafana

Evitar usar `devEUI` o `topic` como selector en paneles. Se recomienda siempre incluir `deviceName` como tag y variable base en dashboards de Grafana.

Esto facilita la legibilidad y navegación por múltiples sensores.

### 14.3 Renombrar campos con unidades explícitas

Para mejorar la comprensión y claridad de datos:

- `temperature` → `temperature_c`
- `humidity` → `humidity_pct`

Esto ayuda a que en Grafana no sea necesario adivinar o aclarar unidades manualmente.

### 14.4 Eliminar duplicados innecesarios

Asegurarse de que no se generen campos duplicados como:

- `leakage_status` y `_leakage_status`

Seleccionar un único nombre y mantener consistencia en todos los nodos.

### 14.5 Garantizar presencia de campos opcionales como `battery`, `status`, etc.

Incluso si el sensor no envía un valor, se recomienda agregar explícitamente estos campos con `null` si no existen, para mantener coherencia de estructura:

```javascript
if (decoded.battery !== undefined) result.battery = decoded.battery;
else result.battery = null;
```

Esto facilita el uso de paneles `stat`, `gauge`, `threshold` o `last()` sin errores por ausencia de campo.

### 14.6 Considerar `history` o datos acumulados

Algunos nodos (como EM300-ZDL) pueden enviar bloques `history` cuando se reconectan. Este campo suele ser un array de puntos con timestamp, temperatura, humedad, y estado.

Ejemplo:

```json
"history": [
  { "timestamp": 1665561758, "temperature": 27.2, "humidity": 46.5, "leakage_status": "leak" }
]
```

Se recomienda separar estos datos en otro flujo o función de Node-RED si se desea almacenarlos, o descartarlos si no se necesitan para análisis.

---

## 15. Validación transversal con múltiples dispositivos

Se ha verificado el correcto funcionamiento de la lógica de desanidado, renombrado, estandarización y limpieza de datos aplicados en Node-RED utilizando datos reales de sensores EM300-TH, EM310, EM300-ZDL y EM300-SLD.

### Dispositivos y ejemplos analizados
- `em300-Id1-Zone1` a `Zone4` con sensores de temperatura y humedad.
- `em310-UDL` sin `object`, para validar el fallback seguro.
- `em300-ZDL-Id1-Zone2`, `em300-SLD-Id1-Zone1` con campos de `leakage_status` y `tags`.

### Comprobaciones realizadas:
- ✅ Desanidado correcto de `object` y `tags`.
- ✅ Renombrado a `temperature_c`, `humidity_pct`.
- ✅ Inclusión de campos opcionales con `null` cuando no están presentes.
- ✅ Eliminación de duplicados como `_leakage_status`.
- ✅ Formato consistente y limpio en todos los mensajes MQTT publicados a `nodered/uplink/processed`.

Esta validación asegura que el flujo Node-RED es compatible con toda la variedad de sensores desplegados, y que está listo para ser consumido sin ajustes adicionales por Telegraf e InfluxDB.


## 16. Configuración final del input MQTT en Telegraf

Una vez transformados y desanidados los datos correctamente en Node-RED, se recomienda adaptar la configuración de entrada MQTT en Telegraf para consumir exclusivamente del canal limpio y estructurado.

### Contenido recomendado para `lorawan.conf` en el host InfluxDB:

```toml
[[inputs.mqtt_consumer]]
  servers = ["tcp://192.168.212.231:1883"]
  topics = ["nodered/uplink/processed"]
  client_id = "telegraf-nodered-app"
  data_format = "json"
  name_override = "uplink_lora"

  tag_keys = [
    "devEUI",
    "deviceName",
    "applicationName",
    "Marca",
    "Modelo",
    "Ubicacion"
  ]

  qos = 0
  persistent_session = true

[[outputs.influxdb_v2]]
  urls = ["http://localhost:8086"]
  token = "<REEMPLAZAR_CON_TOKEN_REAL>"
  organization = "IoT-Lora"
  bucket = "iot-datos"

Notas importantes:

    Si se añaden nuevos tags personalizados desde ChirpStack (como Planta, Zona, TipoEquipo), será necesario añadir esos nombres también a la lista tag_keys[] en este archivo.

    Es recomendable mantener esta lista sincronizada con la estructura que genera Node-RED tras desanidar tags.

    Puedes automatizar esta lógica en el flujo de Node-RED si se desea mantener un diseño dinámico.

## Reinicio y validación del servicio Telegraf

Una vez guardado el archivo lorawan.conf, es necesario reiniciar Telegraf para aplicar los cambios:
### 1. Reiniciar Telegraf

sudo systemctl restart telegraf

2. Verificar el estado del servicio

sudo systemctl status telegraf

Debe indicar que el servicio está “activo (running)” sin errores.
3. Verificar si está recibiendo datos

journalctl -u telegraf -f

Esto muestra en tiempo real los mensajes que Telegraf está procesando. Si aparecen tramas del tópico nodered/uplink/processed, la integración ha sido exitosa.

✅ Consulta básica para verificar datos entrantes

Ejecuta en el host de InfluxDB:

influx query '
from(bucket: "iot-datos")
  |> range(start: -10m)
  |> filter(fn: (r) => r._measurement == "uplink_lora")
  |> limit(n: 5)
'

🔍 ¿Qué deberías ver?

    Al menos 5 resultados con campos como:

        humidity_pct

        temperature_c

        leakage_status, battery, etc.

    Y tags como:

        deviceName

        Marca, Modelo, Ubicacion

