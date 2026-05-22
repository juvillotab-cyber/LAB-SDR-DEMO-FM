# SDR-demo — Receptor FM en Tiempo Real con RTL-SDR

## Objetivo del laboratorio

Implementar un receptor FM en tiempo real usando un RTL-SDR y Linux, capaz de captar señales de radio mediante una antena, procesar las muestras I/Q, visualizar el espectro mediante PSD y demodular una emisora FM para reproducir audio.

El sistema se divide en dos rutas principales:

- Ruta espectral: análisis de potencia y localización de la señal en frecuencia.
- Ruta de audio: demodulación FM y reproducción en tiempo real.

Dashboard web con demodulación FM, Welch PSD en tiempo real, y controles interactivos
de sintonización.  Todo el procesamiento de señal corre en Python puro sobre
librerías científicas estándar (NumPy, SciPy).

---

## Diagrama general del sistema

El flujo completo del sistema parte de la señal captada por la antena y digitalizada por el RTL-SDR como muestras complejas I/Q.

```text
Antena
  │
  ▼
RTL-SDR
  │
  ▼
Muestras I/Q
  │
  ├──────────────────────────────┐
  ▼                              ▼
Proceso 1: Análisis espectral    Proceso 2: Audio en tiempo real
Buffer I/Q                       Buffer rápido
  │                              │
  ▼                              ▼
PSD Welch                        Demodulación FM
  │                              │
  ▼                              ▼
Visualización espectral          Reproducción de audio
Fc, BW, P                        Parlantes

---

## Estructura del proyecto

```
SDR-demo/
├── scripts/                    # Módulos de DSP (sin dependencias web)
│   ├── config.py               # Parámetros globales (dataclass mutable)
│   ├── sdr_acquisition.py      # Hilo de captura I/Q desde RTL-SDR
│   ├── demodulation.py         # Cadena completa de demodulación FM
│   ├── psd.py                  # Welch PSD + ancho de banda ocupado
│   └── audio_player.py         # Salida de audio vía sounddevice
├── backend/                    # Servidor Flask + SocketIO
│   ├── sdr_manager.py          # Orquestador de hilos y buffer PSD
│   └── app.py                  # Rutas REST + streaming WebSocket
├── frontend/                   # UI web (tema oscuro)
│   ├── templates/index.html    # Dashboard con Plotly.js
│   └── static/app.js           # Lógica cliente, SocketIO, controles
├── main.py                     # Entry point
└── requirements.txt
```

---

## Cómo ejecutar

```bash
cd SDR-demo
source ../.venv/bin/activate
python main.py                         # con RTL-SDR conectado
python main.py --no-sdr --port 8080    # solo interfaz web
```

Abrir `http://localhost:5000`.  El dashboard permite cambiar en tiempo real:

| Control          | Efecto                                            |
|------------------|---------------------------------------------------|
| Center Frequency | Frecuencia de sintonía (MHz)                      |
| Gain             | Ganancia del SDR (dB) — afecta la demodulación inmediatamente |
| SPAN             | Rango visible en el eje X (kHz). 0 = auto-escala  |
| NFFT             | Tamaño de la FFT para Welch                       |
| nperseg          | Muestras por segmento en Welch                     |
| noverlap         | Solapamiento entre segmentos en Welch              |

---

# Cómo se logra la demodulación FM

Esta es la parte central del proyecto.  La cadena DSP completa es:

```
Muestras I/Q (240 kS/s)
    │
    ▼
┌──────────────────────┐
│ 1. Filtro RF         │  Butterworth orden 5, corte 100 kHz
│    (pasa-bajas)      │  Aísla una emisora FM del espectro completo
└────────┬─────────────┘
         ▼
┌──────────────────────┐
│ 2. Polar             │  EL CORAZÓN DE LA DEMODULACIÓN
│    Discriminator     │
└────────┬─────────────┘
         ▼
┌──────────────────────┐
│ 3. Decimación        │  240 kS/s → 48 kS/s  (factor 5)
│    + filtro anti-aliasing │
└────────┬─────────────┘
         ▼
┌──────────────────────┐
│ 4. Filtro de audio   │  Butterworth orden 6, corte 16 kHz
│    (pasa-bajas)      │  Elimina residuos de demodulación
└────────┬─────────────┘
         ▼
┌──────────────────────┐
│ 5. De-emphasis       │  τ = 75 µs (estándar FM broadcast)
│    (filtro IIR)      │  Compensa la pre-énfasis del transmisor
└────────┬─────────────┘
         ▼
┌──────────────────────┐
│ 6. DC removal +      │  Centrado en cero + pico normalizado a 0.7
│    normalización     │
└────────┬─────────────┘
         ▼
    Audio 48 kHz float32 → parlantes
```

## 1. Filtro RF — Aislando la emisora

Antes de demodular, se aplica un filtro pasa-bajas Butterworth de orden 5
con frecuencia de corte en 100 kHz.  Esto cumple dos funciones críticas:

- **Aísla una sola emisora FM** del espectro de 240 kHz que entrega el SDR.
  Sin este filtro, el discriminator reaccionaría a múltiples portadoras
  simultáneas.
- **Reduce el ruido fuera de banda**, mejorando la SNR antes de la etapa
  de demodulación.

El filtro se diseña como secciones de segundo orden (SOS) por estabilidad
numérica, y su estado `zi` persiste entre bloques para garantizar
continuidad de fase de un buffer al siguiente.

```python
# scripts/demodulation.py:34-40
_sos_rf = signal.butter(5, 100e3, btype='low', fs=240000, output='sos')
filtered, _rf_zi = signal.sosfilt(_sos_rf, samples, zi=_rf_zi)
```

## 2. Polar Discriminator — El núcleo matemático

La demodulación FM se basa en una propiedad fundamental de la modulación
angular: **la frecuencia instantánea es la derivada de la fase**.

### Base teórica

Una señal FM muestreada se modela como:

$$
s[n] = A \cdot e^{\,j\,\phi[n]}
$$

donde la frecuencia instantánea (en Hz) es:

$$
f[n] = \frac{f_s}{2\pi} \cdot \big(\phi[n] - \phi[n-1]\big)
$$

### Implementación: diferencia de fase conjugada

Para extraer la diferencia de fase entre muestras consecutivas sin calcular
explícitamente `arctan` de cada una (lo cual sería inestable numéricamente
y más lento), se usa el producto conjugado:

$$
d[n] = s[n] \cdot s^*[n-1]
     = A^2 \cdot e^{\,j\,(\phi[n] - \phi[n-1])}
$$

La diferencia de fase está codificada en el argumento de `d[n]`.  Extraerla
con `np.angle()` y escalarla por la tasa de muestreo produce la amplitud
instantánea de audio:

```python
# scripts/demodulation.py:62-64
def fm_demodulate(samples):
    phase = np.angle(samples[1:] * np.conj(samples[:-1]))
    audio = phase * config.sample_rate / (2 * np.pi)
    return audio
```

### Por qué esto funciona

| Propiedad                                          | Consecuencia                                             |
|----------------------------------------------------|----------------------------------------------------------|
| `s[n] · s*[n-1]` usa solo multiplicaciones complejas | Operación O(N), vectorizada en NumPy                     |
| `np.angle()` devuelve valores en (−π, π]            | La diferencia de fase está acotada, sin ambigüedad       |
| Escalar por f_s / 2π                               | Convierte radianes/muestra → Hz (desviación instantánea) |
| No requiere PLL ni NCO                             | Mucho más simple y estable que un demodulador coherente  |

Para FM broadcast con desviación máxima Δf = 75 kHz y f_s = 240 kHz,
la diferencia de fase máxima entre muestras consecutivas es:

$$
|\Delta\phi_{\text{max}}| = 2\pi \cdot \frac{75000}{240000} = 0.625\pi \approx 112^\circ
$$

Esto está cómodamente dentro del rango (−π, π] de `np.angle()`, por lo que
**no hay ambigüedad de fase ni necesidad de unwrapping**.  Esta es una de
las razones por las que el sistema es tan estable.

### Por qué la continuidad temporal es crítica

La modulación FM es **acumulativa en fase**.  Si se pierde una sola muestra
I/Q, la fase se rompe y el discriminador produce un spike de ruido
impulsivo que se escucha como un *click* o *pop* en el audio.  Por eso el
sistema:

- Usa `queue.Queue.put()` (bloqueante) en vez de `put_nowait()` para la
  cola de audio, garantizando que nunca se descarten muestras.
- Mantiene los estados `zi` de todos los filtros entre bloques consecutivos
  (`_rf_zi`, `_decim_zi`, `_audio_zi`, `_deemph_zi`), preservando la
  continuidad de la respuesta del filtro.
- Trabaja a 240 kS/s — una tasa de muestreo baja que minimiza la carga de
  CPU y el tráfico USB, eliminando prácticamente los underruns.


## 3. Decimación 240k → 48k

La salida del discriminador sigue estando a 240 kS/s, pero el audio
audible solo llega hasta ~15 kHz.  Reducir la tasa a 48 kHz (factor 5)
disminuye drásticamente la carga de CPU sin pérdida de calidad.  Un filtro
anti-aliasing Butterworth (orden 6, corte normalizado 0.35) precede al
diezmado `[::5]` para evitar que el ruido de alta frecuencia se pliegue
sobre la banda de audio.

```python
# scripts/demodulation.py:82-85
decimated, _decim_zi = signal.sosfilt(_sos_decim, audio, zi=_decim_zi)
decimated = decimated[::5]
```

## 4. Filtro de audio y De-emphasis

- **Filtro pasa-bajas 16 kHz**: elimina componentes supersónicas
  (residuos del discriminador, ruido de cuantización fuera de banda).
- **De-emphasis (τ = 75 µs)**: filtro IIR de primer orden que atenúa las
  altas frecuencias del audio.  Las emisoras FM aplican *pre-énfasis*
  (boost de agudos) en el transmisor para mejorar la SNR; el receptor debe
  revertirlo con la curva complementaria.  Sin de-emphasis el audio se
  escucha metálico, brillante y con exceso de *hiss*.

```python
# scripts/demodulation.py:57-59
alpha = np.exp(-1.0 / (DEEMPHASIS_TAU * AUDIO_RATE))
de_b = [1.0 - alpha]
de_a = [1.0, -alpha]
```

## 5. Normalización

Antes de enviar el audio a la tarjeta de sonido:
- Se remueve el offset DC (`deemph - np.mean(deemph)`).
- Se normaliza dividiendo por el pico absoluto.
- Se atenúa a 0.7 del rango máximo para evitar clipping en la
  reproducción.


## Por qué la tasa de muestreo del SDR es 240 kS/s y no otra

| Frecuencia de muestreo | Problema                                      |
|------------------------|-----------------------------------------------|
| 2.4 MS/s (inicial)     | Sobrecarga de CPU, pérdida de muestras, jitter, cortes de audio |
| 240 kS/s (óptimo)      | Carga manejable, cero pérdidas, audio estable  |
| < 200 kS/s             | No cubre el ancho de banda FM (~180 kHz)       |

Con 240 kS/s, el ancho de banda de Nyquist es 120 kHz a cada lado de la
portadora — suficiente para cubrir los ~180 kHz de ancho de banda total de
una emisora FM broadcast (regla de Carson: 2 × (75 kHz + 15 kHz) = 180 kHz).


# Welch PSD y ancho de banda ocupado

La PSD se estima con el método de Welch (`scipy.signal.welch`) sobre las
muestras I/Q crudas, usando ventana Hann:

```python
# scripts/psd.py:17-26
f, pxx = signal.welch(samples, fs=240000, window='hann',
                      nperseg=nperseg, noverlap=noverlap,
                      nfft=nfft, return_onesided=False, scaling='density')
```

El **ancho de banda ocupado** se calcula analizando la PSD completa
mediante el método del 99% de potencia acumulada: se ordenan los bins
espectrales por potencia descendente y se acumula hasta alcanzar el 99%
de la potencia total.  El ancho de banda es la diferencia entre la
frecuencia máxima y mínima entre esos bins.  Si la señal es ruido puro,
el sistema cae a un umbral de −20 dB bajo el pico.

Dos líneas verticales rojas punteadas marcan los bordes del ancho de
banda sobre la gráfica, con el valor numérico anotado entre ellas.


# Arquitectura de hilos

El sistema usa tres hilos independientes, igual que `lab.py`:

| Hilo             | Responsabilidad                                  |
|------------------|--------------------------------------------------|
| Capture Thread   | Leer bloques I/Q del RTL-SDR, alimentar colas     |
| Audio Thread     | Cadena DSP completa → salida a parlantes          |
| PSD Broadcast    | Cada ~1 s: snap del buffer I/Q → Welch → WebSocket |

Separar captura, audio y visualización evita que la latencia de GUI
(o del navegador) interfiera con el procesamiento de audio en tiempo real.
Es exactamente la misma arquitectura del `lab.py` original, comprobada
como estable.

El `sdr.close()` se ejecuta en el bloque `finally` del hilo de captura,
no desde el hilo de Flask.  Esto evita segfaults por acceso cross-thread
a libusb y permite que el botón Stop del dashboard apague limpiamente
el pipeline sin matar el servidor.


# Lecciones clave 

1. **La continuidad temporal lo es todo en FM.**  Perder muestras I/Q
   destruye la fase y produce artefactos audibles.  Usar colas bloqueantes
   (`put()`), no descartar paquetes, y mantener estados de filtro entre
   bloques es crítico.

2. **240 kS/s es el sweet spot.**  Suficiente para cubrir el ancho de
   banda FM (~180 kHz), pero lo bastante bajo para no saturar la CPU ni
   el bus USB.

3. **El discriminador polar es inherentemente estable.**  No requiere PLL,
   NCO, ni lazos de enganche.  La diferencia de fase entre muestras
   consecutivas está acotada dentro de (−π, π] para FM broadcast con
   f_s = 240 kHz, lo que elimina ambigüedades de fase.

4. **De-emphasis es obligatorio.**  Sin τ = 75 µs, el audio de una emisora
   FM suena metálico y con exceso de ruido en altas frecuencias.

5. **Separar hilos = estabilidad.**  GUI, audio y captura no deben
   compartir el mismo hilo de ejecución.
