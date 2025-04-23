
# 🧩 Patrón de trabajo para nuevos nodos y decodificadores

## 📂 1. Estructura de datos esperada desde ChirpStack

Al recibir los datos desde los dispositivos, ChirpStack (Application Server) genera un mensaje JSON que puede tener dos estructuras importantes:

- Campos de metadatos generales (fuera de `object`) como `rxInfo`, `txInfo`, `devEUI`, `fPort`, `fCnt`, etc.
- Datos de sensores decodificados ubicados dentro del campo `object`, que es generado por el **payload decoder JS** configurado en el perfil del dispositivo.

Ejemplo:
```json
{
  "deviceName": "em300-Id1-Zone1",
  "rxInfo": [...],
  "object": {
    "temperature": 24.2,
    "humidity": 60.0
  }
}
```

## ⚙️ 2. Configuración de Telegraf (`mqtt_consumer`)

Para asegurar que solo los datos decodificados lleguen a InfluxDB:

```toml
[[inputs.mqtt_consumer]]
  data_format = "json"
  json_query = "object"          # ✔ Captura solo el campo object
  json_string_fields = ["magnet_status", "humidity", "temperature", ...]  # ✔ Evita problemas de tipos de datos
```

- `json_query = "object"` obliga a que Telegraf solo procese el contenido del campo `object`, ignorando el resto.
- Si se desea que `rxInfo.rssi` o `txInfo.frequency` también lleguen a InfluxDB, entonces se debe quitar `json_query` **o** configurar un `processor` personalizado para extraer esos valores desde el JSON completo.

## 🔢 3. InfluxDB: resultado de la ingesta

Los datos que efectivamente aparecen como campos (`_field`) en Influx son aquellos:
- Presentes en el `object` **o**
- Extraídos por Telegraf de manera explícita (dependiendo del plugin `mqtt_consumer`)

Ejemplo en Influx:
```bash
influx query '
import "influxdata/influxdb/schema"
schema.fieldKeys(bucket: "iot-datos")'
```

Esto debe incluir `object_temperature`, `object_humidity`, y en algunos casos también `rxInfo_0_rssi`, `txInfo_frequency`, etc.

## 🪤 Recomendación de patrón para nuevos nodos

1. **Decoder JS claro y validado**: debe retornar un objeto con claves sencillas (`battery`, `humidity`, `temperature`, etc.)
2. **Mantener consistencia en los nombres de campos y tipos** (temperatura en °C, humedad en %, `magnet_status` como texto)
3. **Validar mensajes MQTT desde ChirpStack usando `mosquitto_sub`** antes de conectar Telegraf
4. **Test de ingesta en Telegraf con `--test --debug`** para asegurar que se parsea correctamente
5. **Query de validación en Influx** para confirmar que aparecen las métricas
6. **Ajuste de dashboards en Grafana** según los nuevos `deviceName` o `field`
7. **Registrar configuración del nodo en `Punto De Control.md`** junto con la versión del decoder, perfil, y campos esperados.

---

Este patrón asegura coherencia y trazabilidad a medida que se integran nuevos sensores al sistema.
