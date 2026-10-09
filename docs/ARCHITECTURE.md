# MangaAI — Arquitectura y Stack

> Robot cognitivo con forma de gato: camina en cuatro patas, maúlla, ronronea, habla, reconoce a su persona y tiene memoria y estado emocional propios.

**Decisiones base**

| Decisión | Elección |
|---|---|
| Presupuesto hardware | $800–2000 USD |
| Dónde piensa | Híbrido: reflejos, percepción y voz básica en el robot; conversación y razonamiento profundo en la nube |
| Movilidad | Cuadrúpedo caminante (12 DOF en patas) |
| Equipo | Software (Python/C++), electrónica, impresión 3D/CAD, ROS |

**Principio rector:** el gato tiene que estar *vivo sin internet*. Sin conexión pierde la conversación compleja, pero sigue caminando, maullando, ronroneando, reaccionando a caricias y reconociendo a su persona.

---

## 1. Vista general

```
┌──────────────────────────────────────── NUBE ────────────────────────────────────────┐
│  Claude (LLM)  ·  TTS de alta calidad (opcional)  ·  backup de memoria                │
└───────────────────────────────▲──────────────────────────────────────────────────────┘
                                │ HTTPS (async, streaming) — opcional/degradable
┌───────────────────────────────┴──── CEREBRO: Jetson Orin Nano Super 8GB ─────────────┐
│                                                                                       │
│  ┌─ Capa 4: MENTE (Python) ────────────────────────────────────────────────────────┐ │
│  │  Emociones/Necesidades ─► Selector de comportamientos (Behavior Tree + utilidad) │ │
│  │  Memoria (episódica + semántica)   Personalidad   Cliente LLM (tool use)        │ │
│  └───────────▲──────────────────────────────────────────────────┬──────────────────┘ │
│              │ eventos (ROS 2 topics)                            │ intenciones        │
│  ┌─ Capa 3: PERCEPCIÓN ────────────┐            ┌─ Capa 3: EXPRESIÓN ──────────────┐ │
│  │ Cara / persona / seguimiento    │            │ Ojos (render), maullidos, voz,   │ │
│  │ Audio: VAD, wake word, STT, DOA │            │ ronroneo, animaciones corporales │ │
│  │ Tacto, IMU, contacto de patas   │            └──────────────┬───────────────────┘ │
│  └───────────▲─────────────────────┘                           │                     │
│  ┌─ Capa 2: CONTROL MOTOR (ros2_control) ─────────────────────▼───────────────────┐ │
│  │  Política de locomoción RL (ONNX, 50 Hz) · Mezclador de poses/animaciones       │ │
│  │  Postura, equilibrio, levantarse tras caída                                     │ │
│  └───────────▲─────────────────────────────────────────────────┬──────────────────┘ │
└──────────────┼─────────────────────────────────────────────────┼────────────────────┘
               │ micro-ROS (USB serial)                           │
┌──────────────┴──── Capa 1: TIEMPO REAL — Teensy 4.1 (C++) ──────▼────────────────────┐
│  Bus de servos (500 Hz–1 kHz) · IMU · sensores táctiles · haptics · SEGURIDAD        │
│  (límites de torque/temperatura, watchdog: si el Jetson muere, el gato se sienta)    │
└──────────────────────────────────────────────────────────────────────────────────────┘
```

Cada capa solo habla con la de al lado. La Mente **nunca** mueve un servo directamente: emite intenciones ("acercarse a Ana", "estirarse", "mostrar alegría") y las capas inferiores deciden cómo.

---

## 2. Hardware

### 2.1 Dimensiones objetivo
- Largo ~45 cm (sin cola), alto a la cruz ~25 cm, **peso objetivo 2.5–3.5 kg**.
- El peso es la restricción #1: cada 100 g extra exige más torque en las patas y reduce la batería. Llevar una hoja de presupuesto de masa desde el día 1.

### 2.2 Cómputo
| Componente | Elección | Por qué |
|---|---|---|
| Cerebro | **Jetson Orin Nano Super 8 GB** | GPU para visión, STT y política RL en el robot; buen soporte ROS 2 |
| Tiempo real | **Teensy 4.1** (600 MHz, micro-ROS) | Lazos de control deterministas, independiente del SO Linux |
| Almacenamiento | NVMe 256 GB | Modelos, memoria, logs |

### 2.3 Actuadores (~20 DOF)
| Zona | DOF | Actuador sugerido |
|---|---|---|
| Patas (4 × cadera abd/aducción, cadera, rodilla) | 12 | **Feetech STS3250** (~50 kg·cm @12 V, bus serie, feedback de posición/carga/temperatura) |
| Cuello / cabeza (pan, tilt, roll) | 3 | Feetech STS3215 |
| Cola (base 2 ejes) | 2 | STS3215 o micro-servos de bus; cola flexible por cable |
| Orejas | 2 | Micro-servos de bus (SCS0009) |
| Mandíbula | 1 | Micro-servo (sincronizado con audio al hablar/maullar) |

Alternativa premium para patas si el presupuesto lo permite: Dynamixel XC/XM430 (más silenciosos y precisos, ~3× el precio).

> **Ruido = realismo.** El zumbido de servos rompe la ilusión. Medir dB de los servos candidatos antes de comprar 12; considerar reducciones de engranaje propias o motores BLDC cuasi-directos en una v2.

### 2.4 Sensores
| Sensor | Elección | Uso |
|---|---|---|
| Visión | Cámara IMX219/IMX477 gran angular en la frente/nariz (o **OAK-D Lite** si se quiere profundidad) | Caras, personas, objetos, evitar obstáculos |
| Audio | Array de micrófonos **ReSpeaker XVF3800** (o 4-mic) | Dirección de la voz (girar la cabeza), cancelación de eco, VAD |
| IMU | **BNO085** | Equilibrio, detectar que lo levantan / se cae |
| Tacto | Electrodos capacitivos + **MPR121** (cabeza, mentón, lomo, panza, cola) | Caricias → ronroneo; panza → "trampa" |
| Proximidad | 2–4 ToF **VL53L1X** | Precipicios (mesas/escaleras), obstáculos cercanos |
| Contacto de patas | Corriente/carga del servo (+ FSR opcional) | Locomoción, detectar suelo |

### 2.5 Expresión
- **Ojos:** 2 pantallas redondas IPS 1.28" (**GC9A01**, 240×240) con iris/pupila renderizados (dilatación, parpadeo lento = "te quiero" en gatuno). Detrás de una lente domo para profundidad.
- **Voz y maullidos:** altavoz 3–5 W + amplificador **MAX98357A** (I2S).
- **Ronroneo:** actuador háptico **LRA** + driver **DRV2605** en el pecho. Vibración real a ~25 Hz, no solo sonido. Probablemente el detalle más impactante.
- **Respiración:** ciclo lento del torso/columna cuando está en reposo.

### 2.6 Energía
- Batería Li-ion **4S (14.8 V) ~5000 mAh** con BMS → reguladores: 12 V (servos, alta corriente), 5 V/19 V (Jetson), 3.3 V (sensores).
- Interruptor de emergencia físico accesible.
- Fase posterior: **base de carga** a la que el gato camina sola cuando "tiene sueño" (batería baja = necesidad de dormir).

### 2.7 Cuerpo
- Esqueleto impreso en PETG/PA-CF, piezas críticas con insertos de latón.
- Columna con 1–2 DOF pasivos/activos para que no camine "como mesa".
- Capa intermedia de espuma + **piel de peluche** removible y lavable, con aberturas para sensores táctiles (los capacitivos funcionan a través de tela fina).

---

## 3. Software

### 3.1 Stack
| Área | Tecnología |
|---|---|
| SO del Jetson | JetPack 6.x (Ubuntu 22.04) |
| Middleware | **ROS 2** (Humble nativo; Jazzy en contenedor si se necesita) |
| Firmware | C++ con **PlatformIO** + **micro-ROS** en Teensy |
| Control | **ros2_control** (hardware interface propio para el bus Feetech) |
| Simulación | **MuJoCo** (+ MuJoCo Playground/MJX para entrenar RL en GPU) |
| CAD → sim | Onshape/Fusion → `onshape-to-robot` → URDF/MJCF |
| Locomoción | Política RL (PPO) entrenada en sim → **ONNX** → TensorRT/ONNX Runtime en el Jetson |
| Visión | YOLO (personas/objetos), **InsightFace/ArcFace** (reconocer a su persona), todo con TensorRT |
| Audio | openWakeWord (nombre del gato), Silero VAD, **faster-whisper** (STT local) |
| TTS | Piper (local, sin internet) + TTS en la nube opcional para mejor voz |
| Maullidos | Biblioteca de muestras + modulación (tono, duración, intensidad) según emoción |
| Mente / lógica | **Python 3.11+**, `py_trees` (behavior trees), asyncio |
| LLM | **Claude API** — Haiku 4.5 para respuestas rápidas, Sonnet 5 para reflexión y consolidación de memoria |
| Memoria | **SQLite** + embeddings (`sqlite-vec`) |
| Panel web | **FastAPI** + frontend ligero (configuración, teleop, ver estado emocional) |
| Debug | **Foxglove** Studio, `rosbag2` |
| Calidad | `pytest`, `ruff`, GitHub Actions, Docker para builds reproducibles |

### 3.2 Capa 1 — Firmware (Teensy)
- Lee/escribe el bus de servos a 500 Hz–1 kHz; expone estado articular (posición, velocidad, carga, temperatura).
- PD de bajo nivel según el objetivo que mande el Jetson.
- **Seguridad autónoma:** límites de torque y temperatura, watchdog (sin comandos en 100 ms → postura sentada y torque reducido), detección de batería baja.
- Lee IMU, MPR121 (tacto) y maneja el DRV2605 (ronroneo) para que el ronroneo responda a la caricia con latencia mínima.

### 3.3 Capa 2 — Control motor
- **Locomoción:** política RL que recibe (velocidad deseada, orientación IMU, estado articular, acción previa) y emite objetivos articulares a 50 Hz. Entrenada con *domain randomization* (masa, fricción, latencia, holgura de servos) para cruzar sim→real.
- **Fallback:** generador central de patrones (CPG) clásico para tener un caminar básico antes que el RL.
- **Animaciones:** biblioteca de clips keyframe (sentarse, estirarse, lamerse, amasar, bostezar, acicalarse, "loaf") hechos en Blender o en un editor propio, mezclados con la locomoción por articulación (ej. la cola y la cabeza animan mientras las patas caminan).
- **Mirada:** la cabeza y los ojos siguen el objetivo de atención (cara, sonido, juguete) de forma independiente del cuerpo.

### 3.4 Capa 3 — Percepción y expresión
Convierte el mundo en **eventos** semánticos para la Mente:
`persona_vista(id=ana, dist=1.2m)`, `voz_detectada(dir=40°)`, `caricia(zona=cabeza, intensidad=0.7)`, `levantado()`, `precipicio()`, `nombre_llamado()`.

Y convierte intenciones de la Mente en expresión coordinada: una emoción "contento" ajusta pupilas, orejas, cola, ronroneo y tipo de maullido a la vez.

### 3.5 Capa 4 — La Mente
El corazón del proyecto. Tres subsistemas:

**a) Estado interno (siempre local, ~10 Hz)**
- *Emoción:* modelo continuo valencia/excitación (+ dominancia), que decae con el tiempo.
- *Necesidades:* energía (batería), sueño, social, curiosidad, juego, comodidad. Crecen con el tiempo y se satisfacen con acciones.
- *Personalidad:* parámetros estables (cariñoso, juguetón, independiente, curioso) que sesgan todo. Un gato realista **a veces ignora** y a veces pide atención.

**b) Selección de comportamiento (local)**
- Behavior tree con nodos de **utilidad**: cada comportamiento (dormir, buscar a Ana, explorar, jugar, acicalarse, pedir mimos, ir a cargar) puntúa según necesidades + emoción + contexto.
- Reflejos con prioridad máxima: precipicio, caída, lo levantan, ruido fuerte (susto).

**c) Cognición (nube, asíncrona, degradable)**
- El LLM **no controla el cuerpo en tiempo real**. Se usa para: conversación hablada, interpretar peticiones ("ven aquí", "¿qué hiciste hoy?"), y decidir intenciones de alto nivel mediante *tool use* (`acercarse_a`, `expresar`, `recordar`, `cambiar_comportamiento`).
- El prompt incluye personalidad, emoción actual, necesidades y memorias relevantes recuperadas.
- **"Sueño":** mientras carga de noche, un modelo más capaz consolida los episodios del día en memorias semánticas ("a Ana le gusta que la salude cuando llega", "los domingos juega conmigo").

**Memoria**
- *Episódica:* eventos con marca de tiempo, personas, emoción asociada.
- *Semántica:* hechos y preferencias consolidados, con embeddings para recuperación.
- *Privacidad:* todo vive en el robot; nada de audio/vídeo crudo se guarda ni se sube. Solo transcripciones y resúmenes. Backup cifrado opcional.

### 3.6 Flujo de conversación (presupuesto de latencia)
```
Wake word / mirada + voz ─► VAD ─► STT local en streaming ─► LLM (streaming) ─► TTS (streaming) ─► altavoz + mandíbula
     ~0 ms                  ~200 ms     ~300 ms                   ~400 ms TTFT       ~200 ms
                                          Objetivo: < 1.5 s hasta el primer sonido
```
Mientras espera, expresión inmediata local (orejas hacia ella, parpadeo, "mrrp?") para que nunca parezca congelado.

---

## 4. Estructura del repositorio (monorepo)

```
MangaAI/
├── docs/                    # arquitectura, ADRs, guías de montaje
├── hardware/
│   ├── cad/                 # modelos fuente y STL/STEP exportados
│   ├── electronics/         # esquemas (KiCad), BOM, presupuesto de masa
│   └── bom.csv
├── firmware/
│   └── teensy/              # PlatformIO + micro-ROS
├── ros2_ws/src/
│   ├── manga_description/   # URDF/MJCF, mallas
│   ├── manga_hardware/      # ros2_control hardware interface (bus Feetech)
│   ├── manga_control/       # locomoción RL, CPG, mezclador de animaciones
│   ├── manga_perception/    # visión, audio, tacto → eventos
│   ├── manga_expression/    # ojos, sonido, ronroneo, voz
│   ├── manga_mind/          # emociones, necesidades, BT, memoria, cliente LLM
│   ├── manga_interfaces/    # mensajes/servicios/acciones propios
│   └── manga_bringup/       # launch files y configuración
├── sim/
│   ├── mujoco/              # escenas y modelo
│   └── training/            # entrenamiento RL (corre en el PC con GPU)
├── assets/
│   ├── sounds/              # maullidos, ronroneos
│   ├── eyes/                # texturas/shaders de ojos
│   └── animations/          # clips keyframe
├── apps/dashboard/          # panel web (FastAPI)
└── docker/                  # imágenes de desarrollo y del Jetson
```

---

## 5. Hoja de ruta

| Fase | Objetivo | Resultado visible |
|---|---|---|
| **0. Fundamentos** | Repo, Docker, ROS 2, modelo MuJoCo inicial, presupuesto de masa, comprar 2–3 servos de prueba | Gato simulado de pie en MuJoCo |
| **1. Cabeza viva** | Cabeza + cuello + ojos + orejas + altavoz + micrófonos en un soporte | Te mira, sigue tu cara, parpadea, maúlla, gira hacia tu voz |
| **2. Mente v1** | Emociones, necesidades, behavior tree, tacto, ronroneo | Reacciona a caricias, se aburre, pide atención |
| **3. Voz** | Pipeline STT → Claude → TTS + memoria | Conversa y recuerda cosas |
| **4. Cuerpo y patas** | Chasis completo, 12 DOF, poses estáticas, animaciones (sentarse, estirarse) | Gato completo que cambia de postura |
| **5. Caminar** | Entrenar RL en sim, sim2real, CPG de respaldo | Camina hacia ella |
| **6. Integración** | Navegación en casa, base de carga, rutinas diarias | Vida autónoma diaria |
| **7. Pulido** | Piel, reducción de ruido, personalidad afinada | Se siente como un gato real |

La cabeza va primero a propósito: es donde vive el ~80 % de la percepción de "está vivo", y desarrolla percepción + mente en paralelo mientras se resuelve la parte mecánica más difícil.

---

## 6. Riesgos principales

| Riesgo | Mitigación |
|---|---|
| Peso/torque insuficiente para caminar | Presupuesto de masa estricto; validar torque en sim antes de imprimir; probar 1 pata real temprano |
| Sim2real de la locomoción | Domain randomization, identificar parámetros reales del servo (latencia, holgura), CPG de respaldo |
| Ruido de servos | Medir antes de comprar; perfiles de velocidad suaves; aislamiento acústico en la piel |
| Calor en Jetson y servos dentro del peluche | Flujo de aire planificado, monitoreo de temperatura, modo de reposo agresivo |
| Autonomía de batería corta | Dormir mucho (¡como un gato real!), base de carga |
| Seguridad física | Torque limitado, watchdog, sin puntos de pellizco expuestos, e-stop |
| Dependencia de internet | Todo lo esencial funciona offline; la nube solo enriquece |
| Privacidad | Sin almacenamiento de audio/vídeo crudo, memoria local, control desde el panel |

---

## 7. Decisiones abiertas
- [ ] Feetech STS3250 vs Dynamixel para las patas (tras medir ruido y holgura).
- [ ] Cámara monocular vs OAK-D Lite (profundidad).
- [ ] Nombre del gato / wake word.
- [ ] Voz: ¿habla con voz "humana", estilizada, o mezcla de maullidos y palabras?
- [ ] Raza/apariencia de referencia (define proporciones del CAD).
