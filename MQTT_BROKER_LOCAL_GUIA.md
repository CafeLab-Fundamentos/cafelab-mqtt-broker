# Guia local MQTT CafeLab

## 1. Arquitectura local

Esta demo coloca Mosquitto entre el edge y el API Gateway:

```text
Dispositivo IoT -> edge-clean -> Mosquitto MQTT -> mqtt-bridge -> API Gateway -> backend IoT
```

El edge sigue respondiendo rapido al dispositivo. En modo MQTT, el edge ya no hace el `POST /api/v1/telemetry-records` directamente; publica la lectura en el topico `cafelab/iot/telemetry`. El `mqtt-bridge` consume ese mensaje y lo envia al API Gateway.

Endpoint final:

```text
POST https://cafelab-api-gateway-gnfua0csgsbud3eh.canadacentral-01.azurewebsites.net/api/v1/telemetry-records
```

Body esperado:

```json
{
  "coffeeLotId": 32,
  "temperature": 25.5,
  "humidity": 60.2,
  "timestamp": "2026-07-04T18:47:11.388Z"
}
```

## 2. Levantar Mosquitto

```powershell
cd C:\dev\FUNDAMENTOS\mqtt-broker
docker compose up -d
```

Verificar contenedor:

```powershell
docker compose ps
docker logs -f cafelab-mosquitto
```

Debe aparecer un contenedor llamado `cafelab-mosquitto` escuchando en el puerto `1883`.

## 3. Probar pub/sub manual

Terminal 1: suscribirse al topico:

```powershell
cd C:\dev\FUNDAMENTOS\mqtt-broker
docker exec -it cafelab-mosquitto mosquitto_sub -h localhost -p 1883 -t cafelab/iot/telemetry -v
```

Terminal 2: publicar una lectura de prueba:

```powershell
docker exec cafelab-mosquitto mosquitto_pub -h localhost -p 1883 -t cafelab/iot/telemetry -m '{"coffeeLotId":32,"temperature":25.5,"humidity":60.2,"timestamp":"2026-07-04T18:47:11.388Z"}'
```

Salida esperada en la terminal suscrita:

```text
cafelab/iot/telemetry {"coffeeLotId":32,"temperature":25.5,"humidity":60.2,"timestamp":"2026-07-04T18:47:11.388Z"}
```

## 4. Instalar dependencias del edge

```powershell
cd C:\dev\FUNDAMENTOS\edge-clean
.\.venv\Scripts\activate
pip install -r requirements.txt
```

Si no usas la venv existente:

```powershell
cd C:\dev\FUNDAMENTOS\edge-clean
py -m venv .venv
.\.venv\Scripts\activate
pip install -r requirements.txt
```

## 5. Configurar edge-clean en modo MQTT

Editar `C:\dev\FUNDAMENTOS\edge-clean\.env` y usar:

```env
BACKEND_BASE_URL=https://cafelab-api-gateway-gnfua0csgsbud3eh.canadacentral-01.azurewebsites.net
SYNC_TRANSPORT=mqtt
MQTT_BROKER_HOST=localhost
MQTT_BROKER_PORT=1883
MQTT_TELEMETRY_TOPIC=cafelab/iot/telemetry
MQTT_CLIENT_ID=cafelab-edge-local
COFFEE_LOT_ID=32
```

Notas:

- `SYNC_TRANSPORT=http` mantiene el flujo HTTP directo anterior.
- `SYNC_TRANSPORT=mqtt` publica las lecturas en Mosquitto.
- `COFFEE_LOT_ID=32` se usa como fallback si el dispositivo no tiene lote asignado localmente.
- El payload MQTT no incluye `deviceId` hacia el gateway.

Ejecutar edge:

```powershell
cd C:\dev\FUNDAMENTOS\edge-clean
.\.venv\Scripts\activate
python .\app.py
```

Cuando llegue una lectura del dispositivo al edge, debes ver logs parecidos a:

```text
Publishing telemetry to MQTT topic cafelab/iot/telemetry: {'coffeeLotId': 32, ...}
Synced 1 reading(s) via mqtt
```

## 6. Ejecutar mqtt-bridge

Primera vez:

```powershell
cd C:\dev\FUNDAMENTOS\mqtt-bridge
py -m venv .venv
.\.venv\Scripts\activate
pip install -r requirements.txt
Copy-Item .env.example .env
```

Editar `C:\dev\FUNDAMENTOS\mqtt-bridge\.env` si necesitas token o cambiar usuario:

```env
MQTT_BROKER_HOST=localhost
MQTT_BROKER_PORT=1883
MQTT_TOPIC=cafelab/iot/telemetry
MQTT_CLIENT_ID=cafelab-mqtt-bridge-local
API_GATEWAY_BASE_URL=https://cafelab-api-gateway-gnfua0csgsbud3eh.canadacentral-01.azurewebsites.net
TELEMETRY_ENDPOINT=/api/v1/telemetry-records
X_USER_ID=8
AUTH_TOKEN=
HTTP_TIMEOUT_SECONDS=10
HTTP_MAX_RETRIES=3
```

Ejecutar bridge:

```powershell
cd C:\dev\FUNDAMENTOS\mqtt-bridge
.\.venv\Scripts\activate
python .\mqtt_bridge.py
```

Logs esperados:

```text
Starting CafeLab MQTT bridge
Connected to MQTT broker localhost:1883
Subscribed to MQTT topic cafelab/iot/telemetry
MQTT message received from cafelab/iot/telemetry: {...}
Body to API Gateway: {'coffeeLotId': 32, 'temperature': 25.5, ...}
API Gateway HTTP 201
API Gateway response: {...}
```

Si el gateway exige Bearer token, pega el token sin la palabra `Bearer`:

```env
AUTH_TOKEN=eyJhbGciOi...
```

El bridge enviara:

```text
Authorization: Bearer <token>
```

Si el gateway usa `X-User-Id`, dejalo asi:

```env
X_USER_ID=8
```

## 7. Probar todo sin dispositivo real

Con broker y bridge encendidos, publica manualmente:

```powershell
docker exec cafelab-mosquitto mosquitto_pub -h localhost -p 1883 -t cafelab/iot/telemetry -m '{"coffeeLotId":32,"temperature":25.5,"humidity":60.2,"timestamp":"2026-07-04T18:47:11.388Z"}'
```

El bridge debe recibir el mensaje y llamar al API Gateway. Si responde `201`, el puente funciona.

## 8. Probar con el dispositivo IoT real

1. Levanta Mosquitto.
2. Levanta `mqtt-bridge`.
3. Configura `edge-clean` con `SYNC_TRANSPORT=mqtt`.
4. Ejecuta `python .\app.py` en el edge.
5. Enciende el ESP32 o dispositivo IoT que envia lecturas a `POST /api/v1/edge/readings`.
6. Verifica:
   - El edge responde `201` al dispositivo.
   - El edge publica MQTT.
   - `mosquitto_sub` muestra el JSON.
   - `mqtt-bridge` hace POST al API Gateway.
   - Swagger/Postman muestra las lecturas por lote.

## 9. Verificar lecturas en backend

En Swagger o Postman:

```http
GET https://cafelab-api-gateway-gnfua0csgsbud3eh.canadacentral-01.azurewebsites.net/api/v1/telemetry-records/coffee-lot/32
```

Si requiere headers, usa los mismos que el gateway acepte:

```text
X-User-Id: 8
Authorization: Bearer <token>
Accept: application/json
```

Debes ver registros con `coffeeLotId`, `temperature`, `humidity` y `timestamp`.

## 10. Detener componentes

Broker:

```powershell
cd C:\dev\FUNDAMENTOS\mqtt-broker
docker compose down
```

Bridge:

```text
Ctrl + C
```

Edge:

```text
Ctrl + C
```

Para volver al modo anterior del edge:

```env
SYNC_TRANSPORT=http
```

## 11. Evidencias para el informe

Toma capturas de:

1. `docker compose ps` mostrando `cafelab-mosquitto` corriendo.
2. `docker logs -f cafelab-mosquitto` mostrando conexiones/publicaciones.
3. Terminal con `mosquitto_sub` recibiendo mensajes del topico `cafelab/iot/telemetry`.
4. Logs de `edge-clean` mostrando `Publishing telemetry to MQTT topic`.
5. Logs de `mqtt-bridge` mostrando:
   - mensaje MQTT recibido,
   - body enviado al API Gateway,
   - codigo HTTP `201`.
6. Swagger/Postman mostrando `GET /api/v1/telemetry-records/coffee-lot/32` con lecturas nuevas.

## 12. Explicacion para el profesor

El broker Mosquitto funciona como intermediario asincrono entre el edge y el API Gateway. El edge ya no depende de que el gateway responda inmediatamente para cada lectura; publica el mensaje al broker y continua atendiendo al dispositivo. El `mqtt-bridge` consume esos mensajes y los reenvia al gateway. Esto desacopla la recepcion de lecturas IoT del procesamiento HTTP del backend, lo cual ayuda al atributo de calidad de rendimiento porque absorbe picos y evita que muchos dispositivos golpeen directamente el API Gateway al mismo tiempo.

## 13. Errores comunes

### `docker` no se reconoce

Abre Docker Desktop y vuelve a ejecutar el comando.

### `Connection refused` al broker

Verifica:

```powershell
docker compose ps
docker logs cafelab-mosquitto
```

Asegurate de que `MQTT_BROKER_HOST=localhost` si todo corre local.

### `No module named paho`

Instala dependencias:

```powershell
pip install -r requirements.txt
```

### El bridge recibe MQTT pero el gateway responde 401/403

Configura `AUTH_TOKEN` o `X_USER_ID` en `C:\dev\FUNDAMENTOS\mqtt-bridge\.env`.

### El gateway responde 400

Revisa que el JSON tenga solo estos campos requeridos:

```json
{
  "coffeeLotId": 32,
  "temperature": 25.5,
  "humidity": 60.2,
  "timestamp": "2026-07-04T18:47:11.388Z"
}
```

No envies `deviceId` al endpoint `/api/v1/telemetry-records`.
