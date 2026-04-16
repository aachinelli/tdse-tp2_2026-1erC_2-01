Aquí tienes el análisis y la explicación del funcionamiento del código fuente proporcionado, así como la evolución de las configuraciones de reloj desde el arranque hasta el bucle principal.

### Análisis del código fuente

**1. startup_stm32f103rbtx.s (Archivo de arranque / Ensamblador)**

Este es el primer código que se ejecuta en el microcontrolador cuando se le suministra energía o se reinicia. Sus tareas principales son preparar el entorno para que el código en C pueda funcionar:

- **Tabla de vectores (g_pfnVectors):** Crea un índice en memoria que mapea cada evento del sistema (resets, errores críticos y periféricos) con la dirección de la función que debe atenderlo (Rutina de Servicio de Interrupción o ISR).

- **Reset_Handler:** Es el punto de entrada principal. Primero, invoca a SystemInit para inicializar el sistema de relojes al estado básico de hardware. Luego, inicializa la memoria RAM copiando los valores predeterminados de las variables (desde la Flash a la sección .data en la RAM) y llenando con ceros las variables no inicializadas (sección .bss).

- Finalmente, ejecuta constructores estáticos (\_\_libc_init_array) y da el salto (bl main) a la función main() escrita en C.

**2. main.c (Programa principal)**

Es el núcleo de la aplicación de usuario. Establece la configuración de los periféricos y mantiene el ciclo de vida del programa:

- **HAL_Init():** Se ejecuta primero para reiniciar periféricos, inicializar el temporizador del sistema (SysTick) y configurar las prioridades base de interrupción.

- **SystemClock_Config():** Define a qué velocidad correrá el procesador (\"SystemCoreClock\"). Activa el Oscilador Interno de Alta Velocidad (HSI) y usa un lazo de seguimiento de fase (PLL) para multiplicar esta frecuencia.

- **Inicialización de periféricos:** Llama a rutinas generadas como MX_GPIO_Init() para configurar pines de entrada/salida (como el botón B1_Pin y el LED LD2_Pin) y MX_USART2_UART_Init() para habilitar comunicación serial UART a 115200 baudios.

- **Bucle principal while(1):** Llama a app_init() por única vez, y luego entra en un bucle infinito donde se ejecuta repetidamente app_update(), delegando la lógica del usuario a los archivos de la aplicación (app.c/app.h).

**3. stm32f1xx_it.c (Manejadores de Interrupciones)**

Este archivo contiene la implementación de las funciones que interrumpen el flujo normal del procesador cuando ocurre un evento.

- Posee manejadores de excepciones críticos como HardFault_Handler que atrapan errores de memoria (dejando al sistema en un bucle infinito por seguridad).

- **SysTick_Handler():** Función que es invocada periódicamente por hardware. Llama internamente a HAL_IncTick() para sumar +1 a un contador global de milisegundos, el cual es vital para calcular tiempos y usar funciones como retardos (HAL_Delay()).

- **EXTI15_10_IRQHandler():** Maneja las interrupciones asíncronas de los pines externos configurados (en este caso delegando la atención del botón B1_Pin a la librería HAL).

### Evolución de SysTick y SystemCoreClock

Al encender el microcontrolador, la configuración del tiempo y la velocidad del núcleo (SystemCoreClock) evoluciona a través de las siguientes fases:

**Fase 1: Ejecución del Reset_Handler**

- **Contexto:** El microcontrolador inicia usando el oscilador interno (HSI) por defecto sin multiplicadores. En la familia STM32F1, el HSI es de 8 MHz.

- **SystemCoreClock:** 8 MHz (Velocidad base de hardware).

- **SysTick:** Apagado (El temporizador aún no ha sido configurado).

**Fase 2: Entrada a main() e invocación de HAL_Init()**

- **Contexto:** La capa HAL configura inicialmente el SysTick basándose en el reloj actual.

- **SystemCoreClock:** Se mantiene en 8 MHz.

- **SysTick:** Comienza a funcionar. Se calibra para interrumpir al procesador cada 1 milisegundo (frecuencia de interrupción de **1 kHz**), basándose en la velocidad temporal de 8 MHz.

**Fase 3: Invocación de SystemClock_Config() en main.c**

- **Contexto:** El código reconfigura agresivamente los relojes para ganar más velocidad.

  - Toma la fuente HSI y la divide por 2 (RCC_PLLSOURCE_HSI_DIV2), lo que resulta en 4 MHz.

  - Utiliza el PLL para multiplicar esos 4 MHz por 16 (RCC_PLL_MUL16), resultando en una frecuencia de salida de 64 MHz.

  - Asigna esta nueva velocidad generada (PLLCLK) como fuente principal del sistema usando RCC_SYSCLKSOURCE_PLLCLK sin división general (RCC_SYSCLK_DIV1).

- **SystemCoreClock:** Salta bruscamente de 8 MHz a **64 MHz**.

- **SysTick:** Como la velocidad del núcleo aumentó 8 veces, la función de configuración (HAL_RCC_ClockConfig) recalibra automáticamente los registros del SysTick para compensar este cambio de velocidad. Sigue operando a **1 kHz** (1 ms).

**Fase 4: Ejecución del bucle principal while(1) en main.c**

- **Contexto:** El sistema de hardware está completamente configurado y en velocidad de crucero procesando la lógica de usuario.

- **SystemCoreClock:** Queda fijo a **64 MHz**.

- **SysTick:** Continúa generando interrupciones constantes y estables de **1 kHz** invocando al SysTick_Handler().
