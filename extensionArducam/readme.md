# 🌿 Extensión TinyML con Cámara para Proyecto ViveroSmart.  
**Proyecto base:** `viverosmarth-viveroitt`  
**Autor:** Equipo Vivero ITT  
**Plataforma:** Arduino Tiny Machine Learning Kit (Arduino Nano 33 BLE Sense + OV7675 Camera)

---

## 🎯 Objetivo

Implementar una extensión de visión artificial embebida utilizando TinyML dentro del proyecto ViveroSmart, con el propósito de diagnosticar de manera automática y temprana el estado de salud de las plantas. Esta extensión permitirá identificar síntomas de estrés o enfermedades foliares mediante el procesamiento local de imágenes capturadas por sensores o cámaras integradas, optimizando la gestión del cultivo y reduciendo la necesidad de intervención manual o conexión constante a la nube.

## Objetivos Específicos Complementarios

- Diseñar e implementar un modelo de TinyML entrenado con imágenes de hojas sanas y enfermas de diferentes especies vegetales.

- Optimizar el modelo para ejecución embebida, reduciendo su tamaño y consumo de energía sin afectar la precisión.

- Integrar un módulo de cámara o sensor de imagen que capture hojas y procese las imágenes localmente.

- Evaluar la precisión del diagnóstico en condiciones reales del vivero (iluminación variable, diferentes etapas de crecimiento).

- Conectar el sistema al entorno ViveroSmart, enviando resultados de diagnóstico a la plataforma central o dashboard de monitoreo.

---

## 🧠 Descripción General

El sistema toma imágenes de hojas usando la cámara del **TinyML Kit** y las procesa localmente para clasificar el estado de la planta:

- 🌱 **Sana**  
- 🍂 **Estrés hídrico**  
- 🦠 **Infección fúngica o bacteriana**

El modelo de **aprendizaje automático** se entrena con TensorFlow Lite y se despliega en el microcontrolador para inferencias en tiempo real sin conexión a internet.

---

## 🧩 Componentes

| Componente | Descripción |
|-------------|--------------|
| Arduino Nano 33 BLE Sense | MCU con procesador ARM Cortex-M4F y sensor IMU, micrófono y BLE |
| Cámara OV7675 | Sensor VGA para capturar imágenes (TinyML Kit) |
| TensorFlow Lite Micro | Librería para inferencia de modelos ML embebidos |
| LED RGB | Indicador de estado (verde = sano, rojo = enfermo, azul = estrés) |
| Sensor DHT11 / BME280 | Medición de temperatura y humedad |
| MQTT o Serial | Comunicación de resultados a la plataforma ViveroSmart |

---

## 🧰 Software y Librerías

- Arduino IDE o Arduino Web Editor  
- TensorFlow Lite for Microcontrollers  
- Arduino_TensorFlowLite  
- Arduino_OV767X  
- Arduino_LSM9DS1 (IMU)  
- ArduinoBLE (para comunicación)  

---

## 🔍 Flujo de Trabajo

1. **Captura de Imagen:**  
   La cámara toma una imagen 96x96 px RGB.

2. **Preprocesamiento:**  
   Reducción de tamaño y normalización.

3. **Inferencia TinyML:**  
   El modelo ejecuta la predicción localmente.

4. **Visualización / Alerta:**  
   LED indica el estado de la planta.  
   Los resultados se envían a ViveroSmart vía BLE o Serial.

---

## 🧪 Entrenamiento del Modelo

1. Recolectar dataset de hojas sanas y enfermas.  
   - Ejemplo: `data/healthy/`, `data/diseased/`, `data/stressed/`
2. Entrenar en TensorFlow con CNN ligera (MobileNetV1 adaptada).  
3. Convertir modelo:  
   ```bash
   tflite_convert --output_file=model.tflite --saved_model_dir=plant_model/
   ```
4. Cuantizar para microcontroladores:  
   ```python
   converter.optimizations = [tf.lite.Optimize.DEFAULT]
   ```

---

## 📸 Código Arduino

``` cpp#include <Arduino_OV767X.h>

#define FRAME_WIDTH 160
#define FRAME_HEIGHT 120
#define FRAME_SIZE (FRAME_WIDTH * FRAME_HEIGHT)

// Buffer global alineado para acceso más rápido
uint8_t frame_buffer[FRAME_SIZE] __attribute__((aligned(4)));

const uint8_t FRAME_START[4] = {0xAA, 0x55, 0xAA, 0x55};
const uint8_t FRAME_END[4]   = {0x55, 0xAA, 0x55, 0xAA};

void setup() {
  Serial.begin(115200);
  while (!Serial);

  Serial.println("Iniciando cámara OV7675...");

  // Modo más estable: baja resolución + escala gris
  if (!Camera.begin(QQVGA, GRAYSCALE, 1)) {
    Serial.println("Error al iniciar cámara OV7675");
    while (true) delay(1000);
  }

  Serial.println("Cámara inicializada correctamente");
  delay(500);
}

void loop() {
  static uint32_t last_frame_time = millis();

  // Captura el frame (sin retorno)
  Camera.readFrame(frame_buffer);

  // Marca de inicio
  Serial.write(FRAME_START, sizeof(FRAME_START));

  // Enviar buffer completo
  Serial.write(frame_buffer, FRAME_SIZE);

  // Marca de fin
  Serial.write(FRAME_END, sizeof(FRAME_END));

  // Sincronizar ~5 FPS
  uint32_t frame_time = millis() - last_frame_time;
  last_frame_time = millis();
  uint32_t wait_ms = max(0, 200 - (int)frame_time);
  delay(wait_ms);
}


```
<img width="226" height="206" alt="image" src="https://github.com/user-attachments/assets/ed13f90e-6147-420f-bb3e-440d0ead10de" />

------------------------------------------------------------------------

## 🐍 Script Python: Visualización en tiempo real

Instala dependencias:

``` bash
pip install pyserial numpy opencv-python
```

Ejecuta este script (ajusta el puerto COM según tu caso):

``` pythonimport serial
import numpy as np
import cv2
import time

# ====== CONFIGURACIÓN ======
PORT = "COM11"          # 🔧 Cambia esto según tu puerto Arduino
BAUD = 921600          # Debe coincidir con Serial.begin() del Arduino
FRAME_WIDTH = 160
FRAME_HEIGHT = 120
FRAME_SIZE = FRAME_WIDTH * FRAME_HEIGHT

# Delimitadores definidos en el Arduino
FRAME_START = b'\xAA\x55\xAA\x55'
FRAME_END = b'\x55\xAA\x55\xAA'

# ====== INICIO ======
ser = serial.Serial(PORT, BAUD, timeout=1)
time.sleep(2)  # Esperar que se reinicie el Arduino

print("📡 Esperando datos de la cámara...")

buffer = bytearray()

try:
    while True:
        data = ser.read(ser.in_waiting or 1)
        if not data:
            continue
        buffer.extend(data)

        # Buscar inicio y fin del frame
        start_idx = buffer.find(FRAME_START)
        end_idx = buffer.find(FRAME_END, start_idx + len(FRAME_START))

        if start_idx != -1 and end_idx != -1:
            # Extraer el bloque de imagen entre los delimitadores
            frame_data = buffer[start_idx + len(FRAME_START):end_idx]
            buffer = buffer[end_idx + len(FRAME_END):]  # limpiar el buffer

            if len(frame_data) == FRAME_SIZE:
                # Convertir bytes → numpy array → imagen
                frame = np.frombuffer(frame_data, dtype=np.uint8).reshape((FRAME_HEIGHT, FRAME_WIDTH))

                # Mostrar en ventana OpenCV
                cv2.imshow("Arduino Cam (OV7675)", frame)

                # Tecla 's' para guardar frame
                key = cv2.waitKey(1) & 0xFF
                if key == ord('s'):
                    filename = f"frame_{int(time.time())}.png"
                    cv2.imwrite(filename, frame)
                    print(f"💾 Frame guardado como {filename}")
                elif key == 27:  # ESC para salir
                    break
            else:
                print(f"⚠️ Frame incompleto: {len(frame_data)} bytes")

except KeyboardInterrupt:
    print("\n🛑 Interrupción manual.")

finally:
    ser.close()
    cv2.destroyAllWindows()
    print("🔚 Conexión cerrada.")

```

------------------------------------------------------------------------

## ⚙️ Funcionamiento

1.  El Arduino inicializa la cámara OV7675 en resolución **QQVGA
    (160×120)** y modo **grayscale**.\

2.  En cada iteración del `loop()`, captura un frame y lo envía por
    Serial con el siguiente formato:

        [0xFF][0xD8] + 19200 bytes de imagen + [0xFF][0xD9]

3.  El script Python escucha el puerto serial, detecta los marcadores de
    inicio/fin y reconstruye el frame.

4.  OpenCV muestra el flujo de video en una ventana a \~5 FPS.

------------------------------------------------------------------------

## 🧩 Solución de problemas

  -----------------------------------------------------------------------
  Problema                            Solución
  ----------------------------------- -----------------------------------
  ❌ `No device found on COMx`        Verifica el puerto en Arduino IDE o
                                      cambia de cable USB

  ⚫ Imagen distorsionada             Baja la velocidad
                                      `Serial.begin(57600)` o aumenta
                                      `delay(300)`

  🪫 Cámara no inicializa             Usa
                                      `Camera.begin(QQVGA, RGB565, 1)` si
                                      tu sensor no soporta GRAYSCALE

  🧱 No se muestra nada en Python     Revisa que el puerto COM coincida y
                                      que el buffer contenga datos
  -----------------------------------------------------------------------

## 🚀 Resultados Esperados

- Diagnóstico local en menos de **1 segundo** por imagen  
- Clasificación con **precisión >85%**  
- Sin necesidad de conexión constante a internet  
- Integración directa con dashboard *ViveroSmart*

---

## 📚 Referencias

- Arduino TinyML Kit Documentation – [https://docs.arduino.cc/tutorials/](https://docs.arduino.cc/tutorials/)  
- TensorFlow Lite for Microcontrollers – [https://www.tensorflow.org/lite/microcontrollers](https://www.tensorflow.org/lite/microcontrollers)  
- Pete Warden, Daniel Situnayake. *TinyML: Machine Learning with TensorFlow Lite on Arduino and Ultra-Low-Power Microcontrollers* (O'Reilly, 2020)  

---


**© 2025 Equipo Vivero ITT**  
Proyecto académico para extensión del sistema ViveroSmart con visión embebida y TinyML.
