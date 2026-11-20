# Q9. Aislamiento de Cómputo en Contenedores: Shares Relativos vs. Hard Limits

> Compare las dos filosofías de control de recursos en plataformas de microservicios: repartir la CPU de forma elástica cuando sobra capacidad (shares) versus imponer un techo infranqueable (hard quota). ¿Cuál es la estrategia óptima para evitar vecinos ruidosos ('noisy neighbors') sin desperdiciar infraestructura?

**Responsable:** _(nombre)_ · **Estado:** ☑ Prompt ☑ Respuesta IA ☐ Crítica individual ☐ Solución unificada

---

## 1. Prompt a la IA

**Modelo usado:** Claude Opus 5.5 (Anthropic), 2026-11-20

```text
Actúa como arquitecto senior de sistemas operativos con experiencia en el planificador del kernel de Linux y en sistemas de producción.

Contexto: prueba final de un curso universitario de Sistemas Operativos (Ingeniería de Sistemas). Responde con rigor técnico, citando mecanismos reales (nombres de políticas, syscalls, parámetros del kernel y versiones cuando aplique).

Pregunta: Aislamiento de Cómputo en Contenedores: Shares Relativos vs. Hard Limits. Compare las dos filosofías de control de recursos en plataformas de microservicios: repartir la CPU de forma elástica cuando sobra capacidad (shares) versus imponer un techo infranqueable (hard quota). ¿Cuál es la estrategia óptima para evitar vecinos ruidosos ('noisy neighbors') sin desperdiciar infraestructura?

Restricciones:
- Máximo 600 palabras.
- Explica al menos dos trade-offs con su costo concreto.
- Incluye al menos un ejemplo de sistema real en producción.
- Indica explícitamente qué supuestos de hardware estás haciendo.
- Relaciona tu respuesta con cgroups v2 (cpu.weight, cpu.max) y con requests/limits de Kubernetes.
```

## 2. Respuesta de la IA

**Dos filosofías**

- **Shares relativos** (`cpu.weight` en cgroups v2): cada grupo recibe CPU en proporción a su peso **solo cuando hay contención**. Si la máquina tiene capacidad libre, un contenedor puede usar todo lo que necesite (*work-conserving*).
- **Hard limits** (`cpu.max` en cgroups v2, “cuota periodo”): el grupo no puede usar más de su cuota en cada periodo (por defecto 100 ms), aunque el resto del CPU esté libre (*non-work-conserving*).

**Relación con Kubernetes**

- `requests.cpu` → se traduce en `cpu.weight` del cgroup del pod y lo usa el *scheduler* de Kubernetes para colocar pods en nodos.
- `limits.cpu` → se traduce en `cpu.max`. Por ejemplo, `limits.cpu: 500m` = `50000 100000` (50 ms cada 100 ms).
- Clases de QoS: **Guaranteed** (requests = limits), **Burstable** (requests < limits), **BestEffort** (sin ninguno).

**El problema del throttling**

Con hard limits, una aplicación multihilo puede gastar su cuota de 100 ms en los primeros 25 ms del periodo si tiene 4 hilos activos, y queda **congelada 75 ms** aunque el nodo esté ocioso. Esto aparece como picos en la latencia p99 sin uso alto de CPU. Es un problema documentado en producción y la razón por la que varias organizaciones recomiendan no poner `limits.cpu` en servicios sensibles a la latencia.

**Trade-offs**

1. *Shares*: máximo aprovechamiento de la infraestructura, pero el rendimiento de un contenedor depende de sus vecinos y es menos predecible.
2. *Hard limits*: aislamiento y facturación predecibles, pero desperdician capacidad libre y generan throttling.

**Estrategia recomendada contra vecinos ruidosos**

1. Definir siempre `requests` realistas (medidos, no adivinados): eso fija los pesos y permite al orquestador no sobrecomprometer nodos.
2. Para servicios críticos sensibles a latencia: QoS Guaranteed con CPUs enteras y CPU Manager `static` (núcleos exclusivos vía `cpuset`), o bien solo `requests` sin `limits` para evitar throttling.
3. Para cargas batch o de bajo valor: `limits` estrictos o prioridad baja (`cpu.weight` bajo, `cpu.idle` en cgroups v2) para que cedan CPU.
4. Separar por *node pools* las cargas que compiten por recursos no controlados por cgroups (caché L3, ancho de banda de memoria).
5. Monitorear `cpu.stat` (`nr_throttled`, `throttled_usec`) y la presión de CPU (PSI, `cpu.pressure`).

**Ejemplo real**: varias empresas (por ejemplo Zalando y Buffer) reportaron públicamente mejoras de latencia al quitar los CPU limits en servicios críticos; desde Linux 5.14 existe `cpu.max.burst` para permitir ráfagas acumuladas y reducir el throttling.

**Supuestos de hardware**: nodos x86-64 multinúcleo con SMT, sin aislamiento de caché L3 (Intel RDT/CAT no configurado).

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
