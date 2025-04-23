
# 📌 PUNTO DE CONTROL: Estado de configuración del sistema IoT LoRaWAN

Este documento tiene como objetivo mantener una trazabilidad clara de las configuraciones clave del sistema, incluyendo ajustes en los nodos, ChirpStack (NS, AS), Telegraf, InfluxDB y Grafana. Cada vez que se modifica un componente, se debe reflejar aquí con fecha y justificación.

---

## 🔁 Control de versiones de archivos críticos

### 🗂️ Host: ChirpStack (NS + AS)

#### ✅ `chirpstack-application-server.toml`
```toml
[application_server]
bind="0.0.0.0:8080"
public_host="localhost:8080"

[postgresql]
dsn="postgres://appserver:12341234@localhost/chirpstack_as?sslmode=disable"
automigrate=true

[redis]
url="redis://localhost:6379"

[application_server.integration.mqtt]
server="tcp://localhost:1883"
username=""
password=""

[join_server.default]
server="http://localhost:8003"

[application_server.external_api]
bind="0.0.0.0:8080"
tls_cert=""
tls_key=""
jwt_secret="2J3wkXWEOvDXPTAs8M9xmKfc8vOqdRlmwvv7jf3Wa00="
```

#### ✅ `chirpstack-network-server.toml`
```toml
[postgresql]
dsn="postgres://networkserver:12341234@localhost/chirpstack_ns?sslmode=disable"

[redis]
url="redis://localhost:6379"

[network_server]
net_id="000000"
  [network_server.band]
  name="EU868"

  [network_server.network_settings]

    [[network_server.network_settings.extra_channels]]
    frequency=867100000
    min_dr=0
    max_dr=5
    ...

  [network_server.network_settings.class_b]
  ping_slot_dr=0
  ping_slot_frequency=0

  [network_server.api]
  bind="0.0.0.0:8000"

  [network_server.gateway.backend]
  type="mqtt"
    [network_server.gateway.backend.mqtt]
    server="tcp://localhost:1883"
    username=""
    password=""

[metrics]
timezone="Local"

[join_server.default]
server="http://localhost:8003"
```

#### ✅ `chirpstack-gateway-bridge.toml`
```toml
[backend]
type="semtech_udp"
  [backend.semtech_udp]
  udp_bind = "0.0.0.0:1700"

[integration]
marshaler="json"
  [integration.mqtt]
  event_topic_template="gateway/{{ .GatewayID }}/event/{{ .EventType }}"
  command_topic_template="gateway/{{ .GatewayID }}/command/#"
    [integration.mqtt.auth]
    type="generic"
      [integration.mqtt.auth.generic]
      server="tcp://127.0.0.1:1883"
      username=""
      password=""
```

#### ✅ `mosquitto.conf`
```conf
pid_file /run/mosquitto/mosquitto.pid
persistence true
persistence_location /var/lib/mosquitto/
log_dest file /var/log/mosquitto/mosquitto.log
include_dir /etc/mosquitto/conf.d
listener 1883
allow_anonymous true
```

### 🗂️ Host: InfluxDB + Telegraf + Grafana

#### ✅ `lorawan.conf` (Telegraf)
```toml
[[inputs.mqtt_consumer]]
  servers = ["tcp://192.168.212.231:1883"]
  topics = ["application/+/device/+/event/up"]
  client_id = "telegraf-lorawan-app"
  data_format = "json"
  name_override = "uplink_lora"
  tag_keys = [
    "devEUI",
    "deviceName",
    "applicationName",
    "tags.Marca",
    "tags.Modelo",
    "tags.Ubicacion"
  ]
  qos = 0
  persistent_session = true

[[outputs.influxdb_v2]]
  urls = ["http://localhost:8086"]
  token = "x5StC8WynI0zvcS29z-Z4tR6Cavo7G-yigrbg4ioVzB_LwaAo_QczriaEDd1_XBcDm_yq36QNOcKUA2s-1TYPA=="
  organization = "IoT-Lora"
  bucket = "iot-datos"
```

#### ✅ `telegraf.conf` (solo líneas activas)
```toml
[global_tags]
[agent]
  interval = "10s"
  round_interval = true
  metric_batch_size = 1000
  metric_buffer_limit = 10000
  collection_jitter = "0s"
  flush_interval = "10s"
  flush_jitter = "0s"
  precision = "0s"
```

#### ✅ `decoder_milesight_em300.js` (Decoder personalizado EM300-TH)
```javascript
// Decoder oficial Milesight para EM300-TH (temperatura, humedad, batería)
// Compatible con ChirpStack v3 y v4

function Decode(fPort, bytes) {
    return milesightDeviceDecode(bytes);
}

function milesightDeviceDecode(bytes) {
    var decoded = {};

    for (var i = 0; i < bytes.length; ) {
        var channel_id = bytes[i++];
        var channel_type = bytes[i++];

        // BATTERY
        if (channel_id === 0x01 && channel_type === 0x75) {
            decoded.battery = bytes[i];
            i += 1;
        }
        // TEMPERATURE
        else if (channel_id === 0x03 && channel_type === 0x67) {
            decoded.temperature = readInt16LE(bytes.slice(i, i + 2)) / 10;
            i += 2;
        }
        // HUMIDITY
        else if (channel_id === 0x04 && channel_type === 0x68) {
            decoded.humidity = bytes[i] / 2;
            i += 1;
        }
        // TEMPERATURE & HUMIDITY HISTORY
        else if (channel_id === 0x20 && channel_type === 0xce) {
            var point = {};
            point.timestamp = readUInt32LE(bytes.slice(i, i + 4));
            point.temperature = readInt16LE(bytes.slice(i + 4, i + 6)) / 10;
            point.humidity = bytes[i + 6] / 2;

            decoded.history = decoded.history || [];
            decoded.history.push(point);
            i += 8;
        } else {
            break;
        }
    }

    return decoded;
}

/* Utilities for reading integers from byte arrays */
function readUInt16LE(bytes) {
    return (bytes[1] << 8) + bytes[0];
}

function readInt16LE(bytes) {
    var ref = readUInt16LE(bytes);
    return ref > 0x7fff ? ref - 0x10000 : ref;
}

function readUInt32LE(bytes) {
    return (bytes[3] << 24) + (bytes[2] << 16) + (bytes[1] << 8) + bytes[0];
}
```

---

FECHA DE ULTIMA ACTUALIZACION: 24/04/2025 0:39
---
