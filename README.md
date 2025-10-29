## Documentacion del proyecto
# 🌿 Proyecto IoT con MicroPython y Flespi MQTT

Este proyecto simula un sistema de **monitoreo de plantas** usando un microcontrolador compatible con **MicroPython** (como ESP32 o ESP8266).  
El dispositivo envía lecturas simuladas de temperatura, humedad, suelo, luz y agua hacia un **broker MQTT de Flespi**.

---

## 🚀 Requisitos

- Placa ESP32 , ESP8266 y RPI PICO W 
- Firmware **MicroPython**  
- Conexión WiFi  
- Cuenta en [https://flespi.io](https://flespi.io)

---

## 🔑 Configuración de credenciales

Crea un archivo llamado `password.py` con tus credenciales:

```python
WIFI_SSID = "TuWiFi"
WIFI_PASSWORD = "TuContraseñaWiFi"
MQTT_BROKER = "mqtt.flespi.io"
FLESPI_TOKEN = "TU_TOKEN_FLESPI"
```

> 💡 Tu token Flespi se obtiene en el **panel de Flespi → Tokens → Create Token**  
> Usa permisos de tipo **"full access"** o al menos **MQTT Publish**.

---

## 🧠 Código principal (`main.py`)

```python
import network
import time
import ubinascii
import machine
import random
from umqtt.simple import MQTTClient
from password import WIFI_SSID, MQTT_BROKER, FLESPI_TOKEN, WIFI_PASSWORD

CLIENT_ID = ubinascii.hexlify(machine.unique_id())
TOPICS = {
    "temp": "plantcare/1/temp",
    "hum": "plantcare/1/hum",
    "soil": "plantcare/1/soil",
    "air": "plantcare/1/air",
    "light": "plantcare/1/light",
    "water": "plantcare/1/water"
}

def connect_wifi():
    wlan = network.WLAN(network.STA_IF)
    wlan.active(True)
    wlan.connect(WIFI_SSID, WIFI_PASSWORD)
    print("Conectando a WiFi...")
    while not wlan.isconnected():
        print(".", end="")
        time.sleep(1)
    print("\nWiFi conectado:", wlan.ifconfig())

def connect_mqtt():
    client = MQTTClient(CLIENT_ID, MQTT_BROKER, user=FLESPI_TOKEN, password="")
    client.connect()
    print("Conectado a MQTT:", MQTT_BROKER)
    return client

def read_sensors():
    temp = round(random.uniform(20, 35), 1)
    hum = round(random.uniform(30, 90), 1)
    soil = random.randint(0, 100)
    air = random.randint(0, 100)
    light = random.randint(100, 1000)
    water = random.choice([0, 1])
    return {"temp": temp, "hum": hum, "soil": soil, "air": air, "light": light, "water": water}

connect_wifi()
client = connect_mqtt()

print("Iniciando envío de datos simulados...\n")

while True:
    data = read_sensors()
    for key, topic in TOPICS.items():
        value = str(data[key])
        client.publish(topic, value)
        print(f"Publicado {topic}: {value}")
    print("---- Ciclo completado ----\n")
    time.sleep(5)
```

---

## 📡 Publicación en Flespi MQTT

Cada dato se publica en los siguientes **tópicos**:

| Sensor | Tópico MQTT | Ejemplo de valor |
|:-------|:-------------|:----------------|
| Temperatura | `plantcare/1/temp` | 28.5 |
| Humedad | `plantcare/1/hum` | 65.0 |
| Suelo | `plantcare/1/soil` | 72 |
| Aire | `plantcare/1/air` | 80 |
| Luz | `plantcare/1/light` | 650 |
| Agua | `plantcare/1/water` | 1 |

---

## 🧩 Explicación general

1. **Conexión WiFi:**  
   El dispositivo se conecta a tu red local usando las credenciales del archivo `password.py`.

2. **Cliente MQTT:**  
   Se crea un cliente con `umqtt.simple` usando tu token Flespi como usuario.

3. **Lecturas simuladas:**  
   `read_sensors()` genera datos aleatorios que simulan valores reales de sensores.

4. **Publicación periódica:**  
   Cada 5 segundos se publican los valores en los tópicos definidos en el diccionario `TOPICS`.

---

## 💡 Notas

- Si usas **Thonny** o **mpremote**, puedes subir los archivos fácilmente.  
- Si deseas usar sensores reales (DHT11, LDR, etc.), reemplaza `read_sensors()` con lecturas de tus sensores físicos.  
- Puedes ajustar el intervalo de publicación modificando `time.sleep(5)`.

---

## Aqui algunas fotos

<img width="1600" height="850" alt="image" src="https://github.com/user-attachments/assets/aed72fe4-ff67-4bb0-94d3-f1eeb45e6ec6" />
<img width="1600" height="850" alt="image" src="https://github.com/user-attachments/assets/1457a2ac-443d-4609-9b2a-572cb9b3d082" />
<img width="1600" height="850" alt="image" src="https://github.com/user-attachments/assets/a3955243-338c-49c8-ab5c-bceb8df1830d" />
<img width="1600" height="850" alt="image" src="https://github.com/user-attachments/assets/73b18312-2876-4813-89a1-9d4e5d24bff7" />

