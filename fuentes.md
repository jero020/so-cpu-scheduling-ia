# Fuentes para validar las respuestas de la IA

## Libros
- Silberschatz, Galvin y Gagne. *Operating System Concepts*. Capítulo de planificación de CPU.
- Tanenbaum y Bos. *Modern Operating Systems*. Capítulo de procesos e hilos.
- Arpaci-Dusseau y Arpaci-Dusseau. *Operating Systems: Three Easy Pieces* (OSTEP), capítulos de scheduling (gratis en ostep.org).

## Documentación del kernel de Linux (docs.kernel.org)
- Scheduler: CFS y EEVDF (`scheduler/sched-design-CFS`, `scheduler/sched-eevdf`).
- Tiempo real: `scheduler/sched-deadline`, `scheduler/sched-rt-group`.
- Energy Aware Scheduling: `scheduler/sched-energy`.
- sched_ext: `scheduler/sched-ext`.
- cgroups v2: `admin-guide/cgroup-v2` (controlador `cpu`).
- Páginas de manual: `sched(7)`, `sched_setaffinity(2)`, `sched_setattr(2)`.

## Casos y artículos
- Inversión de prioridades en el Mars Pathfinder (1997).
- Documentación de Kubernetes sobre requests, limits y CPU throttling.

## Usadas en la entrega
<!-- Agregar aquí cada fuente citada, con enlace y la pregunta donde se usó. -->
