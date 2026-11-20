# Q3. Sistemas de Tiempo Real Duro vs. Blando: Filosofía y Resiliencia

> Contraste la filosofía determinista de los algoritmos de tiempo real (donde llegar tarde equivale a un fallo catastrófico) frente a los planificadores de propósito general. ¿Por qué agregar más potencia de CPU no convierte automáticamente un sistema estándar en uno de tiempo real?

**Responsable:** _(nombre)_ · **Estado:** ☑ Prompt ☑ Respuesta IA ☐ Crítica individual ☐ Solución unificada

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

<!-- Borrar las secciones de integrantes que no existan. Cada quien escribe la suya. -->

### Integrante A — _(nombre)_

- **Errores o imprecisiones técnicas:** 
- **Supuestos de arquitectura/hardware que omite:** 
- **Trade-offs que no contempla:** 
- **Fuente con la que lo verifiqué:** 

### Integrante B — _(nombre)_

- **Errores o imprecisiones técnicas:** 
- **Supuestos de arquitectura/hardware que omite:** 
- **Trade-offs que no contempla:** 
- **Fuente con la que lo verifiqué:** 

### Integrante C — _(nombre)_

- **Errores o imprecisiones técnicas:** 
- **Supuestos de arquitectura/hardware que omite:** 
- **Trade-offs que no contempla:** 
- **Fuente con la que lo verifiqué:** 

### Integrante D — _(nombre)_

- **Errores o imprecisiones técnicas:** 
- **Supuestos de arquitectura/hardware que omite:** 
- **Trade-offs que no contempla:** 
- **Fuente con la que lo verifiqué:** 

### Integrante E — _(nombre)_

- **Errores o imprecisiones técnicas:** 
- **Supuestos de arquitectura/hardware que omite:** 
- **Trade-offs que no contempla:** 
- **Fuente con la que lo verifiqué:** 

## 4. Solución unificada del equipo

<!-- Postura técnica definitiva, justificada con teoría. Añadir pseudocódigo, diagrama (en /diagramas) o syscalls cuando aplique. -->

### Postura del equipo

### Justificación

### Aplicación en sistemas de producción

### Referencias
