# Q3. Sistemas de Tiempo Real Duro vs. Blando: Filosofía y Resiliencia

> Contraste la filosofía determinista de los algoritmos de tiempo real (donde llegar tarde equivale a un fallo catastrófico) frente a los planificadores de propósito general. ¿Por qué agregar más potencia de CPU no convierte automáticamente un sistema estándar en uno de tiempo real?

**Responsable:** Jerónimo Vélez Acosta · **Estado:** ☑ Prompt ☑ Respuesta IA ☑ Crítica individual ☑ Solución unificada

---

## 1. Prompt a la IA

**Modelo usado:** Claude Opus 5.5 (Anthropic), 2026-11-20

```text
Actúa como arquitecto senior de sistemas operativos con experiencia en el planificador del kernel de Linux y en sistemas de producción.

Contexto: prueba final de un curso universitario de Sistemas Operativos (Ingeniería de Sistemas). Responde con rigor técnico, citando mecanismos reales (nombres de políticas, syscalls, parámetros del kernel y versiones cuando aplique).

Pregunta: Sistemas de Tiempo Real Duro vs. Blando: Filosofía y Resiliencia. Contraste la filosofía determinista de los algoritmos de tiempo real (donde llegar tarde equivale a un fallo catastrófico) frente a los planificadores de propósito general. ¿Por qué agregar más potencia de CPU no convierte automáticamente un sistema estándar en uno de tiempo real?

Restricciones:
- Máximo 600 palabras.
- Explica al menos dos trade-offs con su costo concreto.
- Incluye al menos un ejemplo de sistema real en producción.
- Indica explícitamente qué supuestos de hardware estás haciendo.
- Menciona las políticas de tiempo real disponibles en Linux y el estado de PREEMPT_RT.
```

## 2. Respuesta de la IA

**Dos filosofías distintas**

- Un **planificador de propósito general** optimiza el caso promedio: throughput, latencia media y equidad.
- Un **sistema de tiempo real** optimiza el **peor caso**: lo importante no es ser rápido sino que cada tarea termine antes de su *deadline*, siempre. En **tiempo real duro** (control de frenos, marcapasos, aviónica) perder un deadline es un fallo del sistema; en **tiempo real blando** (streaming, VoIP) se degrada la calidad pero el sistema sigue siendo útil.

Por eso los algoritmos de tiempo real (Rate Monotonic, EDF) vienen acompañados de **análisis de planificabilidad**: por ejemplo, con Rate Monotonic un conjunto de n tareas periódicas es planificable si la utilización es ≤ n(2^(1/n) − 1) (≈ 69 % para n grande); con EDF, si es ≤ 100 %.

**Por qué más CPU no convierte un sistema en tiempo real**

Más potencia reduce el tiempo **promedio**, pero el tiempo real depende de que el **peor caso esté acotado**. Las fuentes de latencia no acotada no desaparecen con más GHz:

1. *Secciones no preemptibles del kernel* y manejadores de interrupciones largos.
2. *Inversión de prioridades* en locks compartidos.
3. *Efectos de hardware*: fallos de caché, TLB, interferencia entre núcleos por la L3 compartida, SMIs (System Management Interrupts) invisibles para el SO y escalado dinámico de frecuencia.
4. *Paginación*: un fallo de página puede costar milisegundos si va a disco.

Un CPU más rápido con latencia de peor caso no acotada sigue sin poder garantizar deadlines.

**Mecanismos en Linux**

- Políticas `SCHED_FIFO` y `SCHED_RR` (prioridades 1–99) y `SCHED_DEADLINE` (desde Linux 3.14, basada en EDF + Constant Bandwidth Server, con control de admisión).
- **PREEMPT_RT**: hace preemptible casi todo el kernel (interrupciones en hilos, spinlocks convertidos en rt_mutex con herencia de prioridad). Se integró a la rama principal en **Linux 6.12 (2024)**.
- Prácticas complementarias: `mlockall()` para evitar paginación, `isolcpus`/`nohz_full` para aislar núcleos.

**Trade-offs**

1. *Determinismo vs. throughput*: PREEMPT_RT reduce la latencia máxima (típicamente a decenas de µs) a costa de algo de throughput por más preempción y locks más caros.
2. *Reserva de capacidad vs. eficiencia*: garantizar deadlines exige dimensionar por el peor caso (WCET), dejando CPU ociosa la mayor parte del tiempo.

**Ejemplo real**: controladores industriales y robots (por ejemplo con ROS 2) corren Linux con PREEMPT_RT; para tiempo real duro certificable (aviónica, automoción) se usan RTOS como VxWorks, QNX o Zephyr.

**Supuestos de hardware**: CPU embebida o x86 con temporizadores de alta resolución; sin hipervisor de por medio.

## 3. Análisis crítico individual

### Jerónimo Vélez Acosta

- **Aciertos y cumplimiento:** Contrasta eficazmente la optimización del caso promedio frente a la acotación del peor caso (WCET), incluye las cotas teóricas de planificabilidad de Liu & Layland para Rate Monotonic (≈ 69.3 %) y EDF (100 %), y ubica correctamente la fusión de **PREEMPT_RT** en Linux 6.12.
- **Vacíos y puntos críticos:**
  - *Simplificación de la cota del 100 % en EDF:* El modelo afirma que con EDF un conjunto de tareas periódicas es planificable si U ≤ 100 %, omitiendo las hipótesis estrictas bajo las cuales esto es cierto: tareas independientes (sin bloqueo por recursos compartidos), deadlines iguales a los períodos (Dᵢ = Tᵢ), sistema monoprocesador y overhead de cambio de contexto nulo. En multiprocesador (Global EDF), la utilización puede caer drásticamente debido a la anomalía de Dhall.
  - *Imprecisión en el ejemplo de producción:* Cita a **Zephyr** junto a VxWorks y QNX como ejemplos de RTOS para "tiempo real duro certificable (aviónica, automoción)". Aunque Zephyr es un RTOS robusto y cuenta con esfuerzos de certificación (como IEC 61508), en aviónica crítica (DO-178C DAL-A) o automoción (ISO 26262 ASIL-D) los estándares industriales consolidados son VxWorks 653, Integrity-178 o QNX Neutrino, más que Zephyr.

## 4. Solución unificada del equipo

### Postura del equipo

El tiempo real no es velocidad sino predictibilidad del peor caso. Más CPU reduce el promedio pero no acota la latencia máxima, así que un sistema de propósito general no se vuelve de tiempo real por ser más rápido. Las cotas de planificabilidad que da la IA son correctas, pero solo bajo hipótesis que hay que declarar.

### Justificación

- La cota U ≤ 100 % de EDF vale solo para tareas independientes (sin bloqueo por recursos compartidos), con deadline igual al periodo (D = T), en un monoprocesador y con overhead de cambio de contexto despreciable. La de Rate Monotonic (n(2^(1/n) − 1), ≈ 69.3 %) tiene las mismas hipótesis.
- En multiprocesador con EDF global, la utilización planificable puede caer drásticamente (efecto Dhall), por lo que en la práctica se prefiere la planificación particionada.
- Las fuentes de latencia no acotada (secciones no preemptibles, interrupciones, inversión de prioridades, fallos de caché/TLB, SMIs, paginación) no dependen de la frecuencia del procesador.

### Aplicación en sistemas de producción

- **Tiempo real blando o firme** (robótica, control industrial, audio): Linux con PREEMPT_RT (en la rama principal desde 6.12), `SCHED_FIFO` o `SCHED_DEADLINE`, `mlockall()` e `isolcpus`/`nohz_full` o particiones de cpuset.
- **Tiempo real duro certificado**: en aviónica (DO-178C DAL A) y automoción (ISO 26262 ASIL D) se usan RTOS con historial de certificación como VxWorks 653, INTEGRITY-178 o QNX. Zephyr tiene esfuerzos de certificación funcional, pero no es la referencia en esos dominios.
- Dimensionar siempre por el WCET medido y no por el caso promedio.

### Referencias

- Liu y Layland (1973), *Scheduling Algorithms for Multiprogramming in a Hard-Real-Time Environment*.
- Documentación del kernel: `scheduler/sched-deadline`; documentación de PREEMPT_RT.
