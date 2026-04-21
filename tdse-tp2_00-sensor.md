Este conjunto de archivos implementa una arquitectura de software para sistemas embebidos (probablemente STM32, dada la presencia de HAL_GPIO) basada en **máquinas de estado (FSM)** y **comunicación por eventos**.

El diseño separa claramente la configuración (atributos), la lógica de los sensores y la comunicación entre tareas mediante una cola de mensajes.

## 1. Atributos y Estructuras de Datos {#atributos-y-estructuras-de-datos}

### task_sensor_attribute.h y task_system_attribute.h {#task_sensor_attribute.h-y-task_system_attribute.h}

Estos archivos definen el \"esqueleto\" de las tareas.

- **task_sensor_attribute.h**:

  - Define los estados (ST_BTN_IDLE, ST_BTN_ACTIVE) y eventos (EV_BTN_UP, EV_BTN_DOWN) de un sensor de tipo botón.

  - **task_sensor_cfg_t**: Estructura de configuración que asocia un hardware físico (puerto GPIO, pin) con señales lógicas del sistema. Permite que el código sea altamente portable y escalable.

  - **task_sensor_dta_t**: Estructura de datos dinámica que almacena el estado actual, el evento detectado y contadores de tiempo (ticks).

- **task_system_attribute.h**:

  - Define los eventos del sistema principal (EV_SYS_IDLE, EV_SYS_ACTIVE).

  - Contiene la estructura de datos para la tarea de control general (System Task).

## 2. Lógica del Sensor {#lógica-del-sensor}

### task_sensor.c {#task_sensor.c}

Este archivo implementa el comportamiento de los sensores (en este caso, un botón llamado BTN_A).

- **Inicialización (task_sensor_init)**: Configura el estado inicial de todos los sensores definidos en la lista task_sensor_cfg_list. Establece el estado de reposo (ST_BTN_IDLE).

- **Actualización (task_sensor_update)**: Es una función no bloqueante diseñada para ser llamada periódicamente (por ejemplo, cada 1ms). Recorre la lista de sensores y ejecuta su diagrama de estados.

- **Máquina de Estados (task_sensor_statechart)**:

  1.  **Lectura Física**: Lee el estado actual del pin GPIO.

  2.  **Lógica de Transición**:

      - Si está en **IDLE** y detecta que el botón se presiona (EV_BTN_DOWN), envía un evento a la tarea de sistema (put_event_task_system) y cambia a estado **ACTIVE**.

      - Si está en **ACTIVE** y detecta que el botón se suelta (EV_BTN_UP), envía el evento de desconexión y vuelve a **IDLE**.

## 3. Comunicación entre Tareas {#comunicación-entre-tareas}

### task_system_interface.c {#task_system_interface.c}

Este archivo funciona como un **Middleware** o \"buzón de mensajes\". Implementa una **cola circular (FIFO)** para que los sensores puedan comunicarse con la tarea principal del sistema sin que ambas estén \"acopladas\" directamente.

- **Estructura de la Cola**: Utiliza un array de tamaño fijo (QUEUE_LENGTH = 16) con punteros de lectura (tail) y escritura (head).

- **Funciones clave**:

  - put_event_task_system: Inserta un nuevo evento en la cola. Si la cola llega al final del array, vuelve al principio (comportamiento circular).

  - get_event_task_system: Extrae el evento más antiguo de la cola para que el sistema lo procese.

  - any_event_task_system: Devuelve verdadero si hay mensajes pendientes por procesar.

## Resumen del Flujo de Trabajo

1.  **Hardware**: El usuario presiona el botón físico.

2.  **task_sensor_statechart**: Detecta el cambio de nivel eléctrico en el pin, cambia su estado interno y llama a put_event_task_system(EV_SYS_ACTIVE).

3.  **task_system_interface**: Guarda el evento EV_SYS_ACTIVE en la cola circular.

4.  **Tarea de Sistema (implícita)**: En otra parte del código (no adjunta), la tarea de sistema llamará a any_event_task_system() y, al ver que hay algo, usará get_event_task_system() para reaccionar al botón.

### Puntos destacados del diseño:

- **No bloqueante**: No utiliza delay(), lo que permite que el microcontrolador haga múltiples cosas a la vez.

- **Abstracción de Hardware**: La lógica del sensor no sabe qué pin está leyendo; simplemente usa la configuración definida en la lista.

- **Modularidad**: Puedes agregar más botones simplemente añadiendo elementos al array task_sensor_cfg_list en task_sensor.c sin cambiar la lógica de la máquina de estados.

¿Te gustaría que profundizara en alguna función específica o que explique cómo podrías agregar un segundo botón a este esquema?

Para analizar la evolución de estas variables, primero debemos notar que en el archivo task_sensor.c hay un solo sensor configurado en el arreglo task_sensor_cfg_list. Esto significa que SENSOR_DTA_QTY es igual a 1, por lo que la variable index **siempre valdrá 0** durante el procesamiento de los datos del sensor.

A continuación, detallo la evolución lógica de las variables solicitadas desde la inicialización y en las sucesivas ejecuciones de actualización, asumiendo los distintos estados físicos del botón.

### 1. Ejecución de task_sensor_init() {#ejecución-de-task_sensor_init}

Al iniciar la aplicación, esta función establece los valores iniciales para la máquina de estados.

- **index**: Toma el valor 0 para inicializar el único sensor de la lista.

- **task_sensor_dta_list\[0\].tick**: La unidad de medida conceptual es **milisegundos (ms)**, deducido por el comentario (Update by Time Code, period = 1mS) y los valores de las macros (DEL_BTN_MIN, etc.). Sin embargo, es importante destacar que **esta función no inicializa explícitamente el tick**. Al ser un arreglo global, el compilador de C lo inicializa por defecto en 0.

- **task_sensor_dta_list\[0\].state**: Se fuerza explícitamente a ST_BTN_IDLE.

- **task_sensor_dta_list\[0\].event**: Se fuerza explícitamente a EV_BTN_UP.

### 2. Ejecución de task_sensor_update() (Loop Principal) {#ejecución-de-task_sensor_update-loop-principal}

En cada llamada al loop principal (idealmente cada 1 ms), task_sensor_update() itera con index = 0 y llama a task_sensor_statechart(0). La evolución de las variables dependerá de si el botón se presiona o no.

#### Escenario A: El botón NO está presionado (Estado de reposo)

La lectura del pin mediante HAL_GPIO_ReadPin no coincide con el estado presionado (pressed).

- **index**: 0

- **event**: Se asigna a EV_BTN_UP.

- **state**: Al entrar al switch en ST_BTN_IDLE, como el evento es EV_BTN_UP (no EV_BTN_DOWN), no hay transición. Se mantiene en ST_BTN_IDLE.

- **tick**: Se mantiene en 0 (o en su valor previo).

#### Escenario B: El usuario PRESIONA el botón (Flanco de bajada)

La lectura del pin coincide por primera vez con el nivel de pressed.

- **index**: 0

- **event**: Se detecta la pulsación y se actualiza a EV_BTN_DOWN.

- **state**: El switch está en ST_BTN_IDLE. Como event == EV_BTN_DOWN, se cumple la condición del if. Se envía la señal a la cola y el estado **cambia a ST_BTN_ACTIVE**.

- **tick**: Se mantiene inalterado en el código provisto.

#### Escenario C: El usuario MANTIENE presionado el botón

El botón sigue físicamente apretado en las siguientes llamadas al update.

- **index**: 0

- **event**: Se sigue evaluando como EV_BTN_DOWN.

- **state**: El switch entra en ST_BTN_ACTIVE. Como el evento actual *no* es EV_BTN_UP, el if no se cumple. El estado se mantiene en ST_BTN_ACTIVE.

- **tick**: Se mantiene inalterado.

#### Escenario D: El usuario SUELTA el botón (Flanco de subida)

La lectura del pin deja de coincidir con pressed.

- **index**: 0

- **event**: Vuelve a actualizarse a EV_BTN_UP.

- **state**: El switch entra en ST_BTN_ACTIVE. Como el evento es EV_BTN_UP, se cumple la condición, se envía el evento a la cola del sistema y el estado **retorna a ST_BTN_IDLE**.

- **tick**: Se mantiene inalterado.

### Observación Técnica Importante (Falta de implementación de Anti-Rebote)

Notarás que en la explicación de la función task_sensor_update(), la variable **tick no cambia de valor en ningún momento**.

Aunque las estructuras de datos y macros (DEL_BTN_MAX, etc.) sugieren que el autor original planeaba implementar un sistema de **anti-rebote por software** (debouncing), el código fuente provisto en task_sensor_statechart **no incrementa ni decrementa la variable tick** en sus estados principales (ST_BTN_IDLE o ST_BTN_ACTIVE).

El único lugar donde tick se modifica es en la cláusula default del switch:

> C

default:  
p_task_sensor_dta-\>tick = DEL_BTN_MIN;  
p_task_sensor_dta-\>state = ST_BTN_IDLE;  
p_task_sensor_dta-\>event = EV_BTN_UP;  
break;

Este caso default actúa solo como un mecanismo de seguridad o \"salvavidas\" por si la variable de estado se corrompe en memoria y toma un valor distinto a ST_BTN_IDLE (0) o ST_BTN_ACTIVE (1). Fuera de ese caso de error, el código actual reacciona de forma instantánea al cambio del pin físico sin aplicar temporizadores.

Para analizar la evolución de estas variables, primero debemos notar que en el archivo task_sensor.c hay un solo sensor configurado en el arreglo task_sensor_cfg_list. Esto significa que SENSOR_DTA_QTY es igual a 1, por lo que la variable index **siempre valdrá 0** durante el procesamiento de los datos del sensor.

A continuación, detallo la evolución lógica de las variables solicitadas desde la inicialización y en las sucesivas ejecuciones de actualización, asumiendo los distintos estados físicos del botón.

### 1. Ejecución de task_sensor_init() {#ejecución-de-task_sensor_init-1}

Al iniciar la aplicación, esta función establece los valores iniciales para la máquina de estados.

- **index**: Toma el valor 0 para inicializar el único sensor de la lista.

- **task_sensor_dta_list\[0\].tick**: La unidad de medida conceptual es **milisegundos (ms)**, deducido por el comentario (Update by Time Code, period = 1mS) y los valores de las macros (DEL_BTN_MIN, etc.). Sin embargo, es importante destacar que **esta función no inicializa explícitamente el tick**. Al ser un arreglo global, el compilador de C lo inicializa por defecto en 0.

- **task_sensor_dta_list\[0\].state**: Se fuerza explícitamente a ST_BTN_IDLE.

- **task_sensor_dta_list\[0\].event**: Se fuerza explícitamente a EV_BTN_UP.

### 2. Ejecución de task_sensor_update() (Loop Principal) {#ejecución-de-task_sensor_update-loop-principal-1}

En cada llamada al loop principal (idealmente cada 1 ms), task_sensor_update() itera con index = 0 y llama a task_sensor_statechart(0). La evolución de las variables dependerá de si el botón se presiona o no.

#### Escenario A: El botón NO está presionado (Estado de reposo)

La lectura del pin mediante HAL_GPIO_ReadPin no coincide con el estado presionado (pressed).

- **index**: 0

- **event**: Se asigna a EV_BTN_UP.

- **state**: Al entrar al switch en ST_BTN_IDLE, como el evento es EV_BTN_UP (no EV_BTN_DOWN), no hay transición. Se mantiene en ST_BTN_IDLE.

- **tick**: Se mantiene en 0 (o en su valor previo).

#### Escenario B: El usuario PRESIONA el botón (Flanco de bajada)

La lectura del pin coincide por primera vez con el nivel de pressed.

- **index**: 0

- **event**: Se detecta la pulsación y se actualiza a EV_BTN_DOWN.

- **state**: El switch está en ST_BTN_IDLE. Como event == EV_BTN_DOWN, se cumple la condición del if. Se envía la señal a la cola y el estado **cambia a ST_BTN_ACTIVE**.

- **tick**: Se mantiene inalterado en el código provisto.

#### Escenario C: El usuario MANTIENE presionado el botón

El botón sigue físicamente apretado en las siguientes llamadas al update.

- **index**: 0

- **event**: Se sigue evaluando como EV_BTN_DOWN.

- **state**: El switch entra en ST_BTN_ACTIVE. Como el evento actual *no* es EV_BTN_UP, el if no se cumple. El estado se mantiene en ST_BTN_ACTIVE.

- **tick**: Se mantiene inalterado.

#### Escenario D: El usuario SUELTA el botón (Flanco de subida)

La lectura del pin deja de coincidir con pressed.

- **index**: 0

- **event**: Vuelve a actualizarse a EV_BTN_UP.

- **state**: El switch entra en ST_BTN_ACTIVE. Como el evento es EV_BTN_UP, se cumple la condición, se envía el evento a la cola del sistema y el estado **retorna a ST_BTN_IDLE**.

- **tick**: Se mantiene inalterado.

### Observación Técnica Importante (Falta de implementación de Anti-Rebote)

Notarás que en la explicación de la función task_sensor_update(), la variable **tick no cambia de valor en ningún momento**.

Aunque las estructuras de datos y macros (DEL_BTN_MAX, etc.) sugieren que el autor original planeaba implementar un sistema de **anti-rebote por software** (debouncing), el código fuente provisto en task_sensor_statechart **no incrementa ni decrementa la variable tick** en sus estados principales (ST_BTN_IDLE o ST_BTN_ACTIVE).

El único lugar donde tick se modifica es en la cláusula default del switch:

> C

default:  
p_task_sensor_dta-\>tick = DEL_BTN_MIN;  
p_task_sensor_dta-\>state = ST_BTN_IDLE;  
p_task_sensor_dta-\>event = EV_BTN_UP;  
break;

Este caso default actúa solo como un mecanismo de seguridad o \"salvavidas\" por si la variable de estado se corrompe en memoria y toma un valor distinto a ST_BTN_IDLE (0) o ST_BTN_ACTIVE (1). Fuera de ese caso de error, el código actual reacciona de forma instantánea al cambio del pin físico sin aplicar temporizadores.
