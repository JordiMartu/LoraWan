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


# Implementacion_NODE_RED_Argumentario_y_Guia.md

... [contenido previo intacto] ...

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
