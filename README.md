# CafeLab MQTT Broker

Broker Mosquitto local para desacoplar la ingesta IoT de CafeLab.

```text
ESP32 / simulator -> edge-clean -> Mosquitto -> mqtt-bridge -> API Gateway Azure -> IoT Monitoring
```

El broker recibe lecturas publicadas por `edge-clean` en el topico `cafelab/iot/telemetry`. Luego `mqtt-bridge` consume esos mensajes y los reenvia al API Gateway desplegado.

## Requisitos

- Docker Desktop o Docker Engine.
- Puerto local `1883` disponible.
- Repos relacionados en la misma maquina:
  - `C:\dev\FUNDAMENTOS\edge-clean`
  - `C:\dev\FUNDAMENTOS\mqtt-bridge`

## Levantar el broker

```powershell
cd C:\dev\FUNDAMENTOS\mqtt-broker
docker compose up -d
docker compose ps
```

Ver logs:

```powershell
docker logs -f cafelab-mosquitto
```

## Probar publicacion y consumo

Suscribirse al topico:

```powershell
docker exec -it cafelab-mosquitto mosquitto_sub -h localhost -p 1883 -t cafelab/iot/telemetry -v
```

Publicar una lectura de prueba:

```powershell
docker exec -it cafelab-mosquitto mosquitto_pub -h localhost -p 1883 -t cafelab/iot/telemetry -m '{"coffeeLotId":32,"temperature":25.5,"humidity":60.2,"timestamp":"2026-07-04T18:47:11"}'
```

## Apagar el broker

Apagar contenedor sin borrar volumen:

```powershell
cd C:\dev\FUNDAMENTOS\mqtt-broker
docker compose down
```

Apagar y borrar datos persistidos del volumen:

```powershell
cd C:\dev\FUNDAMENTOS\mqtt-broker
docker compose down -v
```

## Configuracion

La configuracion activa esta en `config/mosquitto.conf`.

Para esta demo local se permite acceso anonimo porque el broker corre solo como infraestructura local de pruebas. Para despliegue publico, habilita usuario/contrasena, TLS o restringe el puerto por firewall.

## Guia completa

Ver `MQTT_BROKER_LOCAL_GUIA.md` para levantar todo el flujo CafeLab.
