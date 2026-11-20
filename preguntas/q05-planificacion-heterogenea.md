# Q5. Planificación Heterogénea en la Era del Silicio Diverso (big.LITTLE / DynamIQ)

> Con la convivencia de núcleos orientados a eficiencia energética y núcleos de alto rendimiento, ¿cómo cambia el paradigma conceptual de planificación de CPU? Analice el impacto estratégico en dispositivos móviles y centros de datos 'verdes'.

**Responsable:** _(nombre)_ · **Estado:** ☑ Prompt ☑ Respuesta IA ☐ Crítica individual ☐ Solución unificada

---

## 1. Prompt a la IA

**Modelo usado:** Claude Opus 5.5 (Anthropic), 2026-11-20

```text
Actúa como arquitecto senior de sistemas operativos con experiencia en el planificador del kernel de Linux y en sistemas de producción.

Contexto: prueba final de un curso universitario de Sistemas Operativos (Ingeniería de Sistemas). Responde con rigor técnico, citando mecanismos reales (nombres de políticas, syscalls, parámetros del kernel y versiones cuando aplique).

Pregunta: Planificación Heterogénea en la Era del Silicio Diverso (big.LITTLE / DynamIQ). Con la convivencia de núcleos orientados a eficiencia energética y núcleos de alto rendimiento, ¿cómo cambia el paradigma conceptual de planificación de CPU? Analice el impacto estratégico en dispositivos móviles y centros de datos 'verdes'.

Restricciones:
- Máximo 600 palabras.
- Explica al menos dos trade-offs con su costo concreto.
- Incluye al menos un ejemplo de sistema real en producción.
- Indica explícitamente qué supuestos de hardware estás haciendo.
- Menciona los mecanismos concretos que usan Linux/Android y los fabricantes de CPU para este tipo de planificación.
```

## 2. Respuesta de la IA

**El cambio de paradigma**

En un SMP clásico todos los núcleos son iguales y el planificador solo decide **cuándo** corre cada tarea y en qué núcleo para balancear la carga. Con arquitecturas heterogéneas (ARM big.LITTLE/DynamIQ, Intel P-cores/E-cores desde Alder Lake, Apple M-series) la decisión es **dónde** según el tipo de núcleo, y el objetivo deja de ser solo rendimiento: entra el **consumo de energía**. La métrica pasa de “trabajo por segundo” a “trabajo por joule” sujeto a un requisito de latencia.

**Mecanismos concretos**

- **Linux: Energy Aware Scheduling (EAS)**, en la rama principal desde Linux 5.0. Usa un *Energy Model* del SoC (potencia por frecuencia y por tipo de núcleo) y la utilización estimada de cada tarea (PELT) para colocar la tarea donde el consumo total sea menor sin superar la capacidad del núcleo.
- **Capacity-aware scheduling**: el kernel conoce la capacidad relativa de cada núcleo (`cpu_capacity`) y migra tareas “grandes” a núcleos grandes.
- **uclamp** (utilization clamping, Linux 5.3): permite que el espacio de usuario fije mínimos y máximos de utilización; Android lo usa para subir de prioridad la app en primer plano.
- **schedutil**: el gobernador de frecuencia que se alimenta de la misma información del planificador (DVFS integrado).
- **Intel Thread Director**: el hardware clasifica las instrucciones que ejecuta cada hilo y entrega pistas al SO mediante la tabla HFI (Hardware Feedback Interface); Windows 11 lo usa de forma completa y Linux soporta HFI desde 5.18.
- **Apple**: macOS usa clases de QoS definidas por el desarrollador (user-interactive, background…) para decidir entre P-cores y E-cores.

**Impacto estratégico**

- *Móviles*: la batería es la restricción dominante. Mantener tareas de fondo en núcleos pequeños y despertar los grandes solo para ráfagas interactivas extiende la autonomía significativamente.
- *Centros de datos verdes*: aparecen CPUs con núcleos solo de eficiencia (Intel Xeon 6 con E-cores, ARM Neoverse). El planificador y el orquestador pueden colocar microservicios de baja intensidad en núcleos eficientes y reducir el consumo por solicitud.

**Trade-offs**

1. *Energía vs. latencia*: colocar una tarea en un núcleo pequeño ahorra energía pero si la carga crece de golpe, la migración a un núcleo grande añade latencia (y caché fría).
2. *Precisión del modelo vs. complejidad*: EAS depende de un modelo energético correcto; si el modelo del fabricante es inexacto, las decisiones empeoran el consumo.

**Supuestos de hardware**: SoC con al menos dos tipos de núcleo que comparten el mismo ISA, con DVFS por clúster y un modelo de energía expuesto vía Device Tree o ACPI.

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
