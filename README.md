# so-cpu-scheduling-ia

CFS, tiempo real, inversión de prioridades, big.LITTLE, afinidad y cgroups: análisis crítico de las respuestas de la IA sobre planificación de procesos.

**Prueba Final de Sistemas Operativos (EAFIT)** · Eje 2: Planificación y Gestión de CPU (Process & CPU Scheduling)

## Integrantes

| Integrante | Nombre | Preguntas a cargo |
|---|---|---|
| A | | |
| B | | |
| C | | |
| D | | |
| E | | |

## Cómo está organizada la entrega

Cada pregunta tiene su archivo en [`preguntas/`](preguntas) con los cuatro componentes obligatorios:

1. **Prompt a la IA**: texto literal enviado al modelo.
2. **Respuesta de la IA**: salida sin alterar los conceptos clave.
3. **Análisis crítico individual**: una sección por integrante.
4. **Solución unificada del equipo**: postura técnica definitiva.

El formato vacío está en [`plantilla.md`](plantilla.md) y el plan de trabajo en [`docs/`](docs).

## Preguntas

| # | Pregunta | Responsable | Lista |
|---|---|---|---|
| Q1 | [La Tensión Clásica: Throughput vs. Latencia vs. Equidad (Fairness)](preguntas/q01-throughput-latencia-equidad.md) | _(nombre)_ | ☐ |
| Q2 | [Equidad Absoluta vs. Prioridad Económica (CFS vs. Cuotas Empresariales)](preguntas/q02-cfs-vs-cuotas.md) | _(nombre)_ | ☐ |
| Q3 | [Sistemas de Tiempo Real Duro vs. Blando: Filosofía y Resiliencia](preguntas/q03-tiempo-real-duro-blando.md) | _(nombre)_ | ☐ |
| Q4 | [La Paradoja de la Inversión de Prioridades](preguntas/q04-inversion-prioridades.md) | _(nombre)_ | ☐ |
| Q5 | [Planificación Heterogénea en la Era del Silicio Diverso (big.LITTLE / DynamIQ)](preguntas/q05-planificacion-heterogenea.md) | _(nombre)_ | ☐ |
| Q6 | [El Coste Oculto de la Multitarea Preemptiva](preguntas/q06-costo-multitarea-preemptiva.md) | _(nombre)_ | ☐ |
| Q7 | [Afinidad de CPU (Affinity) vs. Flexibilidad Global](preguntas/q07-afinidad-cpu.md) | _(nombre)_ | ☐ |
| Q8 | [Inanición (Starvation) y Envejecimiento (Aging): Ética y Diseño Algorítmico](preguntas/q08-inanicion-envejecimiento.md) | _(nombre)_ | ☐ |
| Q9 | [Aislamiento de Cómputo en Contenedores: Shares Relativos vs. Hard Limits](preguntas/q09-shares-vs-hard-limits.md) | _(nombre)_ | ☐ |
| Q10 | [Planificación Asistida por Telemetría e Inteligencia Artificial](preguntas/q10-planificacion-ia.md) | _(nombre)_ | ☐ |

## Estructura

```
so-cpu-scheduling-ia/
├── README.md
├── plantilla.md      # formato de 4 componentes
├── preguntas/        # q01 … q10
├── diagramas/        # imágenes o fuentes de diagramas
├── fuentes.md        # bibliografía para validar a la IA
├── docs/             # plan de trabajo
└── entrega/          # documento final exportado
```
