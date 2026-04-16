¡Claro que sí! Pasar de un diagrama de estados (Máquina de Estados Finitos o FSM) a código C es una de las tareas más comunes en sistemas embebidos y desarrollo de lógica de control.

La forma más profesional y escalable de hacerlo es usando la estructura **Switch-Case** dentro de un bucle principal.

## Estructura Recomendada en C

Para que tu trabajo práctico esté bien organizado, te sugiero seguir este patrón de cuatro pasos:

1.  **Enumeraciones (enum):** Para definir los nombres de los estados de forma legible.

2.  **Variable de Estado:** Una variable global o estática que guarde el \"Estado Actual\".

3.  **Lógica de Transición (switch):** Evalúa qué evento ocurrió y a qué estado saltar.

4.  **Lógica de Salida:** Qué acciones realiza el sistema en cada estado.

### 

### 

### 

### 

### 

### 

### 

### 

### 

### 

### 

### 

### 

### 

### 

### 

### 

### Ejemplo: Control de un Semáforo Simple

Supongamos que tienes tres estados: VERDE, AMARILLO y ROJO.

C

\#**include** \<stdio.h\>  
  
// 1. Definición de estados  
typedef enum {  
ESTADO_VERDE,  
ESTADO_AMARILLO,  
ESTADO_ROJO  
} Estado_t;  
  
int main() {  
Estado_t estadoActual = ESTADO_VERDE; // Estado inicial  
int sensorPeaton = 0; // Variable de entrada (evento)  
  
while(1) {  
// 3. Lógica de la Máquina de Estados  
switch (estadoActual) {  
case ESTADO_VERDE:  
printf(\"Luz VERDE - Avance\n\");  
// Transición: Si hay un peatón, cambia a amarillo  
if (sensorPeaton) {  
estadoActual = ESTADO_AMARILLO;  
}  
break;  
  
case ESTADO_AMARILLO:  
printf(\"Luz AMARILLA - Precaución\n\");  
// Transición automática a rojo  
estadoActual = ESTADO_ROJO;  
break;  
  
case ESTADO_ROJO:  
printf(\"Luz ROJA - Pare\n\");  
// Transición: Volver a verde tras un tiempo (simulado)  
estadoActual = ESTADO_VERDE;  
sensorPeaton = 0; // Reset del evento  
break;  
}  
  
// Simulación de espera y entrada de usuario  
printf(\"¿Hay peaton esperando? (1=Si / 0=No): \");  
scanf(\"%d\", &sensorPeaton);  
}  
return 0;  
}

## Consejos para tu TP

- **Evita usar if-else anidados:** El switch-case es mucho más fácil de leer y depurar cuando el diagrama tiene más de 3 estados.

- **Separación de Acciones:** En diagramas complejos (tipo Mealy o Moore), es útil separar la **lógica de transición** (cambiar el estado) de la **lógica de acción** (encender un LED, escribir en consola).

- **Diagrama Visual:** Si tu TP requiere entregar el gráfico, asegúrate de que cada flecha (transición) tenga su condición escrita claramente.

Para poder ayudarte mejor con tu código específico:

Para avanzar con tu trabajo práctico:

Explicar la diferencia entre máquinas de Moore y Mealy

Estructurar un sistema con punteros a funciones
