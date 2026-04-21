Estos tres archivos en lenguaje C conforman el diseño de un componente de software basado en una máquina de estados finitos (FSM) no bloqueante. Su objetivo principal es controlar un actuador de hardware, en este caso específico, un LED conectado a los pines GPIO de un microcontrolador (presumiblemente de la familia STM32, dada la nomenclatura de HAL).

A continuación, se detalla el funcionamiento de cada archivo:

### 1. task_actuator_attribute.h (Definiciones y Estructuras) {#task_actuator_attribute.h-definiciones-y-estructuras}

Este archivo de cabecera actúa como el \"contrato\" de datos del módulo. Define los tipos, enumeraciones y estructuras necesarias para modelar el actuador.

- **Eventos (task_actuator_ev_t)**: Define las señales o comandos que pueden alterar el comportamiento del actuador, siendo EV_LED_IDLE (comando para reposo) y EV_LED_ACTIVE (comando para activación).

- **Estados (task_actuator_st_t)**: Define los estados posibles en los que puede encontrarse el actuador, que son ST_LED_IDLE (apagado/reposo) y ST_LED_ACTIVE (encendido/activo).

- **Identificadores (task_actuator_id_t)**: Enumera los actuadores disponibles en el sistema, en este caso, solo existe ID_LED_A.

- **Estructura de configuración (task_actuator_cfg_t)**: Almacena de forma constante la información del hardware, como el puerto GPIO, el pin de conexión y los estados lógicos (GPIO_PinState) necesarios para encender o apagar el LED.

- **Estructura de datos en tiempo de ejecución (task_actuator_dta_t)**: Contiene las variables dinámicas, incluyendo el estado actual de la FSM, el último evento recibido y una bandera booleana (flag) que señaliza si hay un evento pendiente por procesar.

### 2. task_actuator.c (Implementación de la Máquina de Estados) {#task_actuator.c-implementación-de-la-máquina-de-estados}

Este archivo contiene la lógica operativa y el control de transiciones de la máquina de estados.

- **Datos y Configuración**: Se instancian las listas estáticas de configuración (task_actuator_cfg_list) y las variables en memoria RAM para los datos dinámicos (task_actuator_dta_list).

- **Inicialización (task_actuator_init)**: Se encarga de arrancar el sistema iterando sobre todos los actuadores. Establece el estado inicial en ST_LED_IDLE, el evento en EV_LED_IDLE y la bandera flag en falso. También garantiza que el hardware arranque apagado escribiendo el pin GPIO con el estado led_off.

- **Actualización periódica (task_actuator_update)**: Es una tarea de actualización (pensada para ejecutarse en un bucle temporal, por ejemplo, cada 1 ms) que recorre los actuadores y ejecuta el motor de la máquina de estados para cada uno.

- **Transiciones lógicas (task_actuator_statechart)**: Es el núcleo de evaluación. Utiliza una estructura switch-case basada en el estado actual:

  - Si está en ST_LED_IDLE y detecta un evento EV_LED_ACTIVE (confirmado por flag == true), enciende el LED físicamente, cambia el estado interno a ST_LED_ACTIVE y limpia la bandera (flag = false).

  - Si está en ST_LED_ACTIVE y detecta un evento EV_LED_IDLE (con flag == true), apaga el LED físicamente, cambia el estado a ST_LED_IDLE y limpia la bandera (flag = false).

  - Posee un bloque default para recuperación de errores, devolviendo la máquina a un estado seguro (ST_LED_IDLE) si el estado se corrompe.

### 3. task_actuator_interface.c (Interfaz de Comunicación Externa) {#task_actuator_interface.c-interfaz-de-comunicación-externa}

Este archivo desacopla la máquina de estados del resto del programa, brindando una forma segura para interactuar con ella.

- **Inyección de eventos (put_event_task_actuator)**: Es la API que otras tareas (por ejemplo, botones, sensores o comunicaciones) deben llamar para ordenar acciones al actuador.

- Esta función recibe el evento deseado y el identificador del actuador destino.

- Dentro de la función, se carga el nuevo event en la estructura correspondiente y se levanta la bandera de procesamiento asignando true a la variable flag. Esta acción es la que desencadenará el cambio de estado la próxima vez que el bucle principal llame a task_actuator_update().

Para comprender la evolución de las variables a lo largo de la ejecución del programa, es necesario dividir el análisis en tres fases: la inicialización, la ejecución repetitiva sin alteraciones externas y la ejecución cuando se inyecta un evento a través de la interfaz.

Dado que la lista de actuadores task_actuator_cfg_list contiene un único elemento (ID_LED_A), el valor máximo de cantidad de actuadores (ACTUATOR_DTA_QTY) es 1.

A continuación, se detalla la evolución de las variables para ese elemento en la posición index = 0:

### 1. Inicialización (task_actuator_init()) {#inicialización-task_actuator_init}

Al arrancar el sistema y llamar a la función de inicialización, las variables se establecen en sus estados seguros o de reposo.

- **index**: Toma el valor 0 para iterar sobre el primer y único actuador de la lista.

- **task_actuator_dta_list\[0\].tick**: Aunque la estructura se declara a nivel global (lo que en C inicializa sus valores numéricos en 0 por defecto), la función task_actuator_init no inicializa explícitamente esta variable. **Unidad de medida**: Según los comentarios del código, el período de actualización es de **milisegundos (ms)** (period = 1mS).

- **task_actuator_dta_list\[0\].state**: Se inicializa en ST_LED_IDLE (estado de reposo).

- **task_actuator_dta_list\[0\].event**: Se inicializa en EV_LED_IDLE.

- **task_actuator_dta_list\[0\].flag**: Se inicializa en false.

### 2. Sucesivas ejecuciones (task_actuator_update()) SIN eventos externos {#sucesivas-ejecuciones-task_actuator_update-sin-eventos-externos}

Si el bucle principal llama repetidamente a task_actuator_update() y ninguna otra parte del programa invoca la interfaz externa para enviar un evento, el estado interno se mantiene inmutable.

- **index**: Vuelve a tomar el valor 0 en el bucle for.

- **task_actuator_dta_list\[0\].state**: Permanece en ST_LED_IDLE.

- **task_actuator_dta_list\[0\].event**: Sigue siendo EV_LED_IDLE.

- **task_actuator_dta_list\[0\].flag**: Se mantiene en false. Al entrar al case ST_LED_IDLE, la condición ((true == flag) && (EV_LED_ACTIVE == event)) no se cumple, por lo que el switch finaliza sin hacer cambios.

- **task_actuator_dta_list\[0\].tick**: Permanece en 0. En el código de la máquina de estados provisto, la variable tick **no se incrementa ni se actualiza** en los estados normales, solo se fuerza a DEL_LED_MIN (que vale 0ul) si la máquina cayera en el caso default por algún error.

### 3. Ejecución de task_actuator_update() CON inyección de eventos {#ejecución-de-task_actuator_update-con-inyección-de-eventos}

Para que las variables evolucionen de su estado inercial, otra tarea de la aplicación debe inyectar un evento usando la interfaz put_event_task_actuator(evento, ID_LED_A).

#### A. Evolución al Activar el Actuador: {#a.-evolución-al-activar-el-actuador}

1.  **Inyección**: Se llama a put_event_task_actuator(EV_LED_ACTIVE, ID_LED_A).

    - event pasa a ser EV_LED_ACTIVE.

    - flag pasa a ser true.

2.  **Siguiente update()**:

    - index = 0.

    - La máquina de estados evalúa el case ST_LED_IDLE.

    - La condición ((true == flag) && (EV_LED_ACTIVE == event)) ahora se cumple.

    - flag pasa a ser false (consumiendo el evento).

    - Se enciende el LED a nivel de hardware y **state** pasa a ser ST_LED_ACTIVE.

    - event conserva el valor EV_LED_ACTIVE (el código solo limpia la bandera flag, no el registro del último evento).

#### B. Evolución al Desactivar el Actuador: {#b.-evolución-al-desactivar-el-actuador}

1.  **Inyección**: Estando en el estado activo, se llama a put_event_task_actuator(EV_LED_IDLE, ID_LED_A).

    - event pasa a ser EV_LED_IDLE.

    - flag pasa a ser true.

2.  **Siguiente update()**:

    - index = 0.

    - La máquina evalúa el case ST_LED_ACTIVE.

    - La condición ((true == flag) && (EV_LED_IDLE == event)) ahora se cumple.

    - flag pasa a ser false.

    - Se apaga el LED físicamente y **state** retorna a ser ST_LED_IDLE.

La función void task_actuator_statechart(uint32_t index) contiene el núcleo lógico de la máquina de estados finitos (FSM) encargada de controlar el actuador. Su comportamiento consiste en evaluar el estado actual de un actuador específico y decidir si debe realizar una transición de estado o ejecutar una acción física en el hardware, basándose en los eventos pendientes.

A continuación, se detalla su comportamiento paso a paso:

### 1. Asignación de Punteros {#asignación-de-punteros}

Al recibir el parámetro index (que identifica sobre qué actuador se va a operar), la función primero obtiene los punteros a las estructuras de configuración (task_actuator_cfg_t) y de datos dinámicos (task_actuator_dta_t) correspondientes a ese índice. Esto le permite acceder a qué pin físico debe modificar y cuál es su estado actual.

### 2. Evaluación de Estados (switch-case) {#evaluación-de-estados-switch-case}

Luego, ingresa a un bloque switch que evalúa la variable p_task_actuator_dta-\>state. Dependiendo de este estado, la FSM se comporta de la siguiente manera:

- **Caso ST_LED_IDLE (Estado de Reposo/Apagado)\*\*\*\*:**

  - Verifica si hay un evento pendiente evaluando si la bandera es verdadera (true == p_task_actuator_dta-\>flag) **y** si dicho evento es el comando de activación (EV_LED_ACTIVE == p_task_actuator_dta-\>event).

  - Si se cumplen ambas condiciones:

    1.  Limpia la bandera (flag = false) para indicar que el evento fue consumido.

    2.  Enciende el LED interactuando con el hardware mediante la función HAL_GPIO_WritePin, utilizando los datos del puerto, el pin y el nivel lógico de encendido (led_on) almacenados en la configuración.

    3.  Transiciona al nuevo estado asignando ST_LED_ACTIVE a la variable state.

  - Si no hay evento o es un evento distinto, no hace nada y permanece en reposo.

- **Caso ST_LED_ACTIVE (Estado Activo/Encendido)\*\*\*\*:**

  - Verifica si hay un evento pendiente (true == p_task_actuator_dta-\>flag) **y** si el evento es el comando de reposo (EV_LED_IDLE == p_task_actuator_dta-\>event).

  - Si se cumplen ambas condiciones:

    1.  Limpia la bandera (flag = false).

    2.  Apaga el LED mediante HAL_GPIO_WritePin, utilizando el nivel lógico de apagado (led_off).

    3.  Transiciona de regreso al estado inicial asignando ST_LED_IDLE a la variable state.

  - Si no recibe la orden de apagado, el LED continúa encendido.

- **Caso default (Mecanismo de Seguridad/Recuperación)\*\*\*\*:**

  - Si por algún motivo de corrupción de memoria o error la variable state toma un valor no definido en la enumeración, el programa cae en este bloque.

  - Su comportamiento es forzar a la máquina a un estado seguro reinicializando las variables: establece tick en DEL_LED_MIN, fuerza el estado a ST_LED_IDLE, el evento a EV_LED_IDLE y apaga la bandera (flag = false).

Es importante hacer una pequeña aclaración técnica antes de comenzar: la variable identifier **no existe** dentro de las funciones task_actuator_init() ni task_actuator_update(). En estas dos funciones, el recorrido de los arreglos se realiza mediante la variable local index. La variable identifier (del tipo enumerado task_actuator_id_t) es en realidad el parámetro de entrada de la función externa put_event_task_actuator().

Dado que en este sistema ambos valores (index e identifier) representan lo mismo aritméticamente (apuntan a la posición 0 correspondiente a ID_LED_A), a continuación te detallo cómo evolucionan las variables solicitadas a lo largo del ciclo de vida del programa:

### 1. Inicialización (task_actuator_init()) {#inicialización-task_actuator_init-1}

Durante el arranque, el sistema configura las variables a su estado inicial seguro para cada actuador (en este caso, operando sobre index = 0).

- **identifier / index**: Toma el valor 0 en el bucle for.

- **task_actuator_dta_list\[0\].event**: Se inicializa con el valor EV_LED_IDLE.

- **task_actuator_dta_list\[0\].flag**: Se inicializa explícitamente en false.

### 2. Ejecuciones de task_actuator_update() (Sin interacciones externas) {#ejecuciones-de-task_actuator_update-sin-interacciones-externas}

Mientras la aplicación corre su bucle principal y ninguna otra tarea requiere accionar el LED, el comportamiento es estático.

- **identifier / index**: Recorre periódicamente el valor 0.

- **task_actuator_dta_list\[0\].event**: Permanece inalterado en EV_LED_IDLE.

- **task_actuator_dta_list\[0\].flag**: Permanece en false. Al estar en false, la máquina de estados evaluada en task_actuator_statechart ignora cualquier intento de transición lógica.

### 3. Inyección de un evento mediante put_event_task_actuator() {#inyección-de-un-evento-mediante-put_event_task_actuator}

Aquí es donde interviene propiamente la variable identifier. Si el sistema desea encender el LED, se llama a esta función con los parámetros (EV_LED_ACTIVE, ID_LED_A).

- **identifier**: Ingresa a la función con el valor ID_LED_A (que equivale a 0).

- **task_actuator_dta_list\[identifier\].event**: Se sobrescribe con el nuevo evento solicitado, en este ejemplo, EV_LED_ACTIVE.

- **task_actuator_dta_list\[identifier\].flag**: Se levanta la bandera asignándole el valor true para notificar a la máquina de estados que hay una novedad.

### 4. Siguiente ejecución de task_actuator_update() (Procesando el evento) {#siguiente-ejecución-de-task_actuator_update-procesando-el-evento}

En la iteración inmediatamente posterior a la inyección del evento, la máquina de estados detecta la bandera y actúa en consecuencia dentro de task_actuator_statechart().

- **identifier / index**: Vuelve a iterar sobre el valor 0.

- **task_actuator_dta_list\[0\].flag**: Al cumplirse la condición del estado (true == flag y evento coincidente), la primera acción de la máquina es \"consumir\" el evento devolviendo esta variable a **false**.

- **task_actuator_dta_list\[0\].event**: **Conserva su valor** (EV_LED_ACTIVE). El código de la máquina de estados no reinicia el evento a EV_LED_IDLE, simplemente baja la bandera flag para indicar que el evento actual ya fue atendido.

A partir de ese punto, en las sucesivas ejecuciones de task_actuator_update(), la variable flag seguirá siendo false y event mantendrá el valor del último evento recibido hasta que put_event_task_actuator() vuelva a ser invocada.
