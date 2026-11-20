# Q2. Equidad Absoluta vs. Prioridad Económica (CFS vs. Cuotas Empresariales)

> El planificador Completely Fair Scheduler (CFS) busca garantizar tiempo proporcional a todos los hilos. Analice conceptualmente cuándo esta equidad resulta contraproducente en plataformas multi-tenant de nube donde ciertos clientes pagan por SLAs garantizados.

**Responsable:** _(nombre)_ · **Estado:** ☑ Prompt ☑ Respuesta IA ☐ Crítica individual ☐ Solución unificada

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
