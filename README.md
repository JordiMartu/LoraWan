# LoraWan
Servicio, Deplieque y Documentación de implementación de host núcleo IoT + host DB y Visualizador datos

# Nodo Núcleo LoRaWAN - ChirpStack v3 sobre Debian 11

Este sistema actúa como el núcleo central de una red LoRaWAN privada, orientada inicialmente a entornos urbanos, pero preparado para ser adaptado a otras aplicaciones. Está desplegado sobre Debian 11 y contiene toda la infraestructura necesaria para recibir, procesar y administrar datos de sensores LoRa a través del framework **ChirpStack v3**.

---

## 📦 Documentos - Guias - Notas  datos de instalación:

- /home/open/chirpstackv3/docs

📘 Gateway_ChirpStack_Lora_conceptos
🧾 Guía_Alta_Nodos_Em300
📘 Manual de Administración del Servidor ChirpStack
🧾 Notas de Instalación - ChirpStack IoT LoRaWAN

---

## 📦 Componentes instalados

- ChirpStack Network Server (NS)
- ChirpStack Application Server (AS)
- ChirpStack Gateway Bridge (GWB)
- Mosquitto (MQTT broker)
- PostgreSQL (almacenamiento)
- Redis (cache de sesión)

---

## 📂 Estructura del proyecto

```
chirpstackv3/
├── backups/           → Backups de configuración y base de datos
├── configs/           → Plantillas de configuración (si se añaden)
├── docs/              → Manuales, guías técnicas y documentación interna
├── gatewaybridge/     → Enlace simbólico al archivo .toml del Gateway Bridge
├── logs/              → Registros personalizados del sistema o scripts
├── scripts/           → Scripts operativos para verificación y arranque
├── README.md          → Este archivo
```

---

## ⚙️ Scripts disponibles

- `chirpstack_check.sh`  
  → Verifica el estado de todos los servicios, puertos, migraciones y acceso a interfaz web.

- `chirpstack_servicios.sh`  
  → Script de arranque limpio, inicia servicios de forma ordenada con espera entre ellos.

---

## 🔐 Backups incluidos

Ubicados en `backups/configs/`:
- Archivos `.toml` de ChirpStack
- Configuración de Mosquitto
- Archivos PostgreSQL (`pg_hba.conf`, `postgresql.conf`)

La base de datos puede ser exportada manualmente con:

```bash
pg_dump -U appserver chirpstack_as > backups/db/chirpstack_as.sql
pg_dump -U networkserver chirpstack_ns > backups/db/chirpstack_ns.sql
```

---

## 🛰️ Estado actual

- Gateway UG65 conectado correctamente (modo Semtech UDP)
- Dos nodos EM300 operativos y enviando datos vía OTAA
- MQTT activo y tramas visibles vía `mosquitto_sub`
- ChirpStack accesible vía Web UI en `http://localhost:8080`
- Scripts y servicios operativos y verificados tras reinicio

- Integración externa con InfluxDB + Grafana - Realizado
- Crear decodificadores de payload en ChirpStack - Realizado

---

## 📌 Próximos pasos sugeridos

- Necesitamos capturar en influx/grafana los valores JSON con tags, datos de sensores (open,clore,leak,etc) y no es posible sin Implementar NODE-RED.
- Establecer un mecanismo de backup automático
- Activar monitorización (con `monit` u otra solución)

---

## 👤 Responsable

- Proyecto LoRaWAN Urbano – Abril 2025
- Nodo administrado por: `open@loraIOT`
