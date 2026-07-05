# Guia local CafeLab: broker MQTT + edge + bridge

**Para apagar todo:**
Apagar Tracksilo (desconectar)
Apagar Edge (stop)
Apagar mqtt-bridge y mqtt-broker (ctrl+c)
Apagar Mosquitto broker (docker compose down)
Apagar Docker desktop


**Para encender todo:**
Prender Docker desktop

Levantar broker
cd C:\dev\FUNDAMENTOS\mqtt-broker
docker compose up -d
docker compose ps
Debe salir cafelab-mosquitto en estado Up

Abrir subscriber para ver mensajes:
[cd C:\dev\FUNDAMENTOS\mqtt-broker]
[docker exec -it cafelab-mosquitto mosquitto_sub -h localhost -p 1883 -t cafelab/iot/telemetry -v]
Dejarlo abierto

Levantar mqtt-bridge
cd C:\dev\FUNDAMENTOS\mqtt-bridge
.\.venv\Scripts\activate
python mqtt_bridge.py
Debe decir que se conectó al broker y se suscribió al tópico.
AQUÍ INCIA SESIÓN

Levantar Edge en la red de Martin Router King
cd C:\dev\FUNDAMENTOS\edge-clean
.\.venv\Scripts\activate
Ejecutar en app.py

En caso de querer cambiar de cuenta en caliente, apagar el bridge, volverlo a encender e ingresar las credenciales de la otra cuenta. En onboarding hacer lo mismo.