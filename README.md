# Laboratorio-Task-Struct-5
# Informe de Laboratorio: El Modelo Fork-Exec-Wait
## Gestión y creación de procesos en Linux bajo el capó

## 1. Introducción y el Ciclo de Vida del Proceso
El modelo de gestión de procesos en sistemas Unix/Linux se sustenta en cuatro pilares fundamentales de llamadas al sistema[cite: 38]:
* **`fork()`:** Clona el proceso actual, creando un proceso hijo exacto pero independiente[cite: 38].
* **`exec()`:** Reemplaza por completo la imagen de memoria del proceso hijo con un programa ejecutable nuevo[cite: 38].
* **`exit()`:** Finaliza la ejecución del proceso y reporta un estado de salida al sistema[cite: 38].
* **`wait()`:** Permite al proceso padre recoger el resultado de finalización y limpiar los rastros del hijo en la tabla de procesos[cite: 38].

## 2. Optimización de Memoria: Copy-on-Write (CoW)
* **El problema histórico:** Antiguamente, `fork()` duplicaba físicamente toda la memoria RAM del padre, lo cual tomaba segundos en procesos pesados[cite: 40].
* **La solución moderna (CoW):** El kernel implementa una copia semántica donde padre e hijo comparten inicialmente las mismas páginas físicas marcadas estrictamente como de solo lectura (*Read-Only*)[cite: 41, 42].
* **Page Fault:** Si alguno de los procesos intenta modificar una página de memoria, el hardware y el kernel detectan la violación de escritura (*Page Fault*), asignan un marco nuevo, copian únicamente esa página específica y la remapean de forma transparente[cite: 42].

## 3. Transición de Imagen y Limpieza (`execve` vs. `_exit`)
* **`execve()`:** Sustituye el espacio de direcciones (texto, datos, pila y montículo) conservando intactos el PID y los descriptores de archivos abiertos[cite: 43]. Una regla crítica es que `exec` **nunca retorna** si tiene éxito; si el código posterior se ejecuta, significa que el binario falló[cite: 43].
* **`_exit(status)`:** Syscall directa de emergencia que termina el proceso de inmediato **sin vaciar los búferes** de la librería estándar[cite: 46]. Es vital utilizar siempre `_exit()` si una llamada a `exec` falla en el hijo para evitar duplicar flujos de salida heredados del padre[cite: 46].

## 4. Gestión de Zombies y Drenaje Asíncrono (`waitpid` / `SIGCHLD`)
* **Proceso Zombie (`Z`):** Ocurre cuando un hijo termina su ejecución, pero el padre aún no invoca `wait()`, reteniendo una entrada valiosa en la tabla de procesos[cite: 47].
* **Patrón Asíncrono:** En servidores de alto tráfico que no pueden permitirse bloquear el hilo principal con un `wait()` síncrono, se implementa un manejador para la señal `SIGCHLD` utilizando un bucle con `waitpid(-1, &status, WNOHANG)` para drenar todos los procesos terminados de forma no bloqueante[cite: 48].

## 5. Conclusión
El paradigma `fork-exec-wait`, respaldado por mecanismos eficientes como *Copy-on-Write* y una correcta sincronización de estados, constituye la arquitectura base para la creación segura de procesos, la ejecución de intérpretes de comandos (*shells*) y la prevención de anomalías críticas como la saturación de identificadores por procesos zombie en entornos de producción.
