# 05. Pipeline de Memoria y Cognición

Este documento detalla el sistema de almacenamiento, puntuación de recuerdos y consolidación reflexiva de **SNA**. El objetivo es proporcionar a los NPCs una memoria coherente y acumulativa de su pasado sin desbordar la memoria RAM ni ralentizar el tiempo de respuesta del modelo de lenguaje.

---

## 1. La Jerarquía de Memoria de Tres Niveles

Para imitar la cognición humana sin saturar los recursos del sistema, la memoria se divide en tres niveles de persistencia y abstracción:

```
┌────────────────────────────────────────────────────────────────────────┐
│                   NIVEL 1: BÚFER SENSORIAL INMEDIATO                   │
│  * Almacenamiento: Ring Buffer circular en RAM (últimos 30 eventos)    │
│  * Contenido: Percepciones brutas recientes (< 2 minutos)              │
│  * Coste: 0 CPU / Nanosegundos                                         │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ Filtro de Importancia (Umbral > 40)
┌───────────────────────────────────▼────────────────────────────────────┐
│                   NIVEL 2: MEMORIA EPISÓDICA DEL DÍA                   │
│  * Almacenamiento: Tabla SQLite en memoria / Structs contiguos         │
│  * Contenido: Sucesos significativos ocurridos en la jornada actual    │
│  * Coste: Microsegundos                                                │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ Consolidación Nocturna (Durante el Sueño)
┌───────────────────────────────────▼────────────────────────────────────┐
│                   NIVEL 3: MEMORIA SEMÁNTICA Y REFLEXIONES             │
│  * Almacenamiento: Vector DB local ligera / SQLite con sqlite-vss      │
│  * Contenido: Creencias, opiniones consolidadas y lecciones de vida    │
│  * Coste: Consulta bajo demanda (RAG < 5ms)                            │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Estructura de un Registro de Recuerdo (*Memory Node*)

Cada memoria episódica o semántica se almacena con la siguiente estructura optimizada:

```json
{
  "id": 1042,
  "timestamp": 12840.5,
  "tipo": "EPISODICA",
  "descripcion": "El jugador blandió una espada ensangrentada frente a mi tienda",
  "sujeto_involucrado": "player_id",
  "carga_emocional": -0.85,
  "importancia_intrinseca": 90,
  "etiquetas_semanticas": ["peligro", "crimen", "jugador", "violencia"],
  "vector_embedding": [0.124, -0.052, 0.441, "...", -0.219]
}
```

* **Importancia Intrínseca ($[0, 100]$):** Asignada automáticamente según reglas fijas:
  * Rutina habitual (ej. "Comió estofado"): $10$.
  * Evento social positivo (ej. "Elena le regaló flores"): $60$.
  * Evento traumático o violento (ej. "Vio un robo a mano armada"): $95$.

---

## 3. Función de Puntuación de Recuperación (*Retrieval Scoring*)

Cuando un NPC debe hablar o tomar una decisión crítica ante una consulta o situación $q$, no se inyectan todos sus recuerdos. Se evalúa una función de recuperación inspirada en la investigación de Stanford, adaptada para ejecución ultrarrápida:

$$\text{Score}(m, q) = w_r \cdot \text{Recencia}(m) + w_i \cdot \text{Importancia}(m) + w_s \cdot \text{Relevancia}(m, q)$$

### Componentes de la Puntuación:
1. **Recencia:** Decaimiento exponencial en función del tiempo transcurrido desde el evento:
   $$\text{Recencia}(m) = e^{-\lambda \cdot (t_{\text{actual}} - t_m)}$$
   * $\lambda$: Factor de olvido ($0.995$ por hora de juego).
2. **Importancia:** Puntuación intrínseca normalizada:
   $$\text{Importancia}(m) = \frac{\text{importancia\_intrinseca}}{100}$$
3. **Relevancia Semántica:** Similitud coseno entre el embedding de la situación actual y el embedding del recuerdo:
   $$\text{Relevancia}(m, q) = \frac{\mathbf{v}_m \cdot \mathbf{v}_q}{\|\mathbf{v}_m\| \|\mathbf{v}_q\|}$$

> [!TIP]
> **Optimización sin Embeddings para CPU Modesta:**  
> Si el dispositivo no cuenta con aceleración de embeddings, la similitud vectorial se sustituye por **coincidencia de etiquetas semánticas ponderadas con búsqueda BM25** sobre SQLite FTS5, reduciendo el cálculo a menos de **0.5 milisegundos**.

---

## 4. Consolidación Nocturna: La Fase de "Sueño y Reflexión"

El cuello de botella de los agentes con memoria infinita es que acumulan miles de registros intrascendentes. Para evitarlo, se implementa una **fase de sueño biológico**:

```mermaid
flowchart LR
    A["Recuerdos Episódicos del Día (30-50 eventos)"] --> B["Filtro de Ruido (Descartar I < 30)"]
    B --> C["Prompt de Síntesis Nocturna (SLM Batch)"]
    C --> D["1 o 2 Reflexiones Semánticas Permanentes"]
    D --> E["Almacén Semántico a Largo Plazo"]
    B -.->|"Purgar"| F["Basura / Liberar Memoria"]
```

### Proceso de Consolidación:
1. **Purgado de Ruido:** A las 02:00 AM del reloj de juego (cuando el NPC duerme), se eliminan automáticamente todos los recuerdos del búfer diario con $\text{Importancia} < 30$.
2. **Síntesis y Formación de Opiniones:** Los eventos relevantes restantes se agrupan y se envía una única petición por lotes al SLM local:
   * *Entrada:*
     * "Hoy Tomás me dijo que Elena y Bruno estaban en el bosque".
     * "Elena llegó tarde a casa y no quiso cenar".
     * "Bruno no me miró a los ojos cuando pasé frente a su forja".
   * *Salida Generada por el SLM:*
     * *"Reflexión: Ya no puedo confiar en Elena. Creo que me oculta algo con Bruno. Debo vigilar sus movimientos."*
3. **Persistencia:** La reflexión sintética se guarda permanentemente en la base de datos como una **Creencia/Opinión**. Los eventos atómicos diarios se borran, manteniendo la base de datos limpia y compacta.

---

## 5. El Pipeline de Ensamblaje Dinámico del Prompt

Para lograr respuestas ultra-rápidas con modelos de 1B a 3B parámetros, el prompt se ensambla en tiempo de ejecución combinando módulos estrictos para no superar los **300 tokens de contexto**:

```
┌─────────────────────────────────────────────────────────────┐
│ 1. IDENTIDAD Y ANCLAS LORE (Fijo en System Prompt)          │
│    "Eres Mateo, 35 años, tabernero. Desconfiado y orgulloso"│
├─────────────────────────────────────────────────────────────┤
│ 2. ESTADO BIOLÓGICO Y EMOCIONAL (Variables de Simulación)   │
│    "Estrés: 75/100 (Muy alto). Emoción: Furioso e inseguro" │
├─────────────────────────────────────────────────────────────┤
│ 3. CONTEXTO RELACIONAL (Extraído del Grafo Social)          │
│    "Interlocutor: Elena (Esposa). Confianza: 10/100"        │
├─────────────────────────────────────────────────────────────┤
│ 4. TOP 3 RECUERDOS RECUPERADOS (Por Puntuación)             │
│    - "Crees que Elena te engaña con Bruno (Reflexión ayer)" │
│    - "Viste a Bruno rondar tu calle anoche"                 │
├─────────────────────────────────────────────────────────────┤
│ 5. RESTRICCIÓN DE SALIDA (JSON Estricto)                    │
│    "Genera diálogo de máx. 2 frases y nueva intención"      │
└─────────────────────────────────────────────────────────────┘
```

### Formato de Salida Obligatorio:
El modelo debe responder exclusivamente en JSON para que el motor del juego procese el resultado sin necesidad de parsear lenguaje natural:

```json
{
  "dialogo": "Llegas tarde, Elena... ¿O es que el fuelle de la herrería necesitaba que alguien le diera aire?",
  "emocion_visual": "sarcasmo_amargo",
  "impacto_relacion_delta": {
    "confianza": -5,
    "afinidad": -2
  },
  "proxima_accion": "ignorar_y_limpiar_vasos"
}
```
