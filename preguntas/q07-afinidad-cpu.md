# Q7. Afinidad de CPU (Affinity) vs. Flexibilidad Global

> Discuta el compromiso entre fijar tareas críticas a núcleos dedicados (CPU pinning) para maximizar la localidad de caché vs. permitir que el planificador migre libremente los procesos para balancear la temperatura y la carga de los sockets.

**Responsable:** Jerónimo Vélez Acosta · **Estado:** ☑ Prompt ☑ Respuesta IA ☑ Crítica individual ☑ Solución unificada

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

### Jerónimo Vélez Acosta

- **Aciertos y cumplimiento:** Cubre de forma exhaustiva la pila técnica de aislamiento en Linux: desde syscalls (`sched_setaffinity`) y comandos (`taskset`, `numactl`) hasta parámetros de arranque (`isolcpus`, `nohz_full`, `rcu_nocbs`) y enrutamiento de interrupciones (`smp_affinity`). La advertencia sobre la desalineación entre afinidad de CPU y memoria NUMA es muy acertada.
- **Vacíos y puntos críticos:**
  - *Obsolescencia relativa de `isolcpus`:* El modelo recomienda `isolcpus` como parámetro estándar de arranque, sin advertir que en la documentación oficial del kernel de Linux `isolcpus` está marcado como *deprecated* en favor de particiones exclusivas mediante **cgroups v2 (`cpuset.cpus.partition=isolated`)**. Usar `isolcpus` estáticamente impide reconfigurar el dominio de planificabilidad en caliente sin reiniciar el nodo.
  - *Interferencia de hilos hermanos SMT (Hyper-Threading):* Aunque en los supuestos menciona que SMT está habilitado y los hilos comparten L1/L2, no extrae la consecuencia crítica en los trade-offs: hacer *pinning* de un proceso crítico al núcleo lógico 0 sin aislar o apagar su hermano físico (por ejemplo, núcleo lógico 32) invalida el aislamiento determinista, pues el hilo hermano seguirá compitiendo por las unidades de ejecución y contaminando la caché L1.

## 4. Solución unificada del equipo

### Postura del equipo

El *pinning* es la herramienta correcta para cargas que necesitan latencia predecible, pero solo funciona si el aislamiento es completo: CPU, memoria NUMA, interrupciones y el hilo hermano SMT. Para cargas generales, la migración libre del planificador es mejor. Para aislar núcleos recomendamos particiones de cpuset en cgroups v2 en lugar de `isolcpus`.

### Justificación

- `isolcpus` está marcado como obsoleto en la documentación de parámetros del kernel en favor de cpusets. Además es estático: cambiar el conjunto de núcleos aislados exige reiniciar. Las particiones de cgroups v2 (`cpuset.cpus.partition = isolated`) se reconfiguran en caliente.
- Fijar un proceso a un núcleo lógico sin controlar su hermano SMT no da aislamiento determinista: el hermano sigue compitiendo por las unidades de ejecución y la L1/L2. Hay que reservar el núcleo físico completo o desactivar SMT en esos núcleos.
- Fijar CPU sin fijar memoria (`numactl --membind`) puede empeorar la latencia por accesos al nodo NUMA remoto.

### Aplicación en sistemas de producción

1. Partición aislada de cpuset con núcleos físicos completos.
2. `nohz_full` y `rcu_nocbs` en esos núcleos; IRQs fuera de ellos vía `/proc/irq/*/smp_affinity`.
3. Proceso fijado con `sched_setaffinity` y memoria en el mismo nodo NUMA.
4. Monitorear temperatura y frecuencia: un núcleo fijo con carga sostenida puede perder turbo.

Ejemplos: DPDK y funciones de red virtualizadas, trading de baja latencia, VMs con vCPUs fijadas.

### Referencias

- Documentación del kernel: `admin-guide/kernel-parameters` (`isolcpus`, `nohz_full`), `admin-guide/cgroup-v2` (cpuset partitions).
- Páginas de manual: `sched_setaffinity(2)`, `numactl(8)`.
