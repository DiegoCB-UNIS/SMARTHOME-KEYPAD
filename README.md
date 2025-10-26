# SMARTHOME-KEYPAD
#include "stm32l0xx.h"  // Header para STM32L053R8
#include <string.h>     // Para strcmp y strcpy
#include <math.h>       // Para pow en display

// Defines para pines (sin cambios)
#define SERVO_PIN 0      // PA0 (TIM2_CH1)
#define BUZZER_PIN 1     // PA1 (GPIO_Output)
#define LED_VERDE_PIN 2  // PA2
#define LED_ROJO_PIN 3   // PA3
#define LED_AMARILLO_PIN 4  // PA4
#define LED_AZUL_PIN 5   // PA5

// Pines del keypad (sin cambios)
#define KEYPAD_R0_PIN 8  // PB8
#define KEYPAD_R1_PIN 9  // PB9
#define KEYPAD_R2_PIN 10 // PB10
#define KEYPAD_R3_PIN 11 // PB11
#define KEYPAD_C0_PIN 12 // PB12
#define KEYPAD_C1_PIN 13 // PB13
#define KEYPAD_C2_PIN 14 // PB14
#define KEYPAD_C3_PIN 15 // PB15

// Pines del display 7 segmentos (sin cambios)
#define DISP_A_PIN 0  // PB0
#define DISP_B_PIN 1  // PB1
#define DISP_C_PIN 2  // PB2
#define DISP_D_PIN 3  // PB3
#define DISP_E_PIN 4  // PB4
#define DISP_F_PIN 5  // PB5
#define DISP_G_PIN 6  // PB6
#define DISP_DP_PIN 7 // PB7 (no usado)
#define DISP_D1_PIN 5 // PC5
#define DISP_D2_PIN 6 // PC6
#define DISP_D3_PIN 8 // PC8
#define DISP_D4_PIN 9 // PC9

// Constantes (sin cambios)
#define PASS_LENGTH 4
#define SERVO_OPEN_ANGLE 90
#define SERVO_CLOSE_ANGLE 0
#define BUZZER_DURATION_MS 1000
#define BLINK_DURATION_MS 500

// Variables globales (sin cambios)
volatile char entered_pass[PASS_LENGTH + 1] = {0};
volatile char current_pass[PASS_LENGTH + 1] = "1234";
volatile uint8_t pass_index = 0;
volatile uint8_t error_count = 0;
volatile uint8_t keypad_key = 0;
volatile uint8_t display_digit = 0;
volatile uint32_t timer_ms = 0;
volatile uint8_t state = 0;

// Prototipos (sin cambios)
void GPIO_Init(void);
void Timer_Init(void);
void Keypad_Scan(void);
void Display_Update(void);
void Buzzer_On(void);
void Buzzer_Off(void);
void Servo_SetAngle(uint8_t angle);
void LED_Toggle(uint8_t pin);
void LED_On(uint8_t pin);
void LED_Off(uint8_t pin);

// Tablas (sin cambios)
const uint8_t digit_segments[10] = {
    0b00111111, 0b00000110, 0b01011011, 0b01001111,
    0b01100110, 0b01101101, 0b01111101, 0b00000111,
    0b01111111, 0b01101111
};

const char keypad_map[4][4] = {
    {'1', '2', '3', 'A'},
    {'4', '5', '6', 'B'},
    {'7', '8', '9', 'C'},
    {'*', '0', '#', 'D'}
};

int main(void) {
    GPIO_Init();
    Timer_Init();
    state = 0;
    while (1) {
        if (keypad_key != 255) {
            switch (state) {
                case 0:  // IDLE
                    if (keypad_key >= 0 && keypad_key <= 9) {
                        state = 1;
                        entered_pass[pass_index++] = keypad_map[keypad_key / 4][keypad_key % 4];
                        if (pass_index >= PASS_LENGTH) pass_index = 0;
                    } else if (keypad_key == 10) {  // A
                        if (strcmp((const char *)entered_pass, (const char *)current_pass) == 0) {
                            LED_On(LED_VERDE_PIN);
                            LED_On(LED_AZUL_PIN);
                            Servo_SetAngle(SERVO_OPEN_ANGLE);
                        } else {
                            state = 3;
                            error_count++;
                            LED_On(LED_ROJO_PIN);
                            Buzzer_On();
                        }
                        pass_index = 0;
                        memset((char *)entered_pass, 0, sizeof(entered_pass));
                    } else if (keypad_key == 11) {  // B
                        Servo_SetAngle(SERVO_CLOSE_ANGLE);
                        LED_Off(LED_VERDE_PIN);
                        LED_Off(LED_AZUL_PIN);
                    } else if (keypad_key == 12) {  // C
                        state = 2;
                    } else if (keypad_key == 15) {  // D
                        LED_Toggle(LED_AZUL_PIN);
                    }
                    break;
                case 1:  // ENTER_PASS
                    if (keypad_key >= 0 && keypad_key <= 9) {
                        entered_pass[pass_index++] = keypad_map[keypad_key / 4][keypad_key % 4];
                        if (pass_index >= PASS_LENGTH) pass_index = 0;
                    } else if (keypad_key == 10) {  // A
                        if (strcmp((const char *)entered_pass, (const char *)current_pass) == 0) {
                            LED_On(LED_VERDE_PIN);
                            LED_On(LED_AZUL_PIN);
                            Servo_SetAngle(SERVO_OPEN_ANGLE);
                        } else {
                            state = 3;
                            error_count++;
                            LED_On(LED_ROJO_PIN);
                            Buzzer_On();
                        }
                        state = 0;
                        pass_index = 0;
                        memset((char *)entered_pass, 0, sizeof(entered_pass));
                    }
                    break;
                case 2:  // CHANGE_PASS
                    if (keypad_key >= 0 && keypad_key <= 9) {
                        entered_pass[pass_index++] = keypad_map[keypad_key / 4][keypad_key % 4];
                        if (pass_index >= PASS_LENGTH) pass_index = 0;
                    } else if (keypad_key == 14) {  // #
                        strcpy((char *)current_pass, (const char *)entered_pass);
                        LED_Off(LED_AMARILLO_PIN);
                        state = 0;
                        pass_index = 0;
                        memset((char *)entered_pass, 0, sizeof(entered_pass));
                    }
                    break;
                case 3:  // ERROR
                    if (keypad_key == 13) {  // *
                        LED_Off(LED_ROJO_PIN);
                        Buzzer_Off();
                        state = 0;
                    }
                    break;
            }
            keypad_key = 255;
        }
    }
}

void GPIO_Init(void) {
    RCC->IOPENR |= RCC_IOPENR_GPIOAEN | RCC_IOPENR_GPIOBEN | RCC_IOPENR_GPIOCEN;

    // Servo (PA0): AF para TIM2_CH1
    GPIOA->MODER &= ~GPIO_MODER_MODE0;
    GPIOA->MODER |= GPIO_MODER_MODE0_1;
    GPIOA->AFR[0] |= (1 << GPIO_AFRL_AFSEL0_Pos);

    // Buzzer (PA1): GPIO_Output
    GPIOA->MODER &= ~GPIO_MODER_MODE1;
    GPIOA->MODER |= GPIO_MODER_MODE1_0;
    GPIOA->OTYPER &= ~GPIO_OTYPER_OT_1;

    // LEDs (PA2-PA5): GPIO_Output
    GPIOA->MODER &= ~(GPIO_MODER_MODE2 | GPIO_MODER_MODE3 | GPIO_MODER_MODE4 | GPIO_MODER_MODE5);
    GPIOA->MODER |= (GPIO_MODER_MODE2_0 | GPIO_MODER_MODE3_0 | GPIO_MODER_MODE4_0 | GPIO_MODER_MODE5_0);
    GPIOA->OTYPER &= ~(GPIO_OTYPER_OT_2 | GPIO_OTYPER_OT_3 | GPIO_OTYPER_OT_4 | GPIO_OTYPER_OT_5);
    GPIOA->OSPEEDR |= (GPIO_OSPEEDR_OSPEED2 | GPIO_OSPEEDR_OSPEED3 | GPIO_OSPEEDR_OSPEED4 | GPIO_OSPEEDR_OSPEED5);

    // Keypad filas (PB8-PB11): GPIO_Input con pull-up
    GPIOB->MODER &= ~(GPIO_MODER_MODE8 | GPIO_MODER_MODE9 | GPIO_MODER_MODE10 | GPIO_MODER_MODE11);
    GPIOB->PUPDR |= (GPIO_PUPDR_PUPD8_0 | GPIO_PUPDR_PUPD9_0 | GPIO_PUPDR_PUPD10_0 | GPIO_PUPDR_PUPD11_0);

    // Keypad columnas (PB12-PB15): GPIO_Output
    GPIOB->MODER &= ~(GPIO_MODER_MODE12 | GPIO_MODER_MODE13 | GPIO_MODER_MODE14 | GPIO_MODER_MODE15);
    GPIOB->MODER |= (GPIO_MODER_MODE12_0 | GPIO_MODER_MODE13_0 | GPIO_MODER_MODE14_0 | GPIO_MODER_MODE15_0);
    GPIOB->OTYPER &= ~(GPIO_OTYPER_OT_12 | GPIO_OTYPER_OT_13 | GPIO_OTYPER_OT_14 | GPIO_OTYPER_OT_15);

    // Display segmentos (PB0-PB7): GPIO_Output
    GPIOB->MODER &= ~(GPIO_MODER_MODE0 | GPIO_MODER_MODE1 | GPIO_MODER_MODE2 | GPIO_MODER_MODE3 | GPIO_MODER_MODE4 | GPIO_MODER_MODE5 | GPIO_MODER_MODE6 | GPIO_MODER_MODE7);
    GPIOB->MODER |= (GPIO_MODER_MODE0_0 | GPIO_MODER_MODE1_0 | GPIO_MODER_MODE2_0 | GPIO_MODER_MODE3_0 | GPIO_MODER_MODE4_0 | GPIO_MODER_MODE5_0 | GPIO_MODER_MODE6_0 | GPIO_MODER_MODE7_0);
    GPIOB->OTYPER &= ~(GPIO_OTYPER_OT_0 | GPIO_OTYPER_OT_1 | GPIO_OTYPER_OT_2 | GPIO_OTYPER_OT_3 | GPIO_OTYPER_OT_4 | GPIO_OTYPER_OT_5 | GPIO_OTYPER_OT_6 | GPIO_OTYPER_OT_7);

    // Display dígitos (PC5, PC6, PC8, PC9): GPIO_Output
    GPIOC->MODER &= ~(GPIO_MODER_MODE5 | GPIO_MODER_MODE6 | GPIO_MODER_MODE8 | GPIO_MODER_MODE9);
    GPIOC->MODER |= (GPIO_MODER_MODE5_0 | GPIO_MODER_MODE6_0 | GPIO_MODER_MODE8_0 | GPIO_MODER_MODE9_0);
    GPIOC->OTYPER &= ~(GPIO_OTYPER_OT_5 | GPIO_OTYPER_OT_6 | GPIO_OTYPER_OT_8 | GPIO_OTYPER_OT_9);
}

void Timer_Init(void) {
	RCC->APB1ENR |= RCC_APB1ENR_TIM2EN | RCC_APB1ENR_TIM6EN;
	RCC->APB2ENR |= RCC_APB2ENR_TIM21EN | RCC_APB2ENR_TIM22EN;

    // TIM2: PWM para servo (50Hz, 20ms periodo)
    TIM2->PSC = 159;
    TIM2->ARR = 19999;
    TIM2->CCMR1 |= TIM_CCMR1_OC1M_1 | TIM_CCMR1_OC1M_2;
    TIM2->CCER |= TIM_CCER_CC1E;
    TIM2->CR1 |= TIM_CR1_CEN;

    // TIM3: Escaneo keypad (10ms)
    TIM21->PSC = 1599;
    TIM21->ARR = 159;
    TIM21->DIER |= TIM_DIER_UIE;
    TIM21->CR1 |= TIM_CR1_CEN;
    NVIC_EnableIRQ(TIM21_IRQn);

    // TIM6: Multiplexado display (5ms)
    TIM6->PSC = 1599;
    TIM6->ARR = 79;
    TIM6->DIER |= TIM_DIER_UIE;
    TIM6->CR1 |= TIM_CR1_CEN;
    NVIC_EnableIRQ(TIM6_IRQn);

    // TIM7: Temporización general (1ms)
    TIM22->PSC = 15;
    TIM22->ARR = 999;
    TIM22->DIER |= TIM_DIER_UIE;
    TIM22->CR1 |= TIM_CR1_CEN;
    NVIC_EnableIRQ(TIM22_IRQn);
}

// Handlers de interrupciones
void TIM3_IRQHandler(void) {  // Escaneo keypad (cada 10ms)
    TIM21->SR &= ~TIM_SR_UIF;  // Limpiar flag
    Keypad_Scan();  // Escanear keypad
}

void TIM6_IRQHandler(void) {  // Multiplexado display (cada 5ms)
    TIM6->SR &= ~TIM_SR_UIF;  // Limpiar flag
    Display_Update();  // Actualizar display
}

void TIM7_IRQHandler(void) {  // Temporización general (cada 1ms)
    TIM22->SR &= ~TIM_SR_UIF;  // Limpiar flag
    timer_ms++;  // Incrementar contador de ms

    // Parpadeo LED amarillo en estado CHANGE_PASS (cada 500ms)
    if (state == 2 && (timer_ms % (BLINK_DURATION_MS / 10)) == 0) {  // Cada 50ms para parpadeo rápido
        LED_Toggle(LED_AMARILLO_PIN);
    }

    // Apagar buzzer después de duración en estado ERROR
    if (state == 3 && timer_ms >= BUZZER_DURATION_MS) {
        Buzzer_Off();
        timer_ms = 0;  // Reset para evitar overflow
    }
}

// Funciones auxiliares
void Keypad_Scan(void) {  // Escaneo con lógica invertida: activar columnas, leer filas
    static uint8_t col = 0;  // Columna actual a escanear
    GPIOB->ODR &= ~(1 << (KEYPAD_C0_PIN + col));  // Activar columna actual (low)

    for (uint8_t row = 0; row < 4; row++) {
        if (!(GPIOB->IDR & (1 << (KEYPAD_R0_PIN + row)))) {  // Fila detectada (low)
            keypad_key = row * 4 + col;  // Mapear a índice 0-15
            break;
        } else {
            keypad_key = 255;  // Ninguna tecla presionada
        }
    }

    GPIOB->ODR |= (1 << (KEYPAD_C0_PIN + col));  // Desactivar columna
    col = (col + 1) % 4;  // Siguiente columna
}

void Display_Update(void) {  // Multiplexado para mostrar error_count en 4 dígitos
    // Apagar todos los dígitos (cátodo común: high para apagar)
    GPIOC->ODR |= (1 << DISP_D1_PIN) | (1 << DISP_D2_PIN) | (1 << DISP_D3_PIN) | (1 << DISP_D4_PIN);

    // Calcular valor del dígito actual (de error_count)
    uint8_t digit_value = (error_count / (uint32_t)pow(10, display_digit)) % 10;

    // Encender segmentos para el dígito
    GPIOB->ODR = digit_segments[digit_value];  // Segmentos A-G

    // Encender dígito actual (low para cátodo común)
    switch (display_digit) {
        case 0: GPIOC->ODR &= ~(1 << DISP_D1_PIN); break;
        case 1: GPIOC->ODR &= ~(1 << DISP_D2_PIN); break;
        case 2: GPIOC->ODR &= ~(1 << DISP_D3_PIN); break;
        case 3: GPIOC->ODR &= ~(1 << DISP_D4_PIN); break;
    }

    display_digit = (display_digit + 1) % 4;  // Siguiente dígito
}

void Buzzer_On(void) {  // Encender buzzer (tono simple)
    GPIOA->ODR |= (1 << BUZZER_PIN);
}

void Buzzer_Off(void) {  // Apagar buzzer
    GPIOA->ODR &= ~(1 << BUZZER_PIN);
}

void Servo_SetAngle(uint8_t angle) {  // Ajustar ángulo del servo con PWM
    // Calcular pulse width: 1000us (0°) + (angle * 1000us / 180°)
    uint16_t pulse = 1000 + (angle * 1000 / 180);  // En ticks (basado en TIM2 ARR=19999, PSC=159 para 1us/tick)
    TIM2->CCR1 = pulse;  // Duty cycle para TIM2_CH1
}

void LED_On(uint8_t pin) {  // Encender LED específico
    GPIOA->ODR |= (1 << pin);
}

void LED_Off(uint8_t pin) {  // Apagar LED específico
    GPIOA->ODR &= ~(1 << pin);
}

void LED_Toggle(uint8_t pin) {  // Alternar LED específico
    GPIOA->ODR ^= (1 << pin);
}
