# Q4. La Paradoja de la Inversión de Prioridades

> A partir del dilema conceptual de la inversión de prioridades, evalúe por qué los mecanismos de herencia y techo de prioridad son soluciones reactivas a fallos de diseño en la sincronización. ¿Cómo impactan estos protocolos en la predictibilidad de sistemas aeroespaciales y vehiculares autónomos?

**Responsable:** _(nombre)_ · **Estado:** ☑ Prompt ☑ Respuesta IA ☐ Crítica individual ☐ Solución unificada

---

## 1. Prompt a la IA

**Modelo usado:** Claude Opus 5.5 (Anthropic), 2026-11-20

```text
Actúa como arquitecto senior de sistemas operativos con experiencia en el planificador del kernel de Linux y en sistemas de producción.

Contexto: prueba final de un curso universitario de Sistemas Operativos (Ingeniería de Sistemas). Responde con rigor técnico, citando mecanismos reales (nombres de políticas, syscalls, parámetros del kernel y versiones cuando aplique).

Pregunta: La Paradoja de la Inversión de Prioridades. A partir del dilema conceptual de la inversión de prioridades, evalúe por qué los mecanismos de herencia y techo de prioridad son soluciones reactivas a fallos de diseño en la sincronización. ¿Cómo impactan estos protocolos en la predictibilidad de sistemas aeroespaciales y vehiculares autónomos?

Restricciones:
- Máximo 600 palabras.
- Explica al menos dos trade-offs con su costo concreto.
- Incluye al menos un ejemplo de sistema real en producción.
- Indica explícitamente qué supuestos de hardware estás haciendo.
- Diferencia con precisión herencia de prioridad y techo de prioridad, con un ejemplo de cada uno.
- Incluye un caso histórico real documentado.
```

## 2. Respuesta de la IA

**El problema**

Hay inversión de prioridades cuando una tarea de alta prioridad H espera un recurso (mutex) que tiene una tarea de baja prioridad L, y una tarea de prioridad media M, que no necesita el recurso, preempta a L. Resultado: H queda bloqueada indirectamente por M durante un tiempo **no acotado**. El planificador está haciendo exactamente lo que debe (correr la tarea lista de mayor prioridad), pero la sincronización rompe la semántica de prioridades.

**Los dos protocolos**

- **Herencia de prioridad (Priority Inheritance, PIP)**: cuando H se bloquea en un mutex que tiene L, L hereda temporalmente la prioridad de H hasta liberarlo. Así M ya no puede preemptar a L.
  - *Ejemplo*: L (prio 10) tiene el mutex; H (prio 90) lo pide; L pasa a prio 90, termina su sección crítica, libera y vuelve a 10; H continúa.
  - *Limitaciones*: no evita deadlocks y puede producir **bloqueos encadenados** (H espera a L1 que espera a L2…), lo que complica calcular el peor caso.
- **Techo de prioridad (Priority Ceiling, PCP / Immediate Ceiling)**: cada recurso tiene un techo igual a la prioridad máxima de las tareas que lo usan. Al tomar el recurso, la tarea sube de inmediato al techo.
  - *Ejemplo*: un mutex usado por tareas de prioridad 40 y 80 tiene techo 80; cualquier tarea que lo tome corre a 80 mientras lo tenga.
  - *Ventajas*: acota el bloqueo a una sola sección crítica y previene deadlocks. *Costo*: hay que conocer de antemano qué tareas usan cada recurso.

**Por qué son soluciones reactivas**

Ambos parchean una decisión de diseño: compartir recursos mutables entre tareas de criticidad distinta. La causa raíz se ataca mejor con diseño: secciones críticas cortas, comunicación por mensajes o colas lock-free, o particionar los recursos por nivel de criticidad.

**Caso histórico**

En **1997**, el **Mars Pathfinder** sufrió reinicios repetidos en Marte. Una tarea de baja prioridad (meteorología) tenía un mutex del bus de información; una tarea de prioridad media de comunicaciones la preemptaba; la tarea de alta prioridad de gestión del bus no terminaba a tiempo y el *watchdog* reiniciaba el sistema. En VxWorks el mutex tenía la herencia de prioridad desactivada; JPL la activó con un parche remoto.

**Impacto en sistemas aeroespaciales y vehiculares**

La predictibilidad exige que el peor tiempo de bloqueo sea calculable. PCP lo acota mejor y por eso es preferido en estándares como OSEK/AUTOSAR (que usa el Immediate Ceiling Priority Protocol). En Linux, `pthread_mutexattr_setprotocol` permite `PTHREAD_PRIO_INHERIT` y `PTHREAD_PRIO_PROTECT`, y PREEMPT_RT usa `rt_mutex` con herencia.

**Trade-offs**

1. PIP es transparente pero hace más difícil el análisis de peor caso.
2. PCP es predecible pero sobre-eleva prioridades aunque no haya contención, reduciendo la capacidad de respuesta de tareas intermedias.

**Supuestos de hardware**: monoprocesador o planificación particionada por núcleo; en multinúcleo los protocolos se extienden (MPCP, MSRP) y el análisis es más complejo.

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
