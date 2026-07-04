# cafelab-mqtt-broker

Broker Mosquitto local para telemetria CafeLab:

```text
edge-clean -> MQTT -> mqtt-bridge -> API Gateway
```

## Despliegue rapido

```powershell
docker compose up -d
docker compose ps
docker logs -f cafelab-mosquitto
```

Guia local completa: ver [MQTT_BROKER_LOCAL_GUIA.md](./MQTT_BROKER_LOCAL_GUIA.md).

## Topico MQTT

```text
cafelab/iot/telemetry
```
