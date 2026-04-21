Este código fuente implementa una arquitectura modular basada en eventos y máquinas de estados finitos (FSM), diseñada típicamente para sistemas embebidos en lenguaje C. Separa de forma clara la lógica de control del sistema de la lógica de hardware (actuadores).

A continuación, detallo el funcionamiento de cada archivo y cómo interactúan entre sí.

## 1. Definición de Atributos (Cabeceras .h) {#definición-de-atributos-cabeceras-.h}

Estos archivos actúan como diccionarios que definen el \"vocabulario\" (estados y eventos) y las estructuras de memoria para cada tarea.

### task_system_attribute.h {#task_system_attribute.h}

Este archivo define las características de la tarea principal de control del sistema.

- Define los eventos que pueden excitar al sistema: EV_SYS_IDLE (reposo) y EV_SYS_ACTIVE (activo).

- Define los estados en los que puede encontrarse el sistema: ST_SYS_IDLE y ST_SYS_ACTIVE.

- Crea la estructura de datos task_system_dta_t que almacena el estado actual, el último evento recibido, un contador de tiempo (tick) y una bandera booleana (flag) para indicar si hay un evento pendiente.

### task_actuator_attribute.h {#task_actuator_attribute.h}

Este archivo define las características para la tarea encargada de manejar un componente físico, en este caso, un LED.

- Establece los eventos del actuador: EV_LED_IDLE y EV_LED_ACTIVE.

- Establece los estados del actuador: ST_LED_IDLE y ST_LED_ACTIVE.

- Define identificadores únicos para los actuadores, como ID_LED_A.

- Incluye una estructura de configuración task_actuator_cfg_t que permite mapear el actuador a un puerto (GPIO_TypeDef \* gpio_port) y pin específico del microcontrolador.

- Contiene la estructura de datos dinámica task_actuator_dta_t para almacenar el estado, evento, ticks y flags del actuador durante la ejecución.

## 2. Gestión de Eventos (Interfaces .c) {#gestión-de-eventos-interfaces-.c}

Estos archivos manejan cómo las distintas partes del programa se comunican y se envían comandos entre sí sin bloquear la ejecución.

### task_system_interface.c {#task_system_interface.c}

Implementa una cola de eventos (FIFO circular) para aislar la generación de eventos de su procesamiento en el sistema central.

- Utiliza una estructura con un buffer de tamaño QUEUE_LENGTH (16 elementos) para almacenar eventos.

- Proporciona la función init_event_task_system para inicializar los contadores y limpiar la cola con el valor EMPTY.

- Utiliza put_event_task_system para que fuentes externas inserten nuevos eventos en la cabeza de la cola circular.

- Utiliza get_event_task_system para que la tarea central extraiga el evento más antiguo desde la cola.

- Incluye any_event_task_system para verificar rápidamente si existen eventos pendientes por procesar.

### task_actuator_interface.c {#task_actuator_interface.c}

Proporciona la vía de comunicación mediante la cual el sistema le da órdenes al actuador.

- Implementa la función put_event_task_actuator que recibe el evento deseado y el identificador del actuador.

- A diferencia del sistema, no utiliza una cola circular; en su lugar, escribe directamente el evento en el índice correspondiente del arreglo global task_actuator_dta_list y levanta la bandera (flag = true).

## 3. Lógica Principal (El Archivo Central .c) {#lógica-principal-el-archivo-central-.c}

### task_system.c {#task_system.c}

Este archivo contiene el \"cerebro\" de la operación. Implementa la máquina de estados que toma decisiones basándose en los eventos recibidos.

- Posee la función task_system_init, encargada de arrancar el sistema, inicializar la cola de eventos, configurar el modo en NORMAL y establecer el estado de inicio en ST_SYS_IDLE.

- Posee la función task_system_update, diseñada para ejecutarse periódicamente (de forma no bloqueante, idealmente cada 1 milisegundo) y evaluar la máquina de estados activa.

- Implementa la lógica core en task_system_normal_statechart.

- Dentro del *statechart*, primero revisa si hay nuevos eventos en la cola mediante la interfaz y actualiza sus variables internas.

- Si el sistema se encuentra en estado ST_SYS_IDLE y procesa un evento EV_SYS_ACTIVE, cambia su propio estado a ST_SYS_ACTIVE y envía una orden EV_LED_ACTIVE al actuador.

- De manera inversa, si está en estado ST_SYS_ACTIVE y recibe EV_SYS_IDLE, retorna al estado ST_SYS_IDLE y manda al actuador la orden EV_LED_IDLE.

**Resumen del flujo de trabajo:**

Algún botón o sensor externo genera un evento (ej. EV_SYS_ACTIVE) y lo coloca en la cola del sistema (task_system_interface). La rutina periódica de actualización (task_system.c) lee la cola, nota el cambio, actualiza su estado interno y dispara un comando directo (task_actuator_interface) para encender el hardware configurado (task_actuator_attribute).

Basado en el código proporcionado, aquí está la evolución detallada de las variables al iniciar el sistema y durante su ciclo de ejecución.

A modo de aclaración previa, el valor máximo de la variable index está dictado por SYSTEM_DTA_QTY, que toma el valor de MODE_QTY. Según el enumerador task_system_mode_t, NORMAL vale 0 y MODE_QTY vale 1. Por lo tanto, el sistema gestiona un único set de datos en el índice 0.

### 1. Inicialización (task_system_init()) {#inicialización-task_system_init}

Al ejecutar la función de inicialización, las variables se configuran de la siguiente manera:

- **index**: Comienza en 0 para el bucle for. El bucle se ejecuta una sola vez (ya que SYSTEM_DTA_QTY es 1). Al salir de esta función, la variable temporal index se descarta.

- **task_system_dta_list\[0\].tick**: Aunque no se inicializa de forma explícita en esta función, al ser parte de un arreglo global, el compilador lo inicializa en 0. La unidad de medida esperada es en **milisegundos (mS)**, tal como lo sugieren el comentario del archivo (Update by Time Code, period = 1mS) y la impresión del LOGGER_INFO que utiliza HAL_GetTick() indicando Tick \[mS\].

- **task_system_dta_list\[0\].state**: Se le asigna el valor ST_SYS_IDLE (estado de reposo).

- **task_system_dta_list\[0\].event**: Se le asigna el valor EV_SYS_IDLE.

- **task_system_dta_list\[0\].flag**: Se le asigna el valor booleano false.

### 2. Sucesivas ejecuciones sin eventos en la cola (task_system_update()) {#sucesivas-ejecuciones-sin-eventos-en-la-cola-task_system_update}

Si el ciclo principal (loop) llama a task_system_update() y no ha ocurrido ningún evento externo:

- **index**: La función ya no usa un iterador dinámico; accede directamente al dato utilizando la constante NORMAL (que equivale al índice 0).

- **task_system_dta_list\[0\].tick**: Permanece en 0. En la implementación provista de task_system_normal_statechart(), no hay lógica que incremente ni actualice esta variable durante el flujo normal.

- **task_system_dta_list\[0\].state**: Sigue siendo ST_SYS_IDLE.

- **task_system_dta_list\[0\].event**: Se mantiene en el último evento conocido (EV_SYS_IDLE).

- **task_system_dta_list\[0\].flag**: Sigue en false, ya que la función any_event_task_system() devuelve falso.

### 3. Ejecución al recibir un evento de activación (EV_SYS_ACTIVE) {#ejecución-al-recibir-un-evento-de-activación-ev_sys_active}

Supongamos que una interrupción o botón inserta un evento en la cola y se ejecuta task_system_update():

- La función any_event_task_system() ahora evalúa a true.

- **task_system_dta_list\[0\].flag**: Cambia inmediatamente a true.

- **task_system_dta_list\[0\].event**: Pasa a ser EV_SYS_ACTIVE (valor obtenido de get_event_task_system()).

- Dentro del bloque switch (que evalúa el estado actual ST_SYS_IDLE), se cumple la condición if ((true == flag) && (EV_SYS_ACTIVE == event)). Como consecuencia:

  - **task_system_dta_list\[0\].flag**: Vuelve a pasar a false (el evento fue consumido).

  - **task_system_dta_list\[0\].state**: Transiciona a ST_SYS_ACTIVE.

  - Se envía el comando de encendido (EV_LED_ACTIVE) al actuador.

- **task_system_dta_list\[0\].tick**: Continúa inalterada.

### 4. Ejecución al recibir un evento de reposo (EV_SYS_IDLE) estando activos {#ejecución-al-recibir-un-evento-de-reposo-ev_sys_idle-estando-activos}

El sistema ahora está en ST_SYS_ACTIVE. Si llega un nuevo evento:

- Se detecta el evento en la cola, por lo que **task_system_dta_list\[0\].flag** se pone en true y **task_system_dta_list\[0\].event** toma el valor EV_SYS_IDLE.

- En el bloque switch para el caso ST_SYS_ACTIVE, se cumple la condición y ocurre lo siguiente:

  - **task_system_dta_list\[0\].flag**: Vuelve a ser false.

  - **task_system_dta_list\[0\].state**: Retorna a ST_SYS_IDLE.

  - Se envía el comando de apagado (EV_LED_IDLE) al actuador.

*(Nota sobre el tick: El código fuente muestra la macro DEL_SYS_MIN asignada a tick únicamente dentro del caso default del switch, el cual representa una condición de error. En la operación normal dictada por esta FSM, la variable temporal tick no se está utilizando).*

Con respecto a tu consulta, es importante realizar una aclaración inicial: en el código fuente proporcionado **no existe** una función llamada task_system_statechart(uint32_t index). En su lugar, la función encargada de manejar la máquina de estados se llama void task_system_normal_statechart(void) y no recibe parámetros.

A continuación, explico el comportamiento de esta función existente y la evolución de las variables de la cola de eventos.

### 1. Comportamiento de la función task_system_normal_statechart(void) {#comportamiento-de-la-función-task_system_normal_statechartvoid}

Esta función es el núcleo lógico (la máquina de estados finitos) de la tarea del sistema. Su comportamiento en cada ejecución es el siguiente:

- **Puntero de datos:** Primero, obtiene un puntero (p_task_system_dta) a la estructura de datos correspondiente al modo NORMAL (índice 0).

- **Lectura de eventos:** Llama a la función any_event_task_system() para verificar si hay eventos pendientes en la cola. Si los hay (retorna true), levanta la bandera interna (flag = true) y extrae el evento de la cola usando get_event_task_system(), guardándolo en p_task_system_dta-\>event.

- **Evaluación de la máquina de estados (Switch):** Evalúa el estado actual (p_task_system_dta-\>state):

  - **Caso ST_SYS_IDLE (Reposo):** Si la bandera es true y el evento recibido es EV_SYS_ACTIVE, consume el evento bajando la bandera (flag = false), envía una orden de encendido al actuador (put_event_task_actuator(EV_LED_ACTIVE, ID_LED_A)) y cambia su estado a ST_SYS_ACTIVE.

  - **Caso ST_SYS_ACTIVE (Activo):** Si la bandera es true y el evento recibido es EV_SYS_IDLE, consume el evento (flag = false), envía una orden de apagado al actuador (put_event_task_actuator(EV_LED_IDLE, ID_LED_A)) y vuelve al estado ST_SYS_IDLE.

  - **Caso default (Error):** Si cae en un estado no reconocido, reinicia las variables de la máquina a sus valores por defecto (estado en reposo, sin eventos y bandera en false).

### 2. Evolución de las variables de la cola (event_task_system_queue) {#evolución-de-las-variables-de-la-cola-event_task_system_queue}

La cola está implementada como un buffer circular (FIFO) en task_system_interface.c. Así evolucionan sus variables a lo largo de la ejecución:

#### A. Al inicio (task_system_init()) {#a.-al-inicio-task_system_init}

Cuando el sistema arranca, se llama a init_event_task_system(), la cual inicializa las variables así:

- **i:** Es una variable local utilizada únicamente dentro del ciclo for para limpiar el arreglo. Inicia en 0 y llega hasta 15 (dado que QUEUE_LENGTH es 16). Al salir de la función, esta variable se destruye.

- **event_task_system_queue.head (Cabeza):** Se inicializa en 0. Indica el índice donde se escribirá el *próximo* evento.

- **event_task_system_queue.tail (Cola):** Se inicializa en 0. Indica el índice desde donde se leerá el *próximo* evento.

- **event_task_system_queue.count (Contador):** Se inicializa en 0. Indica que no hay elementos válidos en la cola.

- **event_task_system_queue.queue\[i\]:** A través del bucle for, los 16 elementos del arreglo se sobrescriben con el valor constante EMPTY (definido como 255ul).

#### B. Sucesivas ejecuciones del loop (task_system_update()) SIN nuevos eventos {#b.-sucesivas-ejecuciones-del-loop-task_system_update-sin-nuevos-eventos}

Si ninguna fuente externa ha introducido eventos en el sistema:

- La función any_event_task_system() evalúa (head != tail). Como ambos valen 0, retorna false.

- Las variables head, tail, count y el contenido de queue\[\] **se mantienen intactos** sin ningún cambio.

#### C. Evolución cuando ingresa un evento externo (put_event_task_system()) {#c.-evolución-cuando-ingresa-un-evento-externo-put_event_task_system}

Aunque no es llamado por el update, cuando un agente externo (ej. botón) inserta un evento:

- **count:** Se incrementa en 1.

- **queue\[head\]:** En la posición indicada por head, se sobrescribe el valor EMPTY con el nuevo evento recibido.

- **head:** Se incrementa en 1, moviéndose a la siguiente posición vacía. Si llega a valer 16 (QUEUE_LENGTH), vuelve a cero (0) para lograr el comportamiento circular.

#### D. Evolución cuando el loop procesa el evento (get_event_task_system()) {#d.-evolución-cuando-el-loop-procesa-el-evento-get_event_task_system}

En la siguiente ejecución de task_system_update(), se detectará que head != tail, por lo que se llamará a get_event_task_system(). En ese momento:

- **count:** Se decrementa en 1.

- **Evento extraído:** Se lee el evento de queue\[tail\].

- **queue\[tail\]:** Esa posición que acaba de ser leída se limpia, sobrescribiéndose nuevamente con EMPTY.

- **tail:** Se incrementa en 1, moviendo el puntero de lectura al siguiente evento más antiguo. Al igual que head, si tail alcanza el límite de 16, vuelve a cero (0).

Basado en el código fuente proporcionado de task_actuator_interface.c y el contexto de task_system.c, a continuación se detalla cómo evolucionan las variables relacionadas con el actuador (identifier, event y flag).

Es importante notar que identifier es un parámetro local de la función put_event_task_actuator(), por lo que solo \"existe\" y toma valor durante el breve instante en que esta función es llamada.

### 1. Durante la inicialización (task_system_init()) {#durante-la-inicialización-task_system_init}

Al ejecutarse task_system_init():

- La función de inicialización del sistema **no interactúa** con el actuador ni llama a put_event_task_actuator().

- Por lo tanto, identifier no entra en juego en esta etapa.

- Dado que task_actuator_dta_list es un arreglo global de estructuras, sus variables internas son inicializadas por defecto por el compilador de C antes de arrancar la aplicación. Esto significa que task_actuator_dta_list\[ID_LED_A\].event iniciará en 0 (que equivale a EV_LED_IDLE en el enumerador) y task_actuator_dta_list\[ID_LED_A\].flag iniciará en false.

### 2. Sucesivas ejecuciones del loop SIN eventos (task_system_update()) {#sucesivas-ejecuciones-del-loop-sin-eventos-task_system_update}

Mientras la aplicación corre periódicamente pero el sistema central no recibe ningún evento externo:

- La máquina de estados central se mantiene en ST_SYS_IDLE y no emite órdenes.

- Al no llamarse a put_event_task_actuator(), la variable temporal identifier no se evalúa.

- **task_actuator_dta_list\[ID_LED_A\].event**: Permanece en EV_LED_IDLE.

- **task_actuator_dta_list\[ID_LED_A\].flag**: Permanece en false.

### 3. Evolución al recibir el sistema un evento de activación (EV_SYS_ACTIVE) {#evolución-al-recibir-el-sistema-un-evento-de-activación-ev_sys_active}

Cuando la máquina de estados principal detecta un evento EV_SYS_ACTIVE estando en reposo, cambia su estado y ejecuta la orden put_event_task_actuator(EV_LED_ACTIVE, ID_LED_A). En ese momento dentro de task_actuator_interface.c:

- **identifier**: Toma el valor constante ID_LED_A. Este índice se usa para obtener el puntero a la estructura del LED A (&task_actuator_dta_list\[identifier\]).

- **task_actuator_dta_list\[ID_LED_A\].event**: Se sobrescribe con el valor del parámetro enviado, pasando a ser EV_LED_ACTIVE.

- **task_actuator_dta_list\[ID_LED_A\].flag**: Cambia su estado a true, indicándole a la tarea del actuador (que en algún momento correrá su propio *update*) que tiene un evento pendiente por procesar.

### 4. Evolución al recibir el sistema un evento de apagado (EV_SYS_IDLE) {#evolución-al-recibir-el-sistema-un-evento-de-apagado-ev_sys_idle}

Cuando la máquina de estados principal está activa y recibe un evento para volver al reposo (EV_SYS_IDLE), ejecuta la orden put_event_task_actuator(EV_LED_IDLE, ID_LED_A). En ese momento:

- **identifier**: Nuevamente toma el valor ID_LED_A para direccionar al mismo componente.

- **task_actuator_dta_list\[ID_LED_A\].event**: Pasa a tomar el valor EV_LED_IDLE.

- **task_actuator_dta_list\[ID_LED_A\].flag**: Se vuelve a poner en true (suponiendo que la tarea del actuador ya lo había bajado a false tras procesar el encendido), notificando así que hay una nueva orden (apagar) lista para ser leída.
