# 07. Plan de Implementación: Prototipo 2D "Aldea Mínima Viable" (MVP)

Este documento define la hoja de ruta práctica para construir una prueba de concepto jugable en 2D que valide la arquitectura de **SNA**. El objetivo no es crear un juego comercial completo de inmediato, sino un **laboratorio visual de simulación de vida emergente** con 5 personajes en un mapa cerrado.

---

## 1. Alcance y Escenario de la Aldea Mínima

### El Escenario Físico (Mapa 2D Tilemap de 32x24 celdas)
Un pequeño asentamiento rural con 4 Puntos de Interés (POIs) funcionales:

```
┌────────────────────────────────────────────────────────────────────────┐
│                        MAPA 2D: ALDEA DEL RÍO                         │
│                                                                        │
│   [Granja de Trigo] 🌾                       [Herrería de Bruno] ⚒️    │
│   (3 puestos de cosecha)                     (Yunque, horno y carbón)  │
│                                                                        │
│                 ═════════════[Camino Central]═════════════             │
│                                                                        │
│   [Casa Compartida] 🛏️                       [Taberna El Jabalí] 🍺    │
│   (Camas para descanso nocturno)             (Mesas, barra y ocio)     │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### Los 5 Habitantes del Experimento:
1. **Mateo (El Tabernero):** Casado con Elena. Rasgo: *Rencoroso, Celoso*. Pasa las tardes en la taberna y la noche en casa.
2. **Elena (La Curandera):** Casada con Mateo. Rasgo: *Sociable, Empática*. Recolecta hierbas de día y visita la taberna al atardecer.
3. **Bruno (El Herrero):** Soltero. Rasgo: *Solitario, Apasionado*. Siente una atracción secreta no correspondida hacia Elena.
4. **Tomás (El Granjero):** Soltero. Rasgo: *Chismoso, Extrovertido*. Trabaja la granja; cuando ve interactuar a otros, lo divulga en la taberna.
5. **Clara (La Mercader):** Rasgo: *Materialista, Observadora*. Compra trigo a Tomás y herramientas a Bruno para vender suministros.

### El "Test de Turing Emergente" (La prueba del drama):
El prototipo se considerará exitoso si, sin ningún guión programado:
1. Tomás ve a Elena y Bruno hablando cerca de la herrería.
2. Tomás se lo cuenta a Mateo mientras pide una cerveza en la taberna.
3. El grafo social de Mateo actualiza su desconfianza hacia Elena y odio hacia Bruno.
4. Cuando Elena entra a la taberna, Mateo le lanza un reproche sarcástico generado por el SLM local coherente con la situación.

---

## 2. Pila Tecnológica Recomendada para Prototipado Rápido

Para iterar a máxima velocidad con cero fricción de compilación:

| Capa | Tecnología Seleccionada | Justificación |
| :--- | :--- | :--- |
| **Lenguaje Base** | **Python 3.11+** | Máxima velocidad de desarrollo asistido por IA, excelente ecosistema para IA/LLMs. |
| **Motor 2D / Render** | **Pygame-CE** o **Arcade** | Renderizado 2D directo, sprites simples, gestión de eventos a 60 FPS sin sobrecarga. |
| **Simulación y Estado** | **ECS Ligero (esper/in-house) + SQLite** | Separación limpia de datos en memoria para necesidades y memoria persistente. |
| **Motor de Inferencia SLM** | **`llama-cpp-python` / Ollama local** | Ejecución en local de **Qwen 2.5 0.5B / 1.5B (GGUF 4-bit)** usando CPU o GPU DirectML. |
| **Navegación** | **Pathfinding $A^*$ en grid + 2-opt TSP** | Algoritmo clásico en cuadrícula 2D, ligero e instantáneo. |

---

## 3. Desglose de Fases, Tareas y Estimaciones de Tiempo

A continuación se compara el tiempo estimado en **Desarrollo Tradicional en solitario** frente a **Desarrollo Asistido por IA Generativa / Antigravity**.

### FASE 1: El Tablero y el Bucle Físico (El Cuerpo)
*Construcción del mapa 2D, bucle de juego a 60 FPS y movimiento de personajes.*

| Tarea | Descripción Técnica | Dev Tradicional | Con IA Asistida |
| :--- | :--- | :---: | :---: |
| **1.1 Entorno y Tilemap** | Cuadrícula 2D con renderizado de tiles (hierba, caminos, paredes de POIs). | 6 horas | 1.5 horas |
| **1.2 Componentes ECS de Necesidades** | Structs para Hambre, Energía y Diversión con decaimiento continuo y curvas de utilidad. | 8 horas | 2.0 horas |
| **1.3 Navegación $A^*$ y Rutinas** | Pathfinding sobre cuadrícula para viajar entre POIs según la hora del día. | 10 horas | 2.5 horas |
| **1.4 Optimizador TSP 2-opt** | Algoritmo para secuenciar 3 o 4 tareas de granja/recolección en el orden más corto. | 6 horas | 1.5 horas |
| **Subtotal Fase 1** | | **30 horas** | **7.5 horas** |

---

### FASE 2: La Red Social y los Sentidos (Las Relaciones)
*Percepción sensorial de los NPCs y actualización del grafo relacional.*

| Tarea | Descripción Técnica | Dev Tradicional | Con IA Asistida |
| :--- | :--- | :---: | :---: |
| **2.1 Sensores de Visión/Proximidad** | Detección espacial de qué NPCs u objetos están en el campo de visión de cada agente. | 6 horas | 1.5 horas |
| **2.2 Estructura del Grafo Social** | Matriz de adyacencia dirigida en memoria: Afinidad, Confianza y Romance entre los 5 personajes. | 8 horas | 2.0 horas |
| **2.3 Motor de Cotilleos (Gossip System)** | Intercambio de paquetes de información entre NPCs en el mismo POI con distorsión por antipatía. | 12 horas | 3.0 horas |
| **2.4 Disparadores de Colapso / Reacción** | Fórmulas que alteran el ánimo y detonan interrupciones (ej. confrontación al cruzar miradas). | 8 horas | 2.0 horas |
| **Subtotal Fase 2** | | **34 horas** | **8.5 horas** |

---

### FASE 3: La Voz y la Mente (Integración del SLM Local)
*Conexión asíncrona del modelo de lenguaje para diálogos y pensamientos.*

| Tarea | Descripción Técnica | Dev Tradicional | Con IA Asistida |
| :--- | :--- | :---: | :---: |
| **3.1 Worker Thread de Inferencia** | Hilo desacoplado en segundo plano con cola de prioridad para no congelar los 60 FPS de Pygame. | 10 horas | 2.5 horas |
| **3.2 Ensamblador de Prompts Breves** | Inyector de contexto dinámico (OCEAN + estado de necesidades + relación + último recuerdo en < 250 tokens). | 8 horas | 2.0 horas |
| **3.3 Validador JSON de Salida** | Parser estricto para extraer la frase de diálogo y la variación emocional sin errores de sintaxis. | 6 horas | 1.5 horas |
| **3.4 Bocadillos de Diálogo en Pantalla** | Renderizado de burbujas de texto temporales sobre los sprites cuando interactúan. | 6 horas | 1.5 horas |
| **Subtotal Fase 3** | | **30 horas** | **7.5 horas** |

---

### FASE 4: La Memoria y el Ciclo Día/Noche
*Persistencia episódica, sueño y consolidación reflexiva.*

| Tarea | Descripción Técnica | Dev Tradicional | Con IA Asistida |
| :--- | :--- | :---: | :---: |
| **4.1 Búfer de Recuerdos Episódicos** | Registro en SQLite en memoria de eventos relevantes presenciados durante el día. | 8 horas | 2.0 horas |
| **4.2 Ciclo Día/Noche y Reloj Global** | Transición horaria visual (iluminación diurna/nocturna) y llamada a la cama. | 4 horas | 1.0 hora |
| **4.3 Fase de Consolidación Nocturna** | Batch nocturno donde el SLM sintetiza los eventos del día en una opinión antes de dormir. | 10 horas | 2.5 horas |
| **Subtotal Fase 4** | | **22 horas** | **5.5 horas** |

---

### FASE 5: Panel de Telemetría e Inspección (El "Inspector de Mentes")
*Herramienta visual para que el desarrollador/jugador vea la IA en directo.*

| Tarea | Descripción Técnica | Dev Tradicional | Con IA Asistida |
| :--- | :--- | :---: | :---: |
| **5.1 Interfaz de Selección de NPC** | Clic con ratón sobre un personaje para abrir su ficha lateral. | 4 horas | 1.0 hora |
| **5.2 Visualizador de Necesidades y Emociones** | Barras dinámicas de Hambre, Sueño, Estrés y posición en espacio PAD. | 4 horas | 1.0 hora |
| **5.3 Visor del Grafo Social y Memorias** | Lista de opiniones hacia los otros 4 personajes y los 3 recuerdos más recientes. | 6 horas | 1.5 horas |
| **Subtotal Fase 5** | | **14 horas** | **3.5 horas** |

---

## 4. Resumen Global de Tiempos y Esfuerzo

```
┌────────────────────────────────────────────────────────────────────────┐
│ TOTAL PROYECTO COMPLETO (MVP 2D ALDEA):                                │
│                                                                        │
│ • Desarrollo Tradicional:     130 horas (aprox. 3.5 a 4 semanas)       │
│ • Desarrollo Asistido por IA:  32.5 horas (aprox. 4 a 5 días de trabajo)│
│                                                                        │
│ REDUCCIÓN DE TIEMPO ESTIMADA: ~75% de ahorro en desarrollo             │
└────────────────────────────────────────────────────────────────────────┘
```

> [!TIP]
> **Estrategia de Ejecución Iterativa:**  
> Se puede tener un **"Hito 0 Funcional" (Fase 1 + 2 básica)** listo en unas **10-12 horas de trabajo asistido**, donde ya se ve a los monigotes recorrer el mapa, comer, dormir y enfadarse entre ellos mediante iconos, antes de enchufar el modelo de lenguaje.

---

## 5. Estructura de Archivos Proyectada para la Implementación

Cuando se proceda a codificar, el código se organizará bajo esta estructura modular:

```
sentient-npc-architecture/
├── docs/                             # Documentación de diseño (existente)
│   └── 07-mvp-village-implementation-plan.md
├── src/
│   ├── core/
│   │   ├── ecs.py                    # Gestor de entidades y componentes
│   │   ├── time_manager.py           # Reloj del juego y ciclo día/noche
│   │   └── event_bus.py              # Sistema de eventos global
│   ├── simulation/
│   │   ├── needs.py                  # Curvas de utilidad fisiológica
│   │   ├── navigation.py             # A* y TSP 2-opt
│   │   └── social_graph.py           # Grafo dirigido y propagación de rumores
│   ├── ai/
│   │   ├── slm_worker.py             # Inferencia asíncrona local (llama.cpp)
│   │   ├── prompt_builder.py         # Ensamblador contextual de prompts
│   │   └── memory_db.py              # SQLite para memorias y consolidación
│   ├── view/
│   │   ├── renderer.py               # Renderizado 2D de tiles y sprites
│   │   └── ui_inspector.py           # Panel lateral de telemetría del NPC
│   └── main.py                       # Punto de entrada de la aplicación
└── requirements.txt                  # Dependencias mínimas (pygame-ce, llama-cpp-python)
```
