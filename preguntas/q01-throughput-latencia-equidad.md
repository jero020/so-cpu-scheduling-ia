# Q1. La Tensión Clásica: Throughput vs. Latencia vs. Equidad (Fairness)

> Argumente por qué es teóricamente imposible maximizar simultáneamente el rendimiento global (throughput), la latencia de respuesta y la justicia distributiva. ¿Cómo debe un arquitecto de sistemas parametrizar el planificador según la naturaleza del negocio (batch processing vs. streaming interactivo)?

**Responsable:** Jerónimo Vélez Acosta · **Estado:** ☑ Prompt ☑ Respuesta IA ☑ Crítica individual ☑ Solución unificada

---

## 1. Prompt a la IA

**Modelo usado:** Claude Opus 5.5 (Anthropic), 2026-11-20

```text
Actúa como arquitecto senior de sistemas operativos con experiencia en el planificador del kernel de Linux y en sistemas de producción.

Contexto: prueba final de un curso universitario de Sistemas Operativos (Ingeniería de Sistemas). Responde con rigor técnico, citando mecanismos reales (nombres de políticas, syscalls, parámetros del kernel y versiones cuando aplique).

Pregunta: La Tensión Clásica: Throughput vs. Latencia vs. Equidad (Fairness). Argumente por qué es teóricamente imposible maximizar simultáneamente el rendimiento global (throughput), la latencia de respuesta y la justicia distributiva. ¿Cómo debe un arquitecto de sistemas parametrizar el planificador según la naturaleza del negocio (batch processing vs. streaming interactivo)?

Restricciones:
- Máximo 600 palabras.
- Explica al menos dos trade-offs con su costo concreto.
- Incluye al menos un ejemplo de sistema real en producción.
- Indica explícitamente qué supuestos de hardware estás haciendo.
- Propón parámetros concretos del planificador de Linux para cada perfil (batch vs. interactivo) y explica su efecto.
```

## 2. Respuesta de la IA

**Por qué no se pueden maximizar los tres a la vez**

Los tres objetivos compiten por el mismo recurso, el tiempo de CPU, y cada uno empuja un parámetro del planificador en dirección contraria:

- **Throughput** se maximiza con *time slices* largos: menos cambios de contexto, cachés y TLB calientes, menos overhead del planificador. En el límite, FCFS sin preempción tiene el menor overhead posible.
- **Latencia** se minimiza con *slices* cortos y preempción agresiva: una tarea interactiva que despierta debe correr ya, aunque eso interrumpa a otra que estaba aprovechando la caché.
- **Equidad** exige repartir el CPU en proporción a pesos, aunque la tarea “justa” sea la menos eficiente de correr ahora (por ejemplo, una tarea que acaba de migrar y tiene la caché fría).

Formalmente es un problema multiobjetivo sin un óptimo único: existe una frontera de Pareto. SJF/SRTF minimiza el tiempo medio de espera pero puede causar inanición (falla en equidad); Round Robin con quantum pequeño acota la latencia pero aumenta el overhead (falla en throughput). Por la ley de Little (L = λW), si el sistema está cerca de la saturación, cualquier mejora de latencia para una clase de tareas se paga con espera de otra.

**Trade-offs con costo concreto**

1. *Slice corto vs. throughput*: un cambio de contexto directo cuesta del orden de 1–5 µs, pero el costo indirecto (cachés L1/L2 y TLB que se recalientan) puede ser decenas de µs. Con un slice de 1 ms, ese costo puede representar varios puntos porcentuales de CPU perdidos.
2. *Preempción por despertar vs. equidad*: favorecer a la tarea que despierta (wakeup preemption) mejora la respuesta interactiva, pero penaliza a las tareas CPU-bound que llevan más tiempo esperando.

**Parametrización según el negocio (Linux)**

Desde Linux 6.6 el planificador por defecto es **EEVDF**, que reemplazó a CFS. Ambos reparten según pesos derivados de `nice`.

- *Batch (ETL, entrenamiento, render)*: usar `SCHED_BATCH` (el planificador asume que son CPU-bound y evita preemptarlas por despertares), valores `nice` positivos, y un `base_slice_ns` mayor (en `/sys/kernel/debug/sched/`). Objetivo: throughput.
- *Streaming interactivo / baja latencia*: mantener `SCHED_OTHER` con `nice` negativo para los hilos críticos, o `SCHED_FIFO` con prioridad moderada para el hilo de audio/video; desde Linux 6.12 una tarea puede pedir un slice más corto mediante `sched_attr.sched_runtime` en `sched_setattr(2)`. Mover el trabajo de fondo a `SCHED_IDLE`.
- *Mixto*: separar las cargas con cgroups v2 (`cpu.weight`) para que la equidad sea entre grupos y no entre hilos.

**Ejemplo real**: los servidores de audio de baja latencia (PipeWire/JACK) usan hilos de tiempo real vía `rtkit`, mientras que clusters de Hadoop/Spark corren sus ejecutores como procesos batch con `nice` alto.

**Supuestos de hardware**: CPU x86-64 multinúcleo SMP con cachés privadas L1/L2 y L3 compartida, sin NUMA relevante y sin núcleos heterogéneos.

## 3. Análisis crítico individual

### Jerónimo Vélez Acosta

- **Aciertos y cumplimiento:** La respuesta cumple con las restricciones de extensión, supuestos de hardware y parametrización en Linux, introduciendo acertadamente **EEVDF** (Linux ≥ 6.6) y la interfaz `sched_setattr(2)` de Linux 6.12 para slices dinámicos. Asimismo, fundamenta matemáticamente la imposibilidad mediante la frontera de Pareto y la Ley de Little (L = λW).
- **Vacíos y puntos críticos:**
  - *Parametrización de EEVDF incompleta para producción:* El modelo indica correctamente que `base_slice_ns` se ajusta en `/sys/kernel/debug/sched/base_slice_ns`, pero no advierte que esa ruta es de debugfs: no es una interfaz estable del kernel (no tiene un sysctl equivalente), requiere privilegios de root y en muchos entornos de producción debugfs está restringido o no se monta por seguridad. Además, el valor es global para todo el sistema. Para ajustar el *slice* de forma controlada, la alternativa estable es por tarea, mediante `sched_attr.sched_runtime` en `sched_setattr(2)` (Linux ≥ 6.12), lo cual la IA sí menciona pero sin presentarlo como la opción preferible frente a debugfs.
  - *Omitió el costo de coherencia de caché:* Al analizar el costo indirecto del cambio de contexto, se limita a las cachés privadas L1/L2 y el TLB, pasando por alto el tráfico de coherencia (protocolos MESI/MOESI) e invalidación de líneas en la caché L3 compartida cuando el planificador balancea carga entre núcleos.
  - *Riesgo en la recomendación de `SCHED_FIFO`:* Sugerir `SCHED_FIFO` para streaming interactivo sin advertir explícitamente sobre el mecanismo de protección `kernel.sched_rt_runtime_us` (950 ms por defecto) es riesgoso; un hilo mal comportado bajo `SCHED_FIFO` puede causar inanición total al resto del espacio de usuario.

## 4. Solución unificada del equipo

### Postura del equipo

No existe un planificador que maximice a la vez throughput, latencia y equidad: son objetivos en tensión y el arquitecto debe elegir un punto de la frontera de Pareto según el negocio. La respuesta de la IA es correcta en lo conceptual, pero hay que completarla en dos puntos: el costo de coherencia de caché entre núcleos y los riesgos de usar políticas de tiempo real para cargas interactivas.

### Justificación

- El costo de un *slice* corto no se limita a recalentar L1/L2 y TLB. Cuando el balanceador migra tareas entre núcleos, también aparece tráfico de coherencia (MESI/MOESI) e invalidación de líneas en la L3 compartida, así que la penalización real de favorecer la latencia es mayor que la que estima la IA.
- Por la ley de Little (L = λW), cerca de la saturación cualquier mejora de latencia para una clase de tareas aumenta la espera de otra.
- `SCHED_FIFO` acota la latencia, pero un hilo mal comportado puede dejar sin CPU al resto del sistema. La única protección por defecto es el *RT throttling* (`kernel.sched_rt_runtime_us = 950000` sobre `sched_rt_period_us = 1000000`), que reserva un 5 % del CPU a las tareas normales.

### Aplicación en sistemas de producción

| Perfil | Configuración recomendada (Linux ≥ 6.6, EEVDF) |
|---|---|
| Batch (ETL, render, entrenamiento) | `SCHED_BATCH`, `nice` positivo y, si se requiere, un *slice* base mayor en `/sys/kernel/debug/sched/base_slice_ns` (solo vía debugfs). |
| Interactivo / streaming | `SCHED_OTHER` con `nice` negativo; desde Linux 6.12, *slice* más corto por tarea con `sched_attr.sched_runtime` en `sched_setattr(2)`. `SCHED_FIFO` solo para hilos acotados (audio) y con el *RT throttling* activo. |
| Mixto | Separar cargas en cgroups v2 con `cpu.weight`, para que la equidad sea entre servicios y no entre hilos. |

### Referencias

- Documentación del kernel: `scheduler/sched-design-CFS`, `scheduler/sched-eevdf`, `scheduler/sched-rt-group`.
- Silberschatz, *Operating System Concepts*, capítulo de planificación de CPU.
