# SMARTHOME-KEYPAD
#include <stdint.h>

// Definiciones de registros para STM32L053R8 (usando CMSIS sin HAL)
#define RCC_BASE        0x40021000
#define RCC_AHBENR      (*(volatile uint32_t*)(RCC_BASE + 0x14))
#define RCC_APB1ENR     (*(volatile uint32_t*)(RCC_BASE + 0x1C))
#define RCC_APB2ENR     (*(volatile uint32_t*)(RCC_BASE + 0x18))

#define GPIOA_BASE      0x48000000
#define GPIOB_BASE      0x48000400
#define GPIOA_MODER     (*(volatile uint32_t*)(GPIOA_BASE + 0x00))
#define GPIOA_OTYPER    (*(volatile uint32_t*)(GPIOA_BASE + 0x04))
#define GPIOA_OSPEEDR   (*(volatile uint32_t*)(GPIOA_BASE + 0x08))
#define GPIOA_PUPDR     (*(volatile uint32_t*)(GPIOA_BASE + 0x0C))
#define GPIOA_IDR       (*(volatile uint32_t*)(GPIOA_BASE + 0x10))
#define GPIOA_ODR       (*(volatile uint32_t*)(GPIOA_BASE + 0x14))
#define GPIOA_BSRR      (*(volatile uint32_t*)(GPIOA_BASE + 0x18))
#define GPIOB_MODER     (*(volatile uint32_t*)(GPIOB_BASE + 0x00))
#define GPIOB_OTYPER    (*(volatile uint32_t*)(GPIOB_BASE + 0x04))
#define GPIOB_OSPEEDR   (*(volatile uint32_t*)(GPIOB_BASE + 0x08))
#define GPIOB_PUPDR     (*(volatile uint32_t*)(GPIOB_BASE + 0x0C))
#define GPIOB_IDR       (*(volatile uint32_t*)(GPIOB_BASE + 0x10))
#define GPIOB_ODR       (*(volatile uint32_t*)(GPIOB_BASE + 0x14))
#define GPIOB_BSRR      (*(volatile uint32_t*)(GPIOB_BASE + 0x18))

#define TIM2_BASE       0x40000000
#define TIM2_CR1        (*(volatile uint32_t*)(TIM2_BASE + 0x00))
#define TIM2_DIER       (*(volatile uint32_t*)(TIM2_BASE + 0x0C))
#define TIM2_SR         (*(volatile uint32_t*)(TIM2_BASE + 0x10))
#define TIM2_EGR        (*(volatile uint32_t*)(TIM2_BASE + 0x14))
#define TIM2_CCMR1      (*(volatile uint32_t*)(TIM2_BASE + 0x18))
#define TIM2_CCER       (*(volatile uint32_t*)(TIM2_BASE + 0x20))
#define TIM2_CNT        (*(volatile uint32_t*)(TIM2_BASE + 0x24))
#define TIM2_PSC        (*(volatile uint32_t*)(TIM2_BASE + 0x28))
#define TIM2_ARR        (*(volatile uint32_t*)(TIM2_BASE + 0x2C))
#define TIM2_CCR1       (*(volatile uint32_t*)(TIM2_BASE + 0x34))
#define TIM2_CCR2       (*(volatile uint32_t*)(TIM2_BASE + 0x38))

#define TIM21_BASE      0x40010800
#define TIM21_CR1       (*(volatile uint32_t*)(TIM21_BASE + 0x00))
#define TIM21_DIER      (*(volatile uint32_t*)(TIM21_BASE + 0x0C))
#define TIM21_SR        (*(volatile uint32_t*)(TIM21_BASE + 0x10))
#define TIM21_EGR       (*(volatile uint32_t*)(TIM21_BASE + 0x14))
#define TIM21_CCMR1     (*(volatile uint32_t*)(TIM21_BASE + 0x18))
#define TIM21_CCER      (*(volatile uint32_t*)(TIM21_BASE + 0x20))
#define TIM21_CNT       (*(volatile uint32_t*)(TIM21_BASE + 0x24))
#define TIM21_PSC       (*(volatile uint32_t*)(TIM21_BASE + 0x28))
#define TIM21_ARR       (*(volatile uint32_t*)(TIM21_BASE + 0x2C))
#define TIM21_CCR1      (*(volatile uint32_t*)(TIM21_BASE + 0x34))

#define USART2_BASE     0x40004400
#define USART2_CR1      (*(volatile uint32_t*)(USART2_BASE + 0x00))
#define USART2_BRR      (*(volatile uint32_t*)(USART2_BASE + 0x0C))
#define USART2_ISR      (*(volatile uint32_t*)(USART2_BASE + 0x1C))
#define USART2_TDR      (*(volatile uint32_t*)(USART2_BASE + 0x28))

#define NVIC_ISER       (*(volatile uint32_t*)0xE000E100)

// Pines asignados (ajustables; asumiendo GPIOA/GPIOB en NUCLEO-L053R8)
// LEDs: PA0 (Rojo), PA1 (Verde), PA2 (Amarillo), PA3 (Azul)
// Servo PWM: PA4 (TIM2_CH1)
// Buzzer: PA5 (PWM TIM2_CH2)
// Display 7seg: PB0-PB11 (A-F, D1-D4, etc.; multiplexado)
// Keypad: PB12-PB15 (filas), PA6-PA9 (columnas) - 8 pines total
#define LED_RED_PIN     0
#define LED_GREEN_PIN   1
#define LED_YELLOW_PIN  2
#define LED_BLUE_PIN    3
#define SERVO_PIN       4
#define BUZZER_PIN      5
#define KEYPAD_ROW1     12  // PB12
#define KEYPAD_ROW2     13  // PB13
#define KEYPAD_ROW3     14  // PB14
#define KEYPAD_ROW4     15  // PB15
#define KEYPAD_COL1     6   // PA6
#define KEYPAD_COL2     7   // PA7
#define KEYPAD_COL3     8   // PA8
#define KEYPAD_COL4     9   // PA9
// Display pines: PB0 (A), PB1 (B), PB2 (C), PB3 (D), PB4 (E), PB5 (F), PB6 (G), PB7 (DP), PB8 (D1), PB9 (D2), PB10 (D3), PB11 (D4)

// Variables globales
volatile uint8_t password[5] = "1234";  // Contraseña actual (4 dígitos + null)
volatile uint8_t input_buffer[5] = {0}; // Buffer para ingreso
volatile uint8_t buffer_index = 0;
volatile uint8_t error_count = 0;      // Contador de errores
volatile uint8_t door_open = 0;        // Estado puerta (0=cerrada, 1=abierta)
volatile uint8_t change_mode = 0;      // Modo cambio contraseña
volatile uint8_t blue_led_state = 0;   // Estado LED azul
volatile uint8_t display_digit = 0;    // Dígito actual del display (0-3)

// Rutinas optimizadas
void gpio_init(void) {
    RCC_AHBENR |= (1 << 17) | (1 << 18);  // Habilitar GPIOA y GPIOB
    // LEDs como salida
    GPIOA_MODER |= (1 << (LED_RED_PIN * 2)) | (1 << (LED_GREEN_PIN * 2)) | (1 << (LED_YELLOW_PIN * 2)) | (1 << (LED_BLUE_PIN * 2));
    GPIOA_OTYPER &= ~((1 << LED_RED_PIN) | (1 << LED_GREEN_PIN) | (1 << LED_YELLOW_PIN) | (1 << LED_BLUE_PIN));
    GPIOA_OSPEEDR |= (3 << (LED_RED_PIN * 2)) | (3 << (LED_GREEN_PIN * 2)) | (3 << (LED_YELLOW_PIN * 2)) | (3 << (LED_BLUE_PIN * 2));
    // Servo y Buzzer como salida (AF para PWM)
    GPIOA_MODER |= (2 << (SERVO_PIN * 2)) | (2 << (BUZZER_PIN * 2));  // AF mode
    GPIOA_OSPEEDR |= (3 << (SERVO_PIN * 2)) | (3 << (BUZZER_PIN * 2));
    // Keypad filas como salida, columnas como entrada con pull-up
    GPIOB_MODER |= (1 << (KEYPAD_ROW1 * 2)) | (1 << (KEYPAD_ROW2 * 2)) | (1 << (KEYPAD_ROW3 * 2)) | (1 << (KEYPAD_ROW4 * 2));
    GPIOB_OTYPER &= ~((1 << KEYPAD_ROW1) | (1 << KEYPAD_ROW2) | (1 << KEYPAD_ROW3) | (1 << KEYPAD_ROW4));
    GPIOA_MODER &= ~((3 << (KEYPAD_COL1 * 2)) | (3 << (KEYPAD_COL2 * 2)) | (3 << (KEYPAD_COL3 * 2)) | (3 << (KEYPAD_COL4 * 2)));
    GPIOA_PUPDR |= (1 << (KEYPAD_COL1 * 2)) | (1 << (KEYPAD_COL2 * 2)) | (1 << (KEYPAD_COL3 * 2)) | (1 << (KEYPAD_COL4 * 2));
    // Display como salida
    GPIOB_MODER |= 0x555555;  // PB0-PB11 como salida (01 en cada par de bits)
    GPIOB_OTYPER &= 0x0FFF;
    GPIOB_OSPEEDR |= 0x555555;
}

void timer_init(void) {
    RCC_APB1ENR |= (1 << 0);  // TIM2
    RCC_APB2ENR |= (1 << 2);  // TIM21
    // TIM2 para servo (PWM 50Hz) y buzzer
    TIM2_PSC = 799;  // Prescaler para 50Hz (8MHz / 800 = 10kHz, /20000 = 50Hz)
    TIM2_ARR = 19999;  // Periodo 20ms
    TIM2_CCMR1 = 0x60 | (0x60 << 8);  // PWM mode 1 para CH1 y CH2
    TIM2_CCER |= (1 << 0) | (1 << 4);  // Enable CH1 y CH2
    TIM2_CCR1 = 1500;  // Servo neutral (1.5ms)
    TIM2_CCR2 = 0;     // Buzzer off
    TIM2_CR1 |= (1 << 0);  // Enable TIM2
    // TIM21 para escaneo keypad y refresco display (1ms interrupt)
    TIM21_PSC = 799;
    TIM21_ARR = 9;  // 10kHz / 10 = 1kHz (1ms)
    TIM21_DIER |= (1 << 0);  // Update interrupt
    TIM21_CR1 |= (1 << 0);
    NVIC_ISER |= (1 << 20);  // Enable TIM21 interrupt (IRQ 20)
}

void usart_init(void) {
    RCC_APB1ENR |= (1 << 17);  // USART2
    USART2_BRR = 0x341;  // 9600 baud (8MHz)
    USART2_CR1 = (1 << 3) | (1 << 13);  // TE, UE
}

void clock_init(void) {
    // Usar MSI 8MHz por defecto (NUCLEO-L053R8)
}

void usart_send(char *str) {
    while (*str) {
        while (!(USART2_ISR & (1 << 7)));  // Wait TXE
        USART2_TDR = *str++;
    }
}

void set_led(uint8_t pin, uint8_t state) {
    if (state) GPIOA_BSRR = (1 << pin);
    else GPIOA_BSRR = (1 << (pin + 16));
}

void set_servo(uint16_t pulse) {  // Pulse en us (1000-2000 para 0-180°)
    TIM2_CCR1 = (pulse * 19999UL) / 20000;  // Cálculo correcto: proporcional a ARR
}

void buzzer_sound(uint8_t on) {
    if (on) TIM2_CCR2 = 1000;  // PWM para tono (ajusta para frecuencia deseada)
    else TIM2_CCR2 = 0;
}

void display_number(uint16_t num) {
    uint8_t digits[4] = {num / 1000, (num / 100) % 10, (num / 10) % 10, num % 10};
    uint8_t segments[10] = {0x3F, 0x06, 0x5B, 0x4F, 0x66, 0x6D, 0x7D, 0x07, 0x7F, 0x6F};  // 0-9
    GPIOB_ODR &= ~(0x7F << 0);  // Clear A-G
    GPIOB_ODR |= segments[digits[display_digit]] << 0;
    GPIOB_ODR &= ~(0xF << 8);  // Clear D1-D4
    GPIOB_ODR |= (1 << (8 + display_digit));
    display_digit = (display_digit + 1) % 4;
}

uint8_t scan_keypad(void) {
    uint8_t key = 0;
    for (uint8_t row = 0; row < 4; row++) {
        GPIOB_ODR &= ~(0xF << KEYPAD_ROW1);  // Clear filas
        GPIOB_ODR |= (1 << (KEYPAD_ROW1 + row));  // Set fila
        for (volatile uint16_t i = 0; i < 1000; i++);  // Simple debounce delay (no delayms)
        for (uint8_t col = 0; col < 4; col++) {
            if (!(GPIOA_IDR & (1 << (KEYPAD_COL1 + col)))) {
                key = row * 4 + col + 1;  // 1-16
                while (!(GPIOA_IDR & (1 << (KEYPAD_COL1 + col))));  // Wait release
                return key;
            }
        }
    }
    return 0;
}

void process_key(uint8_t key) {
    if (key >= 1 && key <= 10) {  // 1-9,0
        if (buffer_index < 4) input_buffer[buffer_index++] = '0' + (key == 10 ? 0 : key);
    } else if (key == 11) {  // A: Confirmar contraseña
        input_buffer[buffer_index] = '\0';
        if (change_mode) {
            for (uint8_t i = 0; i < 4; i++) password[i] = input_buffer[i];
            change_mode = 0;
            set_led(LED_YELLOW_PIN, 0);
            usart_send("Password changed\n");
        } else {
            uint8_t match = 1;
            for (uint8_t i = 0; i < 4; i++) if (password[i] != input_buffer[i]) match = 0;
            if (match) {
                set_led(LED_GREEN_PIN, 1);
                set_led(LED_BLUE_PIN, 1);
                set_servo(2000);  // Abrir puerta (derecha)
                door_open = 1;
                usart_send("Door opened\n");
            } else {
                buzzer_sound(1);
                set_led(LED_RED_PIN, 1);
                error_count++;
                usart_send("Wrong password\n");
            }
        }
        buffer_index = 0;
    } else if (key == 12) {  // B: Cerrar puerta
        if (door_open) {
            set_servo(1000);  // Cerrar (izquierda)
            door_open = 0;
            usart_send("Door closed\n");
        }
    } else if (key == 13) {  // C: Cambiar contraseña
        change_mode = 1;
        set_led(LED_YELLOW_PIN, 1);
        buffer_index = 0;
        usart_send("Change password mode\n");
    } else if (key == 14) {  // D: Toggle LED azul
        blue_led_state = !blue_led_state;
        set_led(LED_BLUE_PIN, blue_led_state);
        usart_send("Blue LED toggled\n");
    } else if (key == 15) {  // *: Reset error
        buzzer_sound(0);
        set_led(LED_RED_PIN, 0);
        set_led(LED_GREEN_PIN, 0);
        usart_send("Error reset\n");
    } else if (key == 16) {  // #: Confirmar cambio (ya en A)
        // Nada extra
    }
}
void TIM21_IRQHandler(void) {
    TIM21_SR &= ~(1 << 0);  // Clear update interrupt flag
    // Escanear keypad periódicamente (1ms)
    uint8_t key = scan_keypad();
    if (key) process_key(key);
    // Refrescar display multiplexado
    display_number(error_count);
}
int main(void) {
    clock_init();   // Inicializar clock (MSI 8MHz por defecto)
    gpio_init();    // Configurar GPIOs
    timer_init();   // Configurar TIM2 (PWM) y TIM21 (interrupción)
    usart_init();   // Configurar USART2 para debug opcional
    // No hay interrupciones EXTI configuradas, ya que usamos polling en TIM21
    while (1) {
        // Loop vacío: Todo se maneja en interrupciones para procesos paralelos eficientes
        // No usar delays aquí; el sistema responde en tiempo real
    }
}
