# Q4. La Paradoja de la Inversión de Prioridades

> A partir del dilema conceptual de la inversión de prioridades, evalúe por qué los mecanismos de herencia y techo de prioridad son soluciones reactivas a fallos de diseño en la sincronización. ¿Cómo impactan estos protocolos en la predictibilidad de sistemas aeroespaciales y vehiculares autónomos?

**Responsable:** Jerónimo Vélez Acosta · **Estado:** ☑ Prompt ☑ Respuesta IA ☑ Crítica individual ☑ Solución unificada

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

### Jerónimo Vélez Acosta

- **Aciertos y cumplimiento:** Diferencia con rigor técnico la Herencia de Prioridad (PIP) del Protocolo de Techo de Prioridad Inmediato (ICPP/PCP), ilustra ambos con ejemplos numéricos claros, narra con exactitud el incidente del *Mars Pathfinder* (1997) en VxWorks y acierta al señalar que ambos son parches reactivos frente a arquitecturas lock-free o particionadas.
- **Vacíos y puntos críticos:**
  - *Omisión en el bloque de sincronización de múltiples recursos:* Aunque menciona que PIP no evita *deadlocks* y sufre bloqueos encadenados, no detalla el mecanismo formal por el cual PCP los previene: al elevar inmediatamente la prioridad de la tarea al techo del recurso al adquirirlo, ninguna otra tarea que comparta ese recurso (o cualquiera con techo menor o igual) puede ejecutar ni adquirir locks anidados, garantizando como máximo un único bloqueo por sección crítica.
  - *Riesgo en arquitecturas multinúcleo (SMP):* Si bien en los supuestos aclara que asume un monoprocesador, en sistemas aeroespaciales y vehiculares modernos (SoCs multinúcleo) el techo de prioridad clásico falla si dos tareas en núcleos distintos compiten por el mismo spinlock/mutex. El modelo solo nombra de pasada MPCP/MSRP sin explicar que en SMP la espera deja de ser puramente por preempción y se convierte en espera activa o suspensión inter-núcleo.

## 4. Solución unificada del equipo

### Postura del equipo

La herencia y el techo de prioridad mitigan un problema causado por compartir recursos entre tareas de distinta criticidad; son reactivos. Para sistemas críticos preferimos el techo de prioridad (ICPP) en monoprocesador, por su predictibilidad, y en multinúcleo protocolos diseñados para SMP o, mejor, diseños que eviten compartir recursos entre núcleos.

### Justificación

- **Por qué PCP/ICPP previene deadlocks y bloqueos encadenados:** al adquirir un recurso, la tarea sube de inmediato al techo de ese recurso. Ninguna otra tarea que use ese recurso, ni ninguna con prioridad menor o igual al techo, puede ejecutarse y tomar otro lock en ese intervalo. Por eso cada tarea se bloquea como máximo una vez por una sección crítica de menor prioridad, y no se forman ciclos de espera.
- **PIP** es transparente, pero el peor caso se vuelve difícil de calcular por los bloqueos encadenados.
- **En SMP** el problema cambia: dos tareas en núcleos distintos compiten por el mismo lock sin preemptarse entre sí, y la espera pasa a ser activa (spin) o una suspensión entre núcleos. Los protocolos MPCP y MSRP extienden el techo a ese escenario, pero con cotas más pesimistas.

### Aplicación en sistemas de producción

- RTOS automotriz (OSEK/AUTOSAR): ICPP es el estándar para recursos compartidos.
- Linux: `pthread_mutexattr_setprotocol` con `PTHREAD_PRIO_PROTECT` (techo) o `PTHREAD_PRIO_INHERIT` (herencia); PREEMPT_RT usa `rt_mutex` con herencia.
- Diseño: secciones críticas cortas, recursos particionados por núcleo y comunicación por colas entre niveles de criticidad.
- Caso de referencia: Mars Pathfinder (1997), corregido activando la herencia de prioridad en el mutex de VxWorks.

### Referencias

- Sha, Rajkumar y Lehoczky (1990), *Priority Inheritance Protocols: An Approach to Real-Time Synchronization*.
- Páginas de manual: `pthread_mutexattr_setprotocol(3)`.
