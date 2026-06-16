# Ventilador RF — ESP32 + CC1101 + ESPHome + Home Assistant

Control bidireccional de un ventilador de techo con mando RF 433 MHz, integrado en Home Assistant mediante un ESP32 y un módulo CC1101.

---

## ¿Qué hace este proyecto?

Convierte un ventilador de techo "tonto" (controlado solo por mando RF físico) en un dispositivo de domótica completo integrado en Home Assistant, manteniendo el mando físico 100% funcional. Cualquier cambio —ya sea desde la app o desde el mando— queda reflejado en ambos lados en tiempo real.

**Entidades que aparecen en Home Assistant:**

| Entidad | Funcionalidad |
|---|---|
| Ventilador | On/Off + 6 velocidades |
| Luz 1 | On/Off + 8 niveles de brillo + 3 tonos de color |
| Luz 2 | On/Off + 3 tonos de color |
| Brisa Natural | Botón para activar el modo oscilante |

---

## El problema

El mando RF del ventilador opera a **433.92 MHz con modulación OOK** y un protocolo propietario de 29 bits. El desafío principal fue doble:

1. **TX no funcionaba aunque RX sí.** El módulo CC1101 capturaba las tramas del mando a la perfección, pero el ESP32 no conseguía hacer que el ventilador respondiera a sus transmisiones.

2. **Sincronización bidireccional.** Al usar el mando físico, Home Assistant no sabía qué había cambiado. Al mandar órdenes desde HA, el mando no debía volver a retransmitirlas creando bucles infinitos.

---

## Componentes

| Componente | Detalle |
|---|---|
| **Microcontrolador** | ESP32 (placa `esp32dev`) |
| **Módulo RF** | CC1101 a 433.92 MHz |
| **Firmware** | ESPHome 2026.5.1 con framework ESP-IDF |
| **Integración** | Home Assistant vía API ESPHome |

### Conexiones ESP32 ↔ CC1101

| Función | GPIO ESP32 | Pin CC1101 |
|---|---|---|
| SPI CLK | GPIO14 | SCK |
| SPI MOSI | GPIO23 | MOSI |
| SPI MISO | GPIO19 | MISO / GDO1 |
| CS | GPIO32 | CSN |
| TX (datos) | **GPIO26** | **GDO0** |
| RX (datos) | **GPIO25** | **GDO2** |

La configuración es de **doble pin**: GDO0 exclusivo para TX y GDO2 exclusivo para RX. Esto es clave — el modo single-pin no funciona con este protocolo.

---

## La solución

### Por qué no funcionaba TX

El problema era que las tramas que se transmitían se construían de forma "idealizada" (pulsos perfectamente uniformes de 370 µs y 1090 µs). El ventilador las rechazaba. La solución fue **capturar las tramas reales** tal y como las genera el mando físico, con sus imperfecciones naturales, y transmitirlas exactamente igual con `transmit_raw`.

```
# Trama idealizada (NO funciona):
code: [370,-1090,370,-1090,370,-1090,1090,-370, ...]

# Trama capturada real (SÍ funciona):
code: [375,-1088,364,-1107,359,-1101,1091,-369, ...]
```

Cada uno de los 14 comandos usa su trama capturada real verificada.

### Protocolo RF detectado

- **Frecuencia:** 433.92 MHz
- **Modulación:** OOK (On-Off Keying)
- **Protocolo:** RC Switch protocolo 1
- **Bits:** 29 bits por trama
- **Prefijo común:** `00011111000000100011` (20 bits, identifica el dispositivo)
- **Pulso corto:** ~370 µs | **Pulso largo:** ~1090 µs
- **Transmisión:** 10 repeticiones con 5 ms de espera entre ellas

### Sincronización bidireccional sin bucles

El principal reto lógico era: cuando el mando físico envía una trama, el ESP32 la recibe y actualiza HA. Pero al actualizar HA, éste llama de vuelta al ESP32 para transmitir por RF... lo que crearía un bucle infinito.

La solución es una **ventana de supresión auto-expirante**:

```cpp
// Al recibir del mando físico:
id(suppress_until) = millis() + 500;  // bloquea TX durante 500 ms

// En cada ruta de TX:
if (millis() < id(suppress_until)) return;  // ignora si venimos del mando
```

Este timestamp se expira solo — no hay scripts ni flags que puedan quedarse "pegados".

### Brillo relativo → absoluto

La lámpara no tiene feedback de posición: el botón `dim+` sube un escalón, `dim-` baja uno. Para poder poner el brillo a un porcentaje concreto desde HA, el ESP mantiene un contador interno (`luz1_lvl`, 1–8) y calcula cuántos pasos hay que enviar:

```cpp
int d = lvl_objetivo - id(luz1_lvl);
for (int i = 0; i < d;  i++) id(tx_step_luz1).execute(1);  // sube
for (int i = 0; i < -d; i++) id(tx_step_luz1).execute(2);  // baja
```

Los pasos se encolan (`mode: queued`) con 700 ms de separación para que el ventilador procese cada RF antes de recibir la siguiente.

---

## Estructura de archivos

```
├── ventilador-rf.yaml   # Configuración ESPHome
└── secrets.yaml         # Credenciales WiFi/API (NO subir al repo)
```

**Para flashear:**

```bash
esphome run ventilador-rf.yaml
```

---

## Códigos RF capturados

| Comando | Código hex |
|---|---|
| Ventilador apagado | `0x3E0470B` |
| Velocidad 1 | `0x3E047E8` |
| Velocidad 2 | `0x3E047C8` |
| Velocidad 3 | `0x3E047A9` |
| Velocidad 4 | `0x3E04789` |
| Velocidad 5 | `0x3E0476A` |
| Velocidad 6 | `0x3E0474A` |
| Brisa Natural | `0x3E046B5` |
| Luz 1 On/Off | `0x3E04695` |
| Luz 1 Tono | `0x3E04637` |
| Luz 1 Dim − | `0x3E046F4` |
| Luz 1 Dim + | `0x3E04733` |
| Luz 2 On/Off | `0x3E04676` |
| Luz 2 Tono | `0x3E04752` |

---

## Configuración de secrets.yaml

Crea un archivo `secrets.yaml` en la misma carpeta (nunca subas este archivo):

```yaml
wifi_ssid: "TuRedWiFi"
wifi_password: "TuContraseña"
api_encryption_key: "TuClaveAPI32bytes"
ota_password: "TuContraseñaOTA"
ap_password: "TuContraseñaAP"
```

---

## Limitaciones conocidas / problemas no resueltos

### 1. Punto ciego durante transmisión (~0.5 s)
El CC1101 es half-duplex: mientras transmite **no puede escuchar**. Si se pulsa el mando justo cuando el ESP está transmitiendo (ventana de ~500 ms), esa pulsación se pierde y el estado puede desincronizarse. Con un solo módulo este límite es físico e irresoluble. Solución parcial aplicada: reducir las repeticiones de 20 a 10 para minimizar el tiempo de ceguera.

### 2. Sincronización inicial del brillo tras un reinicio
El ESP guarda el nivel de brillo en memoria flash (`restore_value: true`), pero si se cambia el brillo con el mando físico mientras el ESP está apagado, el contador interno queda desincronizado. Solución: tras un reinicio, pulsar `dim−` unas cuantas veces con el mando hasta llegar al mínimo y luego subir desde cero.

### 3. Botón "Reverse" (modo verano/invierno) no implementado
El mando tiene un botón para invertir el giro del motor. No se capturó la trama durante el desarrollo. Se puede añadir fácilmente si se captura el código.

### 4. Luz 2 no tiene dimmer
El hardware del ventilador no expone un botón de dimmer para la Luz 2. El slider de brillo en HA existe como control decorativo pero no transmite nada.

### 5. Tonalidad: control cíclico sin feedback
El botón de tono cicla entre 3 posiciones (cálido → neutro → frío) sin que la lámpara confirme en qué posición está. Si por algún motivo el contador interno se desincroniza, el tono seleccionado en HA no coincide con el real. No hay solución sin un sensor externo de color.

### 6. ESPHome issue #16876
El parámetro `gdo0_pin` en el bloque `cc1101:` rompe el enrutamiento RMT. La configuración de este proyecto lo omite deliberadamente — el pin GDO0 se declara únicamente en `remote_transmitter`.

---

## Referencias

- [mrkirby153 — Making a Dumb Fan Smart (Dic 2025)](https://www.mrkirby153.com/post/2025/12/esp32-fan-controller/) — mismo escenario, solución clave para el problema TX/RX doble pin
- [Proyecto de referencia cc1101-esp32-esphome-fan-controller](https://github.com/Daedilus/cc1101-esp32-esphome-fan-controller) — usa el componente nativo CC1101 de ESPHome
- [ESPHome CC1101 (componente nativo)](https://esphome.io/components/cc1101.html)
- [ESPHome issue #16876](https://github.com/esphome/esphome/issues/16876) — bug gdo0_pin rompe RMT




## ☕ Support
If you find this project useful, consider buying me a coffee:
[Buy Me a Coffee](https://buymeacoffee.com/blindforlove])
