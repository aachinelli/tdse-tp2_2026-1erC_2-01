A continuación, presento un análisis detallado del funcionamiento de cada uno de los archivos solicitados, los cuales en su conjunto implementan la base de un sistema \"Bare Metal\" disparado por eventos de tiempo para microcontroladores ARM Cortex.

## 1. app.c: Planificador y Ejecución de Tareas {#app.c-planificador-y-ejecución-de-tareas}

Este archivo contiene el núcleo del sistema, actuando como un planificador que decide cuándo deben ejecutarse las tareas de la aplicación.

- **Definición de Tareas:** Define una lista de tareas (sensor, sistema y actuador) utilizando estructuras task_cfg_t que contienen punteros a las funciones de inicialización y actualización de cada tarea.

- **Monitoreo de Rendimiento:** Utiliza la estructura task_dta_t para almacenar el número de ejecuciones (NOE) y medir el tiempo de ejecución en microsegundos, registrando el último tiempo (LET), el mejor tiempo (BCET) y el peor tiempo (WCET).

- **Inicialización (app_init):** Se encarga de inicializar los contadores, el contador de ciclos por hardware, y llama a la función de inicio de cada tarea en la lista, reiniciando sus estadísticas. Además, inicializa el contador de \"ticks\" desactivando y activando interrupciones brevemente para proteger el recurso compartido.

- **Bucle Principal (app_update):** Verifica constantemente si es momento de ejecutar las tareas observando la variable g_app_tick_cnt de manera segura contra interrupciones. Si hay actualizaciones de tiempo pendientes, recorre cada tarea, reinicia el contador de ciclos, ejecuta la función de actualización de la tarea, y luego lee el tiempo transcurrido para actualizar las estadísticas de rendimiento (LET, BCET, WCET).

- **Bajo Consumo:** Tras ejecutar todas las tareas pendientes de ese ciclo, pone al procesador en modo de bajo consumo (HAL_PWR_EnterSLEEPMode(0, PWR_SLEEPENTRY_WFI)) hasta que ocurra la siguiente interrupción.

- **Control de Tiempo (HAL_SYSTICK_Callback):** Es la rutina que incrementa la variable g_app_tick_cnt cada vez que el temporizador del sistema genera una interrupción.

## 2. logger.c y logger.h: Sistema de Registro (Logging) {#logger.c-y-logger.h-sistema-de-registro-logging}

Estos archivos implementan una herramienta segura para enviar mensajes de diagnóstico o información durante la ejecución del programa.

- **Configuración y Macros:** En logger.h se definen las macros LOGGER_LOG y LOGGER_INFO para formatear mensajes usando snprintf.

- **Protección de Interrupciones:** Las macros de registro envuelven la impresión del mensaje con instrucciones en ensamblador (\_\_asm(\"CPSID i\") y \_\_asm(\"CPSIE i\")) para deshabilitar temporalmente las interrupciones, garantizando que el proceso de registro no sea interrumpido a la mitad.

- **Salida de Datos:** Dependiendo de la configuración LOGGER_CONFIG_USE_SEMIHOSTING, logger.c envía los mensajes formateados utilizando la función estándar printf hacia la consola de depuración (semihosting), o simplemente no hace nada si está deshabilitado.

## 3. systick.c: Retardos Bloqueantes de Precisión {#systick.c-retardos-bloqueantes-de-precisión}

Este archivo proporciona funciones para detener la ejecución del microcontrolador durante un tiempo exacto.

- **Retardo en Microsegundos (systick_delay_us):** Calcula la cantidad de ciclos del reloj del sistema que equivalen al tiempo solicitado en microsegundos.

- **Bucle de Espera:** Lee repetidamente el valor del registro SysTick-\>VAL y calcula el tiempo transcurrido desde el inicio.

- **Manejo de Desbordamiento:** El código contempla de manera segura el momento en que el contador SysTick llega a cero y se recarga (wrap around), asegurando que el retardo sea exacto incluso si esto ocurre durante la espera.

## 4. dwt.h: Medición de Ciclos de Reloj por Hardware {#dwt.h-medición-de-ciclos-de-reloj-por-hardware}

El módulo DWT (Data Watchpoint and Trace) es fundamental para perfilar el código de manera no intrusiva y con precisión de un ciclo de reloj.

- **Funciones en Línea (inline):** Todas las funciones están declaradas como static inline y \_\_attribute\_\_((always_inline)) para evitar el costo de saltar a una función en ensamblador, lo cual es crucial cuando se miden tiempos de ejecución muy cortos.

- **Inicialización (cycle_counter_init):** Activa el hardware DWT escribiendo en el registro DEMCR, reinicia el contador CYCCNT a cero y habilita el conteo.

- **Lectura de Tiempo (cycle_counter_get_time_us):** Lee la cantidad exacta de ciclos ejecutados desde el último reinicio y los divide por la velocidad del reloj del sistema (SystemCoreClock / 1000000) para devolver el tiempo transcurrido en microsegundos.

- **Ejemplo Práctico:** El archivo incluye comentarios que muestran cómo usar estas macros junto con pines GPIO para medir el tiempo que toma ejecutar un bloque de código, un método clásico de depuración y validación en sistemas embebidos.

Basado en el código fuente proporcionado en app.c, a continuación se detalla la evolución temporal y lógica de las variables solicitadas. Estas variables son fundamentales para el planificador disparado por eventos de tiempo y para el perfilado (profiling) del rendimiento de las tareas.

### 1. Fase de Inicialización: app_init() {#fase-de-inicialización-app_init}

Al arrancar el sistema, se llama a esta función para establecer las condiciones iniciales:

- **task_dta_list\[index\] (Estadísticas de tareas):** Se utiliza un bucle donde la variable **index** evoluciona de 0 a 2 (ya que TASK_QTY es 3, correspondiente a las tareas *sensor, system* y *actuator*). Para cada tarea, se inicializan sus variables:

  - **NOE** (Number of Execution - *adimensional*): Se inicializa en TASK_X_NOE_INI, que vale 0.

  - **LET** (Last Execution Time - *microsegundos*): Se inicializa en TASK_X_LET_INI, que vale 0.

  - **BCET** (Best-Case Execution Time - *microsegundos*): Se inicializa en TASK_X_BCET_INI, que tiene un valor alto artificial de 1000 para garantizar que la primera lectura de tiempo real lo sobrescriba.

  - **WCET** (Worst-Case Execution Time - *microsegundos*): Se inicializa en TASK_X_WCET_INI, que vale 0.

- **g_app_tick_cnt** (Contador de Ticks - *adimensional*): Se inicializa en G_APP_TICK_CNT_INI, que vale 0, dentro de una sección crítica (con interrupciones deshabilitadas mediante \_\_asm(\"CPSID i\")) para proteger el recurso.

### 2. Eventos Asíncronos: HAL_SYSTICK_Callback() {#eventos-asíncronos-hal_systick_callback}

Paralelamente a la ejecución principal, el temporizador del sistema genera interrupciones periódicas (típicamente cada 1 milisegundo).

- **g_app_tick_cnt:** En cada interrupción, esta variable **se incrementa en 1** (g_app_tick_cnt++). Esto indica al bucle principal que ha transcurrido un \"tick\" de tiempo y las tareas deben ejecutarse.

### 3. Bucle Principal: Sucesivas ejecuciones de app_update() {#bucle-principal-sucesivas-ejecuciones-de-app_update}

Cuando el programa principal llama a app_update(), ocurre la siguiente evolución:

- **Comprobación del Tick:** El sistema verifica de forma segura si g_app_tick_cnt es mayor a 0. Si es así, **decrementa g_app_tick_cnt** en 1 y activa la bandera b_time_update_required para entrar al bucle de tareas.

- **Preparación del ciclo:** \* **g_app_runtime_us** (Tiempo de ejecución total del ciclo - *microsegundos*): Se **reinicia a 0** al comienzo de cada ciclo de ejecución de tareas.

- **Ejecución de las tareas (for loop):** La variable **index** vuelve a evolucionar de 0 a 2 para recorrer cada tarea secuencialmente. Por cada tarea:

  - Se reinicia el contador de ciclos de hardware (cycle_counter_reset()) y se ejecuta la tarea.

  - **task_dta_list\[index\].NOE:** Se **incrementa en 1** (++), reflejando que la tarea se ejecutó una vez más.

  - **task_dta_list\[index\].LET:** Evoluciona tomando el valor exacto en **microsegundos** devuelto por cycle_counter_get_time_us(), el cual mide cuánto tardó la tarea en ejecutarse en esta iteración específica.

  - **task_dta_list\[index\].BCET:** Se compara con el LET actual. Si el LET es **menor** que el BCET almacenado, el BCET evoluciona y se actualiza con el nuevo valor del LET. (Nota: Como se inicializó en 1000, la primera ejecución casi seguramente será menor y actualizará este valor).

  - **task_dta_list\[index\].WCET:** Se compara con el LET actual. Si el LET es **mayor** que el WCET almacenado, el WCET evoluciona y se actualiza con el nuevo valor del LET.

  - **g_app_runtime_us:** Evoluciona acumulando los tiempos de ejecución individuales. Se le **suma el LET** de la tarea actual (g_app_runtime_us += task_dta_list\[index\].LET;). Al final del bucle for, esta variable contendrá la suma del tiempo que tomó ejecutar las 3 tareas juntas.

- **Final del ciclo:** Antes de terminar la iteración del while(b_time_update_required), el sistema vuelve a comprobar g_app_tick_cnt. Si durante la ejecución de las tareas ocurrió otra interrupción y se acumuló otro tick, vuelve a decrementar la variable y el ciclo while se repite; de lo contrario, sale de la función hasta el próximo llamado.

El uso de la macro LOGGER_INFO() dentro de la función de actualización de una tarea (task_update) tendrá un **impacto drástico y negativo** en las mediciones de rendimiento almacenadas en g_app_runtime_us y task_dta_list\[index\].WCET.

El impacto se explica por cómo está construido el sistema de logging y cómo el planificador mide los tiempos:

### 1. El costo computacional de LOGGER_INFO() {#el-costo-computacional-de-logger_info}

- **Formateo de cadenas y Semihosting:** La macro LOGGER_INFO() utiliza internamente snprintf para armar el mensaje. Luego, envía este mensaje utilizando la función logger_log_print\_, la cual ejecuta un printf y un fflush(stdout). Estas operaciones (especialmente el *semihosting* hacia una consola de depuración) son increíblemente lentas en un microcontrolador y bloquean el procesador durante miles o millones de ciclos de reloj.

- **Bloqueo de interrupciones:** Para evitar corrupción de datos, LOGGER_INFO() deshabilita las interrupciones al inicio de su ejecución (\_\_asm(\"CPSID i\")) y las rehabilita al final (\_\_asm(\"CPSIE i\")). El procesador estará \"ciego\" a otros eventos durante este largo periodo.

### 2. Impacto en task_dta_list\[index\].WCET (Worst-Case Execution Time) {#impacto-en-task_dta_listindex.wcet-worst-case-execution-time}

- El archivo app.c calcula el tiempo exacto que tarda una tarea (LET) usando el contador de ciclos de hardware (cycle_counter_get_time_us()).

- Si se llama a LOGGER_INFO() dentro de una tarea, el tiempo que tome el formateo y envío del mensaje por semihosting quedará incluido en el cálculo del LET de esa iteración en particular.

- Como el sistema compara constantemente el LET actual con el WCET para quedarse siempre con el valor más alto (el peor caso), **el WCET de la tarea sufrirá un incremento gigantesco**. Este valor distorsionará las estadísticas, ya que el peor tiempo registrado estará dominado por el retraso introducido por la consola de depuración y no por la lógica real de la tarea.

### 3. Impacto en g_app_runtime_us (Tiempo de ejecución total) {#impacto-en-g_app_runtime_us-tiempo-de-ejecución-total}

- La variable g_app_runtime_us se encarga de acumular (sumar) los tiempos de ejecución (LET) de todas las tareas que se corren en ese \"tick\".

- Al inflarse el LET de la tarea que imprimió el log, **el valor total de g_app_runtime_us se disparará en ese ciclo**.

- Dado que el sistema está diseñado para ejecutarse cada 1 ms, es muy probable que este retraso haga que g_app_runtime_us supere largamente los 1000 microsegundos (1 ms), provocando que la aplicación no pueda cumplir con sus tiempos límite (*deadlines*) y se pierdan \"ticks\" del SysTick debido a que las interrupciones permanecieron deshabilitadas.
