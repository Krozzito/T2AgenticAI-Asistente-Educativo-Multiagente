# Desarrollo de un Asistente Educativo Personal mediante un Sistema Multiagente

Sistema multiagente local que procesa consultas académicas en lenguaje natural, las
delega a un agente especialista y entrega una respuesta integrada validada por un agente
crítico. Todo corre de forma local mediante **Ollama**, sin APIs de pago.

> Entrega 2 — Agentic AI · Universidad De Concepción · 2026

---

## Descripción

En lugar de saltar entre una calculadora, un buscador y un calendario, este asistente
recibe solicitudes en lenguaje natural y las resuelve mediante agentes coordinados:

- *"¿Cuál es la raíz de 64?"* → cálculo exacto con una herramienta de Python.
- *"Agrega un examen de Álgebra el 8 de junio de 2026 a las 09:00."* → evento persistido en un calendario local.
- *"¿Cuáles son las fases de la luna?"* → búsqueda web + explicación sintetizada.

Un LLM por sí solo no ejecuta operaciones exactas, no persiste datos ni busca en la web.
El sistema multiagente resuelve esto asignando roles y herramientas específicas a cada agente.

---

## Arquitectura

El sistema se compone de **5 agentes** y **3 herramientas**, orquestados con LangGraph:

| Agente | Rol | Herramienta | Modelo |
|---|---|---|---|
| **Coordinador** | Recibe la consulta y delega al especialista adecuado | — | `qwen3:8b` |
| **Calculador** | Resuelve operaciones matemáticas exactas | `evaluar` (AST seguro) | `qwen3:8b` |
| **Organizador** | Gestiona eventos en un calendario persistente | `agregar/listar/modificar_evento` | `gemma4:e2b` |
| **Experto** | Responde dudas conceptuales | `buscar` (DuckDuckGo) | `qwen3:8b` |
| **Crítico** | Evalúa la respuesta; si no es válida, reenvía feedback | — | `llama3.2:3b` |

Flujo general:

```
usuario → Coordinador → [Calculador | Organizador | Experto] → Crítico → ¿válida?
              ↑                                                    ├─ sí → respuesta
              └──────────────── feedback (reintenta) ──────────────┘ no
```

El Coordinador y el Crítico usan **salida estructurada** (Pydantic) para decisiones
fiables. El Crítico corre en un modelo sin razonamiento (`llama3.2:3b`) para evitar que
la cadena de pensamiento rompa el parseo del JSON.

*El diagrama detallado se genera desde el código con la función `imprimirGrafo(appfinish)`
(usa `app.get_graph(xray=True).draw_mermaid_png()`).*

---

## Stack tecnológico

- **Ollama** — motor local para ejecutar los modelos.
- **LangGraph** — orquestación de los agentes mediante grafos de estado y composición de subgrafos.
- **Python 3.10+**
- **Modelos:** `gemma4:e2b`, `qwen3:8b`, `llama3.2:3b`.

---

## Instalación y configuración

### 1. Instalar Ollama

```bash
apt-get update && apt-get install -y zstd
curl -fsSL https://ollama.com/install.sh | sh
```

### 2. Levantar el servidor y descargar los modelos

```bash
ollama serve &        # inicia el servidor en segundo plano
ollama pull gemma4:e2b
ollama pull qwen3:8b
ollama pull llama3.2:3b
```

### 3. Dependencias de Python

```bash
pip install -r requirements.txt
```

Contenido de `requirements.txt`:

```
langgraph
langchain-ollama
langchain-core
ddgs
pydantic
```

> **Nota sobre la búsqueda web:** el paquete es `ddgs` (antes `duckduckgo-search`, ahora deprecado).

### Nota para Google Colab

El proyecto fue desarrollado en Colab. El entorno es **efímero**: al reconectarse se pierde
todo, por lo que hay que reinstalar Ollama, relanzar `ollama serve`, repullar los modelos y
re-ejecutar las celdas. Las celdas de instalación están al inicio del cuaderno para re-correrlas
de un saque.

---

## Uso

1. Ejecutar las celdas de instalación y configuración.
2. Ejecutar las celdas que definen herramientas, agentes y el grafo (`appfinish`).
3. Invocar el sistema:

```python
entrada = {
    "messages": [("user", "¿Cuánto es la raíz de 64?")],
    "siguiente": "",
    "valida": False,
    "feedback": "",
    "intentos": 0,
}

for evento in appfinish.stream(entrada, stream_mode="updates"):
    print(evento)
    print("---")
```

La salida imprime la **traza de ejecución**: delegación del coordinador, invocación de
herramientas, evaluación del crítico y respuesta final.

---

## Estructura del repositorio

```
.
├── README.md
├── requirements.txt
├── calendario.json            # base de datos del calendario (persistencia local)
├── Tarea_Agentic_AI.ipynb     # cuaderno principal
└── [informe.pdf]              # informe del proyecto
```

---

## Dataset de prueba

El sistema se evaluó con consultas que cubren los tres agentes satélite (cálculo, calendario
y búsqueda). Las trazas de ejecución exitosas y el diagnóstico de fallos se documentan en el
informe.


## Autor

Cristóbal Figueroa Burgos — crfigueroa2021@udec.cl/krozzito — 2026
