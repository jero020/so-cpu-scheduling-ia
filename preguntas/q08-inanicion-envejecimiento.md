# Q8. Inanición (Starvation) y Envejecimiento (Aging): Ética y Diseño Algorítmico

> Explore la justificación conceptual del mecanismo de envejecimiento ('Priority Aging') en colas multinivel. ¿Bajo qué condiciones una decisión algorítmica legítima de priorizar siempre lo más urgente puede paralizar indefinidamente tareas de mantenimiento esenciales?

**Responsable:** _(nombre)_ · **Estado:** ☑ Prompt ☑ Respuesta IA ☐ Crítica individual ☐ Solución unificada

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
