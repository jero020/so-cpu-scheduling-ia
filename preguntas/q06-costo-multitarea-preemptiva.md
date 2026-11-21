# Q6. El Coste Oculto de la Multitarea Preemptiva

> Evalúe el impacto acumulado del cambio de contexto y la invalidación de memorias caché (L1/L2/L3) frente al paradigma de concurrencia cooperativa orientada a eventos. ¿Por qué arquitecturas de alto rendimiento (ej. NGINX, Node.js) prefieren modelos mono-hilo no bloqueantes frente a miles de hilos preemptivos?

**Responsable:** Jerónimo Vélez Acosta · **Estado:** ☑ Prompt ☑ Respuesta IA ☑ Crítica individual ☑ Solución unificada

---

## 1. Prompt a la IA

**Modelo usado:** Claude Opus 5.5 (Anthropic), 2026-11-20

```text
Actúa como arquitecto senior de sistemas operativos con experiencia en el planificador del kernel de Linux y en sistemas de producción.

Contexto: prueba final de un curso universitario de Sistemas Operativos (Ingeniería de Sistemas). Responde con rigor técnico, citando mecanismos reales (nombres de políticas, syscalls, parámetros del kernel y versiones cuando aplique).

Pregunta: El Coste Oculto de la Multitarea Preemptiva. Evalúe el impacto acumulado del cambio de contexto y la invalidación de memorias caché (L1/L2/L3) frente al paradigma de concurrencia cooperativa orientada a eventos. ¿Por qué arquitecturas de alto rendimiento (ej. NGINX, Node.js) prefieren modelos mono-hilo no bloqueantes frente a miles de hilos preemptivos?

Restricciones:
- Máximo 600 palabras.
- Explica al menos dos trade-offs con su costo concreto.
- Incluye al menos un ejemplo de sistema real en producción.
- Indica explícitamente qué supuestos de hardware estás haciendo.
- Da órdenes de magnitud del costo de un cambio de contexto y explica de dónde salen.
- Describe con precisión el modelo de hilos real de NGINX y de Node.js.
```

## 2. Respuesta de la IA

**El costo real de un cambio de contexto**

- **Costo directo**: guardar y restaurar registros, cambiar de pila, ejecutar el planificador y, entre procesos distintos, cambiar el espacio de direcciones (CR3). En hardware moderno es del orden de **1–5 µs**. Con KPTI (mitigación de Meltdown) cada entrada al kernel cuesta más, aunque PCID reduce el vaciado del TLB.
- **Costo indirecto**, que suele dominar: el hilo que entra encuentra las cachés L1/L2 y el TLB llenos de datos del hilo anterior. Recalentarlos puede costar decenas o cientos de µs según el tamaño del working set. Las mediciones clásicas (Li, Ding y Shen, 2007) mostraron que el costo indirecto puede ser varias veces el directo.

Con miles de hilos activos (modelo *thread-per-connection*), el sistema pasa una fracción creciente del tiempo cambiando de contexto, cada hilo reserva su pila (típicamente 8 MB de memoria virtual en Linux) y la contención por locks aumenta.

**El modelo orientado a eventos**

Un *event loop* atiende miles de conexiones con un solo hilo: pregunta al kernel qué descriptores están listos (`epoll` en Linux, `kqueue` en BSD) y ejecuta el manejador de cada evento hasta que vuelve a bloquear de forma lógica (no física). No hay preempción entre manejadores: el cambio entre “tareas” es una llamada a función, y la caché permanece caliente.

**Cómo es realmente el modelo de NGINX y Node.js**

- **NGINX**: un proceso *master* y varios procesos *worker* (normalmente uno por núcleo, `worker_processes auto`). Cada worker es mono-hilo con su propio event loop sobre `epoll`. Para operaciones de disco que bloquearían, desde la versión 1.7.11 puede usar *thread pools* (`aio threads`).
- **Node.js**: el código JavaScript corre en un único hilo con el event loop de **libuv**, pero libuv mantiene un *thread pool* (4 hilos por defecto, `UV_THREADPOOL_SIZE`) para operaciones de sistema de archivos, DNS (`getaddrinfo`), compresión y criptografía. Para paralelismo de CPU existen `worker_threads` y `cluster`.

Es decir, ninguno es “mono-hilo puro”: usan un hilo por núcleo para el trabajo de red y delegan lo bloqueante.

**Trade-offs**

1. *Eventos vs. tareas CPU-bound*: un manejador que hace cómputo pesado bloquea todo el event loop y dispara la latencia de todas las conexiones. Los hilos preemptivos sí protegen contra esto.
2. *Simplicidad de programación*: el modelo de eventos obliga a escribir código asíncrono (callbacks, promesas); los hilos permiten código secuencial más fácil de razonar, a costa de sincronización.

**Ejemplo real**: el “problema C10K” (Dan Kegel, 1999) motivó este cambio; NGINX desplazó a Apache con `mpm_prefork` en servidores de alta concurrencia porque mantiene memoria y latencia estables con decenas de miles de conexiones.

**Supuestos de hardware**: servidor x86-64 multinúcleo, cargas mayoritariamente I/O-bound (red), cachés privadas por núcleo.

## 3. Análisis crítico individual

### Jerónimo Vélez Acosta

- **Aciertos y cumplimiento:** Desglosa con rigor los costos directos (registros, cambio de `CR3`, impacto de KPTI y mitigación vía PCID) e indirectos (invalidación de L1/L2 y TLB, citando el estudio empírico de Li et al., 2007). Además, desmitifica acertadamente la idea de que NGINX y Node.js son estrictamente mono-hilo al explicar sus *worker processes* y los pools de hilos de `libuv` / `aio`.
- **Vacíos y puntos críticos:**
  - *Omisión de `io_uring` en la evolución moderna de Linux:* El modelo describe `epoll` y los pools de hilos en espacio de usuario (`libuv`) como el estado del arte para evitar bloqueos de disco. Sin embargo, en kernels Linux modernos (≥ 5.1), **`io_uring`** transforma este paradigma al permitir I/O verdaderamente asíncrono tanto para red como para almacenamiento mediante colas compartidas en memoria (*submission/completion rings*), reduciendo drásticamente las llamadas al sistema (*syscalls*) y el overhead de cambios de modo usuario-kernel.
  - *Costos ocultos del modelo orientado a eventos:* En los trade-offs menciona el bloqueo por tareas CPU-bound y la complejidad de callbacks, pero omite la fragmentación de memoria y la propagación de contexto (*stack unwinding* limitado en depuración), así como el hecho de que las arquitecturas modernas híbridas (*green threads* / corrutinas M:N en Go o Rust/Tokio) combinan la ergonomía secuencial con la eficiencia del *event loop* sobre múltiples núcleos.

## 4. Solución unificada del equipo

### Postura del equipo

El costo de la multitarea preemptiva con miles de hilos lo dominan los efectos indirectos (cachés, TLB) más que el cambio de contexto en sí, y eso justifica los modelos orientados a eventos para cargas de E/S intensiva. Pero el estado del arte ya no es solo `epoll` + pools de hilos: hoy incluye E/S asíncrona real con `io_uring` y modelos híbridos M:N.

### Justificación

- `epoll` notifica disponibilidad y obliga a usar pools de hilos para el disco (como libuv o `aio threads` de NGINX). `io_uring` (Linux ≥ 5.1) ofrece E/S asíncrona para red y almacenamiento con colas compartidas de envío y completado, reduciendo syscalls y transiciones usuario-kernel.
- El modelo de eventos también tiene costos: bloqueo total ante tareas CPU-bound, código asíncrono más difícil de depurar (trazas de pila poco útiles) y propagación de contexto manual.
- Los *runtimes* M:N (goroutines en Go, Tokio en Rust) combinan código secuencial con la eficiencia de un event loop repartido en varios núcleos.

### Aplicación en sistemas de producción

| Carga | Modelo recomendado |
|---|---|
| Proxy o servidor web con muchas conexiones | Un worker por núcleo con event loop (NGINX); `io_uring` donde el runtime lo soporte. |
| Servicio con lógica de negocio y E/S mezcladas | Runtime M:N (Go, Tokio). |
| Cómputo intensivo | Hilos del sistema (uno por núcleo) o `worker_threads`, fuera del event loop. |

### Referencias

- Li, Ding y Shen (2007), *Quantifying the Cost of Context Switch*.
- Axboe, *Efficient IO with io_uring*; documentación de libuv y NGINX.
