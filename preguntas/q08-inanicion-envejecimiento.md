# Q8. Inanición (Starvation) y Envejecimiento (Aging): Ética y Diseño Algorítmico

> Explore la justificación conceptual del mecanismo de envejecimiento ('Priority Aging') en colas multinivel. ¿Bajo qué condiciones una decisión algorítmica legítima de priorizar siempre lo más urgente puede paralizar indefinidamente tareas de mantenimiento esenciales?

**Responsable:** Jerónimo Vélez Acosta · **Estado:** ☑ Prompt ☑ Respuesta IA ☑ Crítica individual ☑ Solución unificada

---

## 1. Prompt a la IA

**Modelo usado:** Claude Opus 5.5 (Anthropic), 2026-11-20

```text
Actúa como arquitecto senior de sistemas operativos con experiencia en el planificador del kernel de Linux y en sistemas de producción.

Contexto: prueba final de un curso universitario de Sistemas Operativos (Ingeniería de Sistemas). Responde con rigor técnico, citando mecanismos reales (nombres de políticas, syscalls, parámetros del kernel y versiones cuando aplique).

Pregunta: Inanición (Starvation) y Envejecimiento (Aging): Ética y Diseño Algorítmico. Explore la justificación conceptual del mecanismo de envejecimiento ('Priority Aging') en colas multinivel. ¿Bajo qué condiciones una decisión algorítmica legítima de priorizar siempre lo más urgente puede paralizar indefinidamente tareas de mantenimiento esenciales?

Restricciones:
- Máximo 600 palabras.
- Explica al menos dos trade-offs con su costo concreto.
- Incluye al menos un ejemplo de sistema real en producción.
- Indica explícitamente qué supuestos de hardware estás haciendo.
- Incluye pseudocódigo de una cola multinivel con envejecimiento.
- Explica cómo se elige la tasa de envejecimiento y qué pasa si es muy alta o muy baja.
```

## 2. Respuesta de la IA

**Justificación del envejecimiento**

En una cola multinivel con prioridades estrictas, el planificador siempre elige la cola de mayor prioridad que tenga tareas listas. Es una decisión localmente correcta: lo urgente primero. El problema aparece cuando la llegada de trabajo urgente es **continua**: las colas inferiores nunca se atienden y sus tareas sufren **inanición** (starvation) indefinida.

El **envejecimiento** introduce una noción de justicia temporal: la prioridad efectiva de una tarea crece con el tiempo que lleva esperando. Así se garantiza un límite superior de espera, sin renunciar a favorecer lo urgente en el corto plazo. Conceptualmente convierte una garantía de “mejor esfuerzo” en una garantía de **progreso** (*liveness*).

**Cuándo priorizar lo urgente paraliza el mantenimiento**

- Cuando la tasa de llegada de trabajo de alta prioridad se acerca a la capacidad del sistema (utilización ≈ 100 % en las colas altas).
- Cuando las tareas de baja prioridad son **esenciales a largo plazo**: rotación de logs, compactación de bases de datos (*vacuum* en PostgreSQL), recolección de basura, backups, reconstrucción de índices.
- Cuando la inanición tiene efectos acumulativos: si no se compacta, el disco se llena; si no se rotan logs, el servicio urgente termina fallando. El sistema se degrada justamente por proteger lo urgente.

**Pseudocódigo: cola multinivel con envejecimiento**

```text
constantes: N_NIVELES, UMBRAL_ESPERA, QUANTUM[nivel]

cada tick del reloj:
    para cada nivel n desde 1 hasta N_NIVELES-1:      // 0 = más alta prioridad
        para cada tarea t en cola[n]:
            t.espera += 1
            si t.espera >= UMBRAL_ESPERA:
                mover t a cola[n-1]                     // envejece: sube un nivel
                t.espera = 0

planificar():
    para n desde 0 hasta N_NIVELES-1:
        si cola[n] no vacía:
            t = sacar_frente(cola[n])
            ejecutar t durante QUANTUM[n]
            si t usó todo su quantum:
                mover t a cola[min(n+1, N_NIVELES-1)]   // CPU-bound: baja
            si no:
                volver a encolar t en cola[n]           // I/O-bound: se mantiene
            t.espera = 0
            retornar
```

**Cómo elegir la tasa de envejecimiento**

- **Muy alta** (umbral pequeño): todas las tareas suben rápido y la cola se aplana; se pierde la diferenciación de prioridades y la latencia de las tareas urgentes empeora.
- **Muy baja** (umbral grande): el envejecimiento casi nunca actúa y la inanición práctica persiste.
- Criterio práctico: fijar el umbral a partir del **máximo tiempo de espera tolerable** para la tarea de menor prioridad (por ejemplo, “el vacuum debe correr al menos cada 10 minutos”).

**Trade-offs**

1. Más envejecimiento = mejor equidad pero peor latencia para lo urgente.
2. Más niveles = más control fino pero más estado y más overhead en cada tick.

**Ejemplo real**: el planificador O(1) de Linux (2.6, antes de CFS) usaba bonificaciones dinámicas de prioridad; Windows aplica *priority boosting* a hilos que llevan ~4 segundos listos sin ejecutarse para evitar inanición.

**Supuestos de hardware**: monoprocesador lógico o una cola por núcleo; en multinúcleo el envejecimiento interactúa con el balanceo de carga.

## 3. Análisis crítico individual

### Jerónimo Vélez Acosta

- **Aciertos y cumplimiento:** Articula sólidamente la diferencia entre equidad instantánea y garantía de progreso (*liveness*), ilustra con ejemplos reales de sistemas operativos (planificador O(1) de Linux y *priority boosting* en Windows) y señala las consecuencias sistémicas de paralizar tareas de mantenimiento como `vacuum` en PostgreSQL o la rotación de logs.
- **Vacíos y puntos críticos:**
  - *Fallos lógicos y de diseño en el pseudocódigo:*
    1. En la sección `cada tick del reloj`, el pseudocódigo promueve tareas de `cola[n]` a `cola[n-1]` inmediatamente al alcanzar `UMBRAL_ESPERA`, pero en `planificar()`, si una tarea no consume todo su *quantum* (comportamiento típico I/O-bound), se vuelve a encolar en su cola actual `cola[n]` y resetea `t.espera = 0`. Esto implica que una tarea envejecida que subió a `cola[0]` (máxima prioridad) y hace una operación de I/O corta se queda permanentemente en `cola[0]`, causando una **inversión de prioridad permanente** y desvirtuando el propósito de las colas inferiores.
    2. El recorrido en cada *tick* sobre todas las colas (O(N) respecto al número de procesos listos) viola el principio de escalabilidad en tiempo constante O(1) de los planificadores modernos; en implementaciones reales se utilizan marcas de tiempo (*timestamps* de encolado) evaluadas de forma perezosa (*lazy*) en lugar de incrementar contadores en cada interrupción de reloj.

## 4. Solución unificada del equipo

### Postura del equipo

El envejecimiento está justificado porque convierte una política "lo urgente primero" en una con garantía de progreso. Sin embargo, el pseudocódigo de la IA tiene dos fallas: una tarea envejecida puede quedarse para siempre en la cola más alta, y recorrer todas las colas en cada tick no escala. Proponemos una versión corregida.

### Justificación

- En la versión de la IA, una tarea que sube por envejecimiento a la cola 0 y luego hace E/S corta se reencola en la cola 0 con `espera = 0`, así que se queda ahí indefinidamente aunque su prioridad original fuera baja.
- Incrementar un contador en cada tick para todas las tareas listas cuesta O(N) por interrupción de reloj. Los planificadores reales registran la marca de tiempo de encolado y evalúan la espera de forma perezosa al elegir.

### Pseudocódigo corregido

```text
encolar(t, nivel):
    t.nivel_actual = nivel
    t.t_encolado = ahora()
    insertar t al final de cola[nivel]

elegir():
    para n desde 0 hasta N_NIVELES-1:
        para cada t al frente de cola[n]:              // revisión perezosa, solo el frente
            si n > 0 y ahora() - t.t_encolado >= UMBRAL_ESPERA:
                sacar t; t.boost = verdadero
                encolar(t, n-1)                        // envejece un nivel
        si cola[n] no vacía:
            retornar sacar_frente(cola[n])

al_terminar_quantum(t, uso_completo):
    si t.boost:
        t.boost = falso
        encolar(t, t.nivel_base)                       // vuelve a su nivel original
    sino si uso_completo:
        encolar(t, min(t.nivel_actual+1, N_NIVELES-1)) // CPU-bound: baja
    sino:
        encolar(t, t.nivel_actual)                     // I/O-bound: se mantiene
```

El *boost* por envejecimiento es temporal: tras ejecutar, la tarea vuelve a su nivel base. Así se garantiza progreso sin invertir prioridades de forma permanente.

### Aplicación en sistemas de producción

- Fijar `UMBRAL_ESPERA` a partir del máximo retraso tolerable de las tareas de mantenimiento (por ejemplo, que el *vacuum* o la rotación de logs corra al menos cada N minutos).
- Ejemplos reales: el *priority boost* de Windows para hilos que llevan ~4 s listos sin ejecutarse, y la regla de *priority boost* periódico de MLFQ.

### Referencias

- Arpaci-Dusseau, *OSTEP*, capítulo "Multi-Level Feedback Queue".
- Silberschatz, *Operating System Concepts*, sección de colas multinivel con retroalimentación.
