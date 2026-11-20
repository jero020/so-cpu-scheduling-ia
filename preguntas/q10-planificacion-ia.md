# Q10. Planificación Asistida por Telemetría e Inteligencia Artificial

> Debata si el futuro de la planificación de procesos radica en algoritmos heurísticos estáticos o en agentes adaptativos basados en aprendizaje automático. ¿Qué riesgos de estabilidad e interpretabilidad introduce un planificador no determinista en el núcleo de un sistema operativo?

**Responsable:** _(nombre)_ · **Estado:** ☑ Prompt ☑ Respuesta IA ☐ Crítica individual ☐ Solución unificada

---

## 1. Prompt a la IA

**Modelo usado:** Claude Opus 5.5 (Anthropic), 2026-11-20

```text
Actúa como arquitecto senior de sistemas operativos con experiencia en el planificador del kernel de Linux y en sistemas de producción.

Contexto: prueba final de un curso universitario de Sistemas Operativos (Ingeniería de Sistemas). Responde con rigor técnico, citando mecanismos reales (nombres de políticas, syscalls, parámetros del kernel y versiones cuando aplique).

Pregunta: Planificación Asistida por Telemetría e Inteligencia Artificial. Debata si el futuro de la planificación de procesos radica en algoritmos heurísticos estáticos o en agentes adaptativos basados en aprendizaje automático. ¿Qué riesgos de estabilidad e interpretabilidad introduce un planificador no determinista en el núcleo de un sistema operativo?

Restricciones:
- Máximo 600 palabras.
- Explica al menos dos trade-offs con su costo concreto.
- Incluye al menos un ejemplo de sistema real en producción.
- Indica explícitamente qué supuestos de hardware estás haciendo.
- Distingue claramente lo que ya existe en producción de lo que es investigación.
- Menciona si el kernel de Linux ofrece hoy algún mecanismo para cargar planificadores personalizados.
```

## 2. Respuesta de la IA

**Lo que existe hoy en producción vs. investigación**

- **En producción**: los planificadores de los kernels de uso general (EEVDF en Linux, el de Windows, XNU en macOS) son **heurísticos y deterministas**. Usan telemetría simple (utilización con PELT, tiempo de espera, estado de energía), pero no modelos de aprendizaje automático dentro del kernel.
- **Mecanismo intermedio real**: **sched_ext**, integrado en **Linux 6.12 (2024)**, permite cargar planificadores escritos como programas **eBPF** sin recompilar el kernel. Si el planificador falla o deja tareas sin ejecutar, el kernel lo desactiva automáticamente y vuelve al planificador por defecto. Meta y Google lo usan para experimentar con políticas específicas por carga (por ejemplo, `scx_rusty`, `scx_lavd` orientado a juegos).
- **Telemetría + ML fuera del kernel**: el aprendizaje sí se usa a nivel de **clúster** (colocación de contenedores, autoscaling, predicción de demanda), por ejemplo en Borg/Kubernetes con políticas guiadas por modelos, y en trabajos académicos como Decima (MIT, 2019) para planificar trabajos en Spark.
- **Investigación**: planificadores con aprendizaje por refuerzo dentro del SO; resultados prometedores en simulación, sin adopción en kernels principales.

**El argumento a favor de agentes adaptativos**

Las heurísticas estáticas se ajustan para cargas “típicas”. Un agente que observa contadores de hardware, patrones de acceso y latencias podría especializarse en cada carga y en hardware heterogéneo, donde el espacio de decisiones (qué núcleo, qué frecuencia, cuánto slice) es muy grande.

**Riesgos de un planificador no determinista en el núcleo**

1. *Estabilidad*: un modelo puede tomar decisiones degeneradas ante entradas fuera de su distribución de entrenamiento: inanición, oscilaciones de migración, livelocks. Un error en el planificador afecta a todo el sistema.
2. *Interpretabilidad y depuración*: cuando la latencia p99 sube, hay que poder explicar por qué una tarea esperó. Con una heurística se puede razonar; con un modelo opaco, reproducir el fallo es difícil.
3. *Garantías formales*: los sistemas de tiempo real y los SLAs necesitan cotas demostrables; un modelo aprendido normalmente no las ofrece.
4. *Costo de inferencia*: el planificador corre en cada cambio de contexto (microsegundos); incluso un modelo pequeño añade overhead y presión de caché.
5. *Seguridad*: un modelo que se adapta a la carga puede ser manipulado por un proceso adversario que fabrique patrones para obtener más CPU.

**Trade-offs**

1. *Adaptabilidad vs. predictibilidad*: más especialización por carga a cambio de comportamiento menos predecible.
2. *Ganancia media vs. riesgo de cola*: pocos puntos de mejora promedio pueden no compensar un comportamiento patológico raro.

**Postura razonable**: el camino realista es híbrido. El kernel mantiene un núcleo determinista con garantías (y un mecanismo de *fallback* como el de sched_ext), y el aprendizaje ajusta **parámetros** o políticas desde fuera, con límites acotados.

**Supuestos de hardware**: servidores multinúcleo, posiblemente heterogéneos, con contadores de rendimiento (PMU) disponibles para la telemetría.

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
