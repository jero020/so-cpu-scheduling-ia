# Q2. Equidad Absoluta vs. Prioridad Económica (CFS vs. Cuotas Empresariales)

> El planificador Completely Fair Scheduler (CFS) busca garantizar tiempo proporcional a todos los hilos. Analice conceptualmente cuándo esta equidad resulta contraproducente en plataformas multi-tenant de nube donde ciertos clientes pagan por SLAs garantizados.

**Responsable:** Jerónimo Vélez Acosta · **Estado:** ☑ Prompt ☑ Respuesta IA ☑ Crítica individual ☑ Solución unificada

---

## 1. Prompt a la IA

**Modelo usado:** Claude Opus 5.5 (Anthropic), 2026-11-20

```text
Actúa como arquitecto senior de sistemas operativos con experiencia en el planificador del kernel de Linux y en sistemas de producción.

Contexto: prueba final de un curso universitario de Sistemas Operativos (Ingeniería de Sistemas). Responde con rigor técnico, citando mecanismos reales (nombres de políticas, syscalls, parámetros del kernel y versiones cuando aplique).

Pregunta: Equidad Absoluta vs. Prioridad Económica (CFS vs. Cuotas Empresariales). El planificador Completely Fair Scheduler (CFS) busca garantizar tiempo proporcional a todos los hilos. Analice conceptualmente cuándo esta equidad resulta contraproducente en plataformas multi-tenant de nube donde ciertos clientes pagan por SLAs garantizados.

Restricciones:
- Máximo 600 palabras.
- Explica al menos dos trade-offs con su costo concreto.
- Incluye al menos un ejemplo de sistema real en producción.
- Indica explícitamente qué supuestos de hardware estás haciendo.
- Indica qué planificador usa hoy el kernel de Linux por defecto y desde qué versión.
- Menciona qué mecanismos (por ejemplo cgroups) permiten implementar SLAs sobre el planificador.
```

## 2. Respuesta de la IA

**Planificador actual**

Desde **Linux 6.6 (2023)** el planificador por defecto para `SCHED_OTHER` es **EEVDF** (Earliest Eligible Virtual Deadline First), que reemplazó a CFS. Conserva la idea de CFS (reparto proporcional según pesos derivados de `nice`) pero añade un criterio de latencia basado en *virtual deadlines*. Las conclusiones de esta respuesta aplican a ambos.

**Cuándo la equidad es contraproducente**

CFS/EEVDF reparten tiempo **entre entidades planificables** según su peso. Eso es “justo” desde el punto de vista del kernel, pero no coincide con la justicia económica de una nube multi-tenant:

1. *Equidad por hilo, no por cliente*: si el cliente A lanza 100 hilos y el cliente B 4, sin agrupación A obtiene ~96 % del CPU aunque ambos paguen lo mismo. El kernel no sabe quién paga.
2. *Equidad proporcional, no garantía absoluta*: un peso alto garantiza una **proporción**, no una cantidad de milisegundos por segundo. Si llegan más tenants, la parte de cada uno se diluye, lo que rompe un SLA expresado como “2 vCPU garantizadas”.
3. *Latencia de cola*: aunque el reparto medio sea correcto, un servicio con SLA de p99 puede sufrir picos cuando compite con cargas batch en el mismo núcleo.

**Mecanismos para implementar SLAs sobre el planificador**

- **cgroups v2, controlador `cpu`**:
  - `cpu.weight` (1–10000, por defecto 100): equidad entre grupos en lugar de entre hilos. Resuelve el punto 1.
  - `cpu.max` (`cuota periodo`, p. ej. `200000 100000` = 2 CPU): techo duro de ancho de banda.
- **cpusets** (`cpuset.cpus`): reservar núcleos físicos a un tenant premium.
- **SCHED_DEADLINE**: garantías de tiempo (runtime/period) con control de admisión, útil para cargas con requisitos temporales estrictos.

**Trade-offs**

1. *Peso vs. techo*: `cpu.weight` aprovecha la capacidad sobrante pero no garantiza un mínimo absoluto; `cpu.max` garantiza un techo pero produce *throttling* aunque haya CPU libre, aumentando la latencia de cola.
2. *Núcleos dedicados vs. densidad*: `cpuset` da aislamiento casi perfecto pero reduce la consolidación, aumentando el costo por cliente.

**Ejemplo real**: Kubernetes traduce `requests.cpu` en `cpu.weight` y `limits.cpu` en `cpu.max`; las clases de QoS (Guaranteed, Burstable, BestEffort) se construyen sobre esta jerarquía de cgroups. AWS usa instancias “burstable” (T3) con créditos de CPU, que es una cuota económica aplicada en el hipervisor.

**Supuestos de hardware**: servidores x86-64 multi-socket con SMT habilitado; se asume que la contención principal es de CPU y no de memoria o red.

## 3. Análisis crítico individual

### Jerónimo Vélez Acosta

- **Aciertos y cumplimiento:** Identifica con claridad la transición de CFS a EEVDF en Linux 6.6 y articula correctamente la diferencia entre la equidad a nivel de entidad planificable (`task_struct`) frente a la equidad organizacional o de cliente mediante el controlador `cpu` de **cgroups v2** (`cpu.weight` y `cpu.max`). La correlación con las clases de QoS de Kubernetes es exacta.
- **Vacíos y puntos críticos:**
  - *Visión incompleta del "vecino ruidoso" en hardware:* El modelo asume que la contención es exclusivamente de tiempo de CPU. Sin embargo, en servidores multi-tenant x86 con SMT habilitado, dos clientes pueden cumplir estrictamente sus cuotas de `cpu.max` y aun así degradar mutuamente sus SLAs de latencia p99 debido a la contención en recursos compartidos del núcleo físico (unidades de ejecución, puertos de despacho, caché L1/L2 en hilos hermanos SMT y ancho de banda de memoria DRAM).
  - *Matiz sobre `SCHED_DEADLINE`:* Menciona `SCHED_DEADLINE` como mecanismo para SLAs en la nube, pero omite que esta política opera a nivel de hilo individual (`sched_setattr`) y no cuenta con un controlador jerárquico nativo equivalente a los grupos de ancho de banda de cgroups para contenedores arbitrarios, lo que limita su adopción directa en orquestadores multi-tenant estándar.

## 4. Solución unificada del equipo

### Postura del equipo

La equidad de CFS/EEVDF es equidad entre entidades planificables, no entre clientes, y garantiza proporciones, no cantidades. Para vender SLAs en una nube multi-tenant hace falta agrupar por cliente (cgroups v2) y, para latencia de cola, aislar también los recursos físicos del núcleo, no solo el tiempo de CPU.

### Justificación

- Sin agrupación, un cliente con 100 hilos se lleva casi todo el CPU frente a uno con 4. `cpu.weight` corrige esto repartiendo entre grupos.
- Cumplir `cpu.max` no basta para un SLA de p99: con SMT, dos clientes en hilos hermanos compiten por las unidades de ejecución, la L1/L2 del mismo núcleo físico y el ancho de banda de DRAM, aunque cada uno respete su cuota. La IA asumió que la contención era solo de tiempo de CPU.
- `SCHED_DEADLINE` da garantías temporales, pero opera por hilo (`sched_setattr`) y no tiene un controlador jerárquico equivalente al de cgroups, así que no sirve directamente para contenedores arbitrarios en un orquestador.

### Aplicación en sistemas de producción

1. Un cgroup por cliente: `cpu.weight` proporcional al plan contratado y `cpu.max` para el techo.
2. Clientes con SLA de latencia: núcleos físicos completos (`cpuset`) sin compartir hermanos SMT con otros clientes (en Kubernetes, CPU Manager `static` con la opción `full-pcpus-only`).
3. Contención de caché y memoria: Intel RDT (CAT/MBA) o equivalente para repartir L3 y ancho de banda.
4. Kubernetes: `requests` → `cpu.weight`, `limits` → `cpu.max`, con clases de QoS Guaranteed para los clientes premium.

### Referencias

- Documentación del kernel: `admin-guide/cgroup-v2` (controladores `cpu` y `cpuset`), `scheduler/sched-deadline`.
- Documentación de Kubernetes: CPU Manager Policies y Resource Management for Pods.
