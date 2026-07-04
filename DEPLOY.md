# MediTrack MQTT Broker (Mosquitto) — capa FOG

Broker MQTT para telemetría: **edge → Mosquitto → devices**.

## Dónde desplegar

**Recomendado:** misma VM de GCP donde corre `meditrack-devices` (`34.176.0.154`).

| Cliente | URL MQTT |
|---------|----------|
| **devices** (en la VM) | `tcp://localhost:1883` |
| **edge** (casa / Mac) | `tcp://34.176.0.154:1883` |

## 1. Subir archivos a la VM

Desde tu Mac (en la carpeta del repo):

```bash
scp -r meditrack-mqtt-broker ubuntu@34.176.0.154:~/meditrack-mqtt-broker
```

## 2. En la VM — levantar Mosquitto

```bash
ssh ubuntu@34.176.0.154
cd ~/meditrack-mqtt-broker
docker compose up -d
docker compose ps
docker logs meditrack-mosquitto --tail 20
```

Debe escuchar en el puerto **1883**.

## 3. Firewall GCP (obligatorio para el edge remoto)

En Google Cloud Console → VPC → Firewall:

| Campo | Valor |
|-------|--------|
| Puerto | `1883` |
| Protocolo | TCP |
| Origen | `0.0.0.0/0` (demo) o IP del edge si la conoces |
| Destino | VM `34.176.0.154` |

Sin esto, el edge en tu Mac **no** podrá publicar.

## 4. Probar desde la VM

Suscriptor:

```bash
docker run --rm eclipse-mosquitto:2 mosquitto_sub -h 34.176.0.154 -p 1883 -t 'meditrack/devices/telemetry' -v
```

Publicador de prueba:

```bash
docker run --rm eclipse-mosquitto:2 mosquitto_pub -h 34.176.0.154 -p 1883 -t 'meditrack/devices/telemetry' -m '{"deviceId":1,"heartRate":72}'
```

## 5. `.env` exacto para el encargado de devices (en la VM)

Si devices corre con perfil `prod`:

```env
SPRING_PROFILES_ACTIVE=prod

MYSQL_CONNECTION=jdbc:mysql://<HOST_MYSQL>:3306/meditrack_devices?useSSL=false&serverTimezone=UTC&allowPublicKeyRetrieval=true
MYSQL_USER=<usuario>
MYSQL_PASSWORD=<password>

CLINICAL_SERVICE_URL=https://meditrack-clinical-hph0avgbe2ejbyew.eastus-01.azurewebsites.net
CLINICAL_THRESHOLD_CACHE_TTL_SECONDS=120

MQTT_ENABLED=true
MQTT_BROKER_URL=tcp://localhost:1883
MQTT_TELEMETRY_TOPIC=meditrack/devices/telemetry
MQTT_CLIENT_ID=meditrack-devices
```

Si devices corre **sin** perfil `prod`:

```env
DB_URL=jdbc:mysql://<HOST_MYSQL>:3306/meditrack_devices?useSSL=false&serverTimezone=UTC&allowPublicKeyRetrieval=true
DB_USERNAME=<usuario>
DB_PASSWORD=<password>

CLINICAL_SERVICE_URL=https://meditrack-clinical-hph0avgbe2ejbyew.eastus-01.azurewebsites.net

MQTT_ENABLED=true
MQTT_BROKER_URL=tcp://localhost:1883
MQTT_TELEMETRY_TOPIC=meditrack/devices/telemetry
MQTT_CLIENT_ID=meditrack-devices
```

## 6. Tu `.env` del edge (Mac / sitio del parche)

```env
EDGE_PORT=5050
BACKEND_BASE_URL=https://meditrack-gateway.duckdns.org
SYNC_TRANSPORT=mqtt
MQTT_ENABLED=true
MQTT_BROKER_HOST=34.176.0.154
MQTT_BROKER_PORT=1883
MQTT_TELEMETRY_TOPIC=meditrack/devices/telemetry
MQTT_CLIENT_ID=meditrack-edge
```

## 7. Desarrollo local (opcional)

Con **Docker Desktop** encendido en tu Mac:

```bash
cd meditrack-edge
docker compose -f docker-compose.mqtt.yaml up -d
```

Entonces devices local usaría `tcp://localhost:1883` y edge `MQTT_BROKER_HOST=localhost`.

## Tópico acordado

```text
meditrack/devices/telemetry
```
