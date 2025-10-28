# 🏠 SMARTHOME KEYPAD

## 📖 Introducción
Este proyecto consiste en la elaboración de una maqueta de un sistema de teclado electrónico para el hogar, cuya función principal es controlar una cerradura electrónica que mantiene la vivienda cerrada y segura. Además, incorpora otras funcionalidades complementarias que enriquecen la simulación.
La lógica interna del sistema está desarrollada en HAL para STM32, utilizando un núcleo STM32 como elemento esencial para el funcionamiento del keypad. A lo largo de este documento se presentan los componentes empleados, así como las funcionalidades implementadas en la maqueta.

---

## 🔧 Componentes y materiales
- Keypad 4x4  
  ![Keypad](data:image/png;base64,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)

- Display de 7 segmentos de 4 dígitos cátodo común  
  ![Display](imagenes/display.png)

- LEDs (Verde, Amarillo, Azul y Rojo)  
  ![LEDs](imagenes/leds.png)

- Micro servo MG90S engranaje metálico 180°  
  ![Servo](imagenes/servo.png)

- Buzzer  
  ![Buzzer](imagenes/buzzer.png)

- Fuente de alimentación para protoboard de 3.3V y 5V  
  ![Fuente](imagenes/fuente.png)

- Jumpers  
  ![Jumpers](imagenes/jumpers.png)

- Cables UTP  
  ![Cables](imagenes/cables.png)

- Cartón  
  ![Carton](imagenes/carton.png)

- Madera  
  ![Madera](imagenes/madera.png)

- NUCLEO-L053R8  
  ![Nucleo](imagenes/nucleo.png)

---

## ⚙️ Funcionalidades del Keypad
1. **Contraseña predeterminada:** `1234`.  
   - Se ingresa en el keypad y se confirma con el botón `[A]`.

2. **Botón [B]:** Cierra el motor servo (mecanismo de la puerta).

3. **Botón [C]:** Cambia la contraseña.  
   - Se enciende/parpadea un LED amarillo.  
   - Se ingresa la nueva contraseña y se confirma con el botón `[#]`.

4. **Botón [D]:** Enciende y apaga un LED azul.

5. **Validación de contraseña:**  
   - Correcta → LED verde + LED azul encendidos + movimiento del servo.  
   - Incorrecta → LED rojo encendido + buzzer activado (alarma).

6. **Botón [*]:** Apaga todos los componentes (modo reposo).

---

## 🏗️ Proceso de elaboración de la maqueta
1. Corte de cartón resistente para las dimensiones de la casa.  
   ![Corte](imagenes/corte.png)

2. Uso de cubos de madera como soporte para paredes, techo y suelo.

3. Construcción del techo triangular con madera de coco.  
   - El techo no se unió directamente para permitir accesibilidad al interior.

4. Creación del circuito eléctrico.

5. Desarrollo del código (ver rama `Codigo-v4`).

6. Integración del circuito en la maqueta, mostrando LEDs y displays en la superficie.

---

## ✅ Resultado final
![Resultado](imagenes/resultado.png)

---

## 💻 Código
El código final se encuentra en la rama: **`Codigo-v4`**

---

## 🔌 Configuración de pines del Nucleo
| Pin  | Función        |
|------|----------------|
| PA0  | TIM2_CH1       |
| PA1  | GPIO_Output    |
| PA2  | GPIO_Output    |
| PA3  | GPIO_Output    |
| PA4  | GPIO_Output    |
| PA5  | GPIO_Output    |
| PB0  | GPIO_Output    |
| PB1  | GPIO_Output    |
| PB2  | GPIO_Output    |
| PB3  | GPIO_Output    |
| PB4  | GPIO_Output    |
| PB5  | GPIO_Output    |
| PB6  | GPIO_Output    |
| PB7  | GPIO_Output    |
| PB8  | GPIO_Input     |
| PB9  | GPIO_Input     |
| PB10 | GPIO_Input     |
| PB11 | GPIO_Input     |
| PB12 | GPIO_Output    |
| PB13 | GPIO_Output    |
| PB14 | GPIO_Output    |
| PB15 | GPIO_Output    |
| PC5  | GPIO_Output    |
| PC8  | GPIO_Output    |
| PC9  | GPIO_Output    |

---
