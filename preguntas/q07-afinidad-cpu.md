# Q7. Afinidad de CPU (Affinity) vs. Flexibilidad Global

> Discuta el compromiso entre fijar tareas críticas a núcleos dedicados (CPU pinning) para maximizar la localidad de caché vs. permitir que el planificador migre libremente los procesos para balancear la temperatura y la carga de los sockets.

**Responsable:** _(nombre)_ · **Estado:** ☑ Prompt ☑ Respuesta IA ☐ Crítica individual ☐ Solución unificada

---

## 1. Prompt a la IA

**Modelo usado:** Claude Opus 5.5 (Anthropic), 2026-11-20

```text
Actúa como arquitecto senior de sistemas operativos con experiencia en el planificador del kernel de Linux y en sistemas de producción.

Contexto: prueba final de un curso universitario de Sistemas Operativos (Ingeniería de Sistemas). Responde con rigor técnico, citando mecanismos reales (nombres de políticas, syscalls, parámetros del kernel y versiones cuando aplique).

Pregunta: Afinidad de CPU (Affinity) vs. Flexibilidad Global. Discuta el compromiso entre fijar tareas críticas a núcleos dedicados (CPU pinning) para maximizar la localidad de caché vs. permitir que el planificador migre libremente los procesos para balancear la temperatura y la carga de los sockets.

Restricciones:
- Máximo 600 palabras.
- Explica al menos dos trade-offs con su costo concreto.
- Incluye al menos un ejemplo de sistema real en producción.
- Indica explícitamente qué supuestos de hardware estás haciendo.
- Menciona las herramientas, syscalls y parámetros de arranque de Linux relacionados con afinidad y aislamiento de núcleos.
```

## 2. Respuesta de la IA

**Los dos extremos**

- **CPU pinning**: fijar una tarea a uno o varios núcleos. La tarea conserva su caché L1/L2 caliente, evita migraciones, y si además se aísla el núcleo, no sufre interferencia de otras tareas ni de interrupciones. Resultado: latencia baja y, sobre todo, **predecible**.
- **Migración libre**: el planificador mueve tareas entre núcleos y sockets para balancear la carga, aprovechar núcleos ociosos, repartir el calor y permitir que núcleos inactivos entren en estados de bajo consumo.

**Herramientas y mecanismos en Linux**

- Syscalls: `sched_setaffinity(2)` / `sched_getaffinity(2)`; en pthreads, `pthread_setaffinity_np`.
- Comandos: `taskset -c 2,3 <cmd>`; `numactl --cpunodebind=0 --membind=0` para fijar CPU y memoria al mismo nodo NUMA.
- cgroups: controlador `cpuset` (`cpuset.cpus`, `cpuset.mems`), usado por Kubernetes con la política `static` del CPU Manager para pods Guaranteed con CPUs enteras.
- Parámetros de arranque: `isolcpus=` (saca núcleos del balanceo del planificador), `nohz_full=` (elimina el tick del temporizador en núcleos con una sola tarea), `rcu_nocbs=` (saca los callbacks de RCU).
- Interrupciones: `/proc/irq/<n>/smp_affinity` e `irqbalance` para mantener las IRQ fuera de los núcleos aislados.

**Trade-offs**

1. *Localidad vs. utilización*: un núcleo dedicado a una tarea que no está siempre activa queda ocioso; esa capacidad no la puede usar nadie más. En un servidor de 64 núcleos, reservar 8 para un proceso crítico es ~12 % de la máquina.
2. *Determinismo vs. gestión térmica*: un núcleo fijo con carga sostenida se calienta y puede sufrir *thermal throttling* o perder margen de turbo, mientras otros núcleos están fríos; el planificador no puede mover la tarea para compensar.
3. *NUMA*: fijar la CPU sin fijar la memoria puede ser peor que no fijar nada, porque los accesos van al nodo remoto.

**Cuándo conviene cada uno**

- *Pinning*: trading de alta frecuencia, procesamiento de paquetes (DPDK fija un hilo por núcleo en *busy polling*), audio profesional, bases de datos en máquinas dedicadas, VMs con vCPUs fijadas.
- *Migración libre*: cargas generales, servidores multi-tenant, máquinas con carga variable.

**Ejemplo real**: DPDK y las funciones de red virtualizadas (NFV) en operadores de telecomunicaciones usan `isolcpus` + `nohz_full` + pinning para procesar paquetes con latencias de microsegundos.

**Supuestos de hardware**: servidor x86-64 de dos sockets con NUMA, SMT habilitado (dos hilos por núcleo físico comparten L1/L2) y control térmico por paquete.

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
