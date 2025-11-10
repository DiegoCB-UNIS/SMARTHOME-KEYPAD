# 🏠 SMARTHOME KEYPAD con funcionalida extra y funcion de interrupciones

El cosdigo es el mismo solo que ahora se añadio una función extra y ahora este codigo opera ahora bajo interrupciones. La funcionalidad extra seria que antes el display mostraba un conteo de cuantas veces se ingresaba la contraseña equivocada. Esa función aun lo sigue teniendo pero ahora aparte de mostrar ese conteo eh incluso despues, ahora se muestra la contrasela que se ingreso. Recordando que la contraseña es de cuatro digitos. Tomando en cuenta que el display de 7 segementos tambien es de cuatro digitos y por lo tanto solo cuatro digitos como maximo mostrara. En el momento que se ingrese la contraseña al presionar los botones del keypad, por ejemplo: si la contraseña actual es "1234", entoces se preciona los botons con los respectivos numeros. Y al hacerlo se mostrara el "1234" en el display de esta manera: 1.2.3.4. 
Y uno vez que se presiona el boton [A] que es el que se encarga de ingresar esa contraseña ingresada al sistema, automaticamente se borrara los numeros mostrados en el display para volver a estar en su funcion principal de contar las veces que se ingreso la contraseña incorrecta. Ahora el codigo vendria siendo el siguiente con sus respectivas modificaciones para llevar a cabo las funcionalidades nuevas:





/* USER CODE BEGIN Header */
/**
  ******************************************************************************
  * @file           : main.c
  * @brief          : Smart Home Keypad with STM32L053R8
  *                  - Keypad 4x4: Rows PB0-PB3 (EXTI), Columns PB4-PB7
  *                  - Display 7-seg 4 digits: Segments PA6-PA13, Digits PB8-PB11
  *                  - LEDs: Green PA2, Red PA3, Yellow PA4, Blue PA5
  *                  - Buzzer: PA1
  *                  - Servo: PA0 (TIM2_CH1 PWM)
  *                  - Timers: TIM2 (PWM servo), TIM6 (keypad scan), TIM21 (display multiplex)
  ******************************************************************************
  */
/* USER CODE END Header */

/* Includes ------------------------------------------------------------------*/
#include "main.h"
#include <string.h>
#include <stdbool.h>
#include <stdint.h>

/* Private typedef -----------------------------------------------------------*/

/* Private define ------------------------------------------------------------*/

/* Private macro -------------------------------------------------------------*/

/* Private variables ---------------------------------------------------------*/
TIM_HandleTypeDef htim2;

/* USER CODE BEGIN PV */
TIM_HandleTypeDef htim6;   // Escaneo del keypad
TIM_HandleTypeDef htim21;  // Display multiplexacion

#define SERVO_PIN GPIO_PIN_0  // PA0 for TIM2_CH1 PWM
#define BUZZER_PIN GPIO_PIN_1  // PA1
#define LED_GREEN_PIN GPIO_PIN_2  // PA2
#define LED_RED_PIN GPIO_PIN_3  // PA3
#define LED_YELLOW_PIN GPIO_PIN_4  // PA4
#define LED_BLUE_PIN GPIO_PIN_5  // PA5

// Keypad filas (EXTI inputs)
#define ROW1_PIN GPIO_PIN_0  // PB0
#define ROW2_PIN GPIO_PIN_1  // PB1
#define ROW3_PIN GPIO_PIN_2  // PB2
#define ROW4_PIN GPIO_PIN_3  // PB3

// Keypad columnas (outputs)
#define COL1_PIN GPIO_PIN_4  // PB4
#define COL2_PIN GPIO_PIN_5  // PB5
#define COL3_PIN GPIO_PIN_6  // PB6
#define COL4_PIN GPIO_PIN_7  // PB7

// Display segmentos (PA6-PA13)
#define SEG_A_PIN GPIO_PIN_6
#define SEG_B_PIN GPIO_PIN_7
#define SEG_C_PIN GPIO_PIN_8
#define SEG_D_PIN GPIO_PIN_9
#define SEG_E_PIN GPIO_PIN_10
#define SEG_F_PIN GPIO_PIN_11
#define SEG_G_PIN GPIO_PIN_12
#define SEG_DP_PIN GPIO_PIN_13

// Display digitos (PB8-PB11)
#define DIG1_PIN GPIO_PIN_8   // PB8
#define DIG2_PIN GPIO_PIN_9   // PB9
#define DIG3_PIN GPIO_PIN_10  // PB10
#define DIG4_PIN GPIO_PIN_11  // PB11

volatile uint8_t current_col = 0;  // Activacion actual de columnas (0-3)
volatile uint32_t last_key_time = 0;
volatile uint8_t active_col = 0;

char stored_password[5] = "1234";  // Contrseña predeterminada
char entry_buffer[5] = {0};  // Input usuario
uint8_t entry_len = 0;  // Tamaño del input
bool change_mode = false;  // Modo de cambio de constraseña

volatile uint32_t wrong_count = 0;  //Contador de contrseñas incorrectas

//Display buffer
uint8_t disp_digits[4] = {0, 0, 0, 0};

uint8_t show_entered_code = 0;   // 1 cuando se muestran los dígitos ingresados
uint16_t entered_number = 0;     // Guarda el número que el usuario está digitando

char input_code[5] = {0};  // Guarda los 4 dígitos ingresados + terminador '\0'
uint8_t code_index = 0;    // Apunta al dígito actual (0–4)

//Patrones de segmentos para0-9
const uint8_t seg_patterns[10] = {
  0b00111111, // 0
  0b00000110, // 1
  0b01011011, // 2
  0b01001111, // 3
  0b01100110, // 4
  0b01101101, // 5
  0b01111101, // 6
  0b00000111, // 7
  0b01111111, // 8
  0b01101111  // 9
};


void set_column(uint8_t col);
char map_key(uint8_t row, uint8_t col);
void process_key(char key);
void servo_set_angle(uint8_t angle);
void update_display(void);
void set_segments(uint8_t digit);
void buzzer_on(void);
void buzzer_off(void);
/* USER CODE END PV */

void SystemClock_Config(void);
static void MX_GPIO_Init(void);
static void MX_TIM2_Init(void);
/* USER CODE BEGIN PFP */
void display_show_number(uint16_t number);
static void MX_TIM21_Init(void);
/* USER CODE END PFP */


/* Private user code ---------------------------------------------------------*/

/**
  * @brief  The application entry point.
  * @retval int
  */
int main(void)
{
  HAL_Init();
  __HAL_RCC_SYSCFG_CLK_ENABLE();

  SystemClock_Config();

  MX_GPIO_Init();
  MX_TIM2_Init();
  MX_TIM21_Init();

  HAL_TIM_PWM_Start(&htim2, TIM_CHANNEL_1);  // Servo PWM
  HAL_TIM_Base_Start_IT(&htim21);  // Display multiplexacion

  //Inicia el servo
  servo_set_angle(0);

  //Inicia los diplays
  update_display();

  set_column(0);

  while (1)
  {
    __WFI();
  }
}

/**
  * @brief System Clock Configuration (MSI 2.097MHz)
  * @retval None
  */
void SystemClock_Config(void)
{
  RCC_OscInitTypeDef RCC_OscInitStruct = {0};
  RCC_ClkInitTypeDef RCC_ClkInitStruct = {0};

  __HAL_PWR_VOLTAGESCALING_CONFIG(PWR_REGULATOR_VOLTAGE_SCALE1);

  RCC_OscInitStruct.OscillatorType = RCC_OSCILLATORTYPE_MSI;
  RCC_OscInitStruct.MSIState = RCC_MSI_ON;
  RCC_OscInitStruct.MSICalibrationValue = 0;
  RCC_OscInitStruct.MSIClockRange = RCC_MSIRANGE_5;
  RCC_OscInitStruct.PLL.PLLState = RCC_PLL_NONE;
  HAL_RCC_OscConfig(&RCC_OscInitStruct);

  RCC_ClkInitStruct.ClockType = RCC_CLOCKTYPE_HCLK | RCC_CLOCKTYPE_SYSCLK
                                | RCC_CLOCKTYPE_PCLK1 | RCC_CLOCKTYPE_PCLK2;
  RCC_ClkInitStruct.SYSCLKSource = RCC_SYSCLKSOURCE_MSI;
  RCC_ClkInitStruct.AHBCLKDivider = RCC_SYSCLK_DIV1;
  RCC_ClkInitStruct.APB1CLKDivider = RCC_HCLK_DIV1;
  RCC_ClkInitStruct.APB2CLKDivider = RCC_HCLK_DIV1;
  HAL_RCC_ClockConfig(&RCC_ClkInitStruct, FLASH_LATENCY_0);
}

/**
  * @brief TIM2 Initialization (PWM for servo, 50Hz)
  * @param None
  * @retval None
  */
static void MX_TIM2_Init(void)
{
  __HAL_RCC_TIM2_CLK_ENABLE();
  htim2.Instance = TIM2;
  htim2.Init.Prescaler = 79;  //26.2125kHz
  htim2.Init.CounterMode = TIM_COUNTERMODE_UP;
  htim2.Init.Period = 524;  //50Hz
  htim2.Init.ClockDivision = TIM_CLOCKDIVISION_DIV1;
  HAL_TIM_PWM_Init(&htim2);

  TIM_OC_InitTypeDef sConfigOC = {0};
  sConfigOC.OCMode = TIM_OCMODE_PWM1;
  sConfigOC.Pulse = 26;
  sConfigOC.OCPolarity = TIM_OCPOLARITY_HIGH;
  HAL_TIM_PWM_ConfigChannel(&htim2, &sConfigOC, TIM_CHANNEL_1);

  //GPIO para el PA0
  GPIO_InitTypeDef GPIO_InitStruct = {0};
  __HAL_RCC_GPIOA_CLK_ENABLE();
  GPIO_InitStruct.Pin = SERVO_PIN;
  GPIO_InitStruct.Mode = GPIO_MODE_AF_PP;
  GPIO_InitStruct.Pull = GPIO_NOPULL;
  GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_LOW;
  GPIO_InitStruct.Alternate = GPIO_AF2_TIM2;  // AF para el TIM2_CH1
  HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);
}

/**
  * @brief TIM21 Initialization (Display multiplexing, 1ms tick)
  * @param None
  * @retval None
  */
static void MX_TIM21_Init(void)
{
  __HAL_RCC_TIM21_CLK_ENABLE();
  htim21.Instance = TIM21;
  htim21.Init.Prescaler = 2096;
  htim21.Init.CounterMode = TIM_COUNTERMODE_UP;
  htim21.Init.Period = 0;
  htim21.Init.ClockDivision = TIM_CLOCKDIVISION_DIV1;
  HAL_TIM_Base_Init(&htim21);
  HAL_NVIC_SetPriority(TIM21_IRQn, 4, 0);
  HAL_NVIC_EnableIRQ(TIM21_IRQn);
}

/**
  * @brief GPIO Initialization
  * @param None
  * @retval None
  */
static void MX_GPIO_Init(void)
{
  GPIO_InitTypeDef GPIO_InitStruct = {0};
  __HAL_RCC_GPIOA_CLK_ENABLE();
  __HAL_RCC_GPIOB_CLK_ENABLE();

  // Buzzer de PA1
  GPIO_InitStruct.Pin = BUZZER_PIN;
  GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_PP;
  GPIO_InitStruct.Pull = GPIO_NOPULL;
  GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_LOW;
  HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);
  buzzer_off();

  //LEDs PA2-PA5
  GPIO_InitStruct.Pin = LED_GREEN_PIN | LED_RED_PIN | LED_YELLOW_PIN | LED_BLUE_PIN;
  GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_PP;
  HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);
  HAL_GPIO_WritePin(GPIOA, LED_GREEN_PIN | LED_RED_PIN | LED_YELLOW_PIN | LED_BLUE_PIN, GPIO_PIN_RESET);

  //Keypad columnas PB4-PB7
  GPIO_InitStruct.Pin = COL1_PIN | COL2_PIN | COL3_PIN | COL4_PIN;
  GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_PP;
  GPIO_InitStruct.Pull = GPIO_NOPULL;
  HAL_GPIO_Init(GPIOB, &GPIO_InitStruct);
  HAL_GPIO_WritePin(GPIOB, COL1_PIN | COL2_PIN | COL3_PIN | COL4_PIN, GPIO_PIN_SET);  // Inactive HIGH

  //Keypad filas PB0-PB3
  GPIO_InitStruct.Pin = ROW1_PIN | ROW2_PIN | ROW3_PIN | ROW4_PIN;
  GPIO_InitStruct.Mode = GPIO_MODE_IT_RISING;
  GPIO_InitStruct.Pull = GPIO_PULLDOWN;
  HAL_GPIO_Init(GPIOB, &GPIO_InitStruct);

  //Display segmentos PA6-PA13
  GPIO_InitStruct.Pin = SEG_A_PIN | SEG_B_PIN | SEG_C_PIN | SEG_D_PIN
                      | SEG_E_PIN | SEG_F_PIN | SEG_G_PIN | SEG_DP_PIN;
  GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_PP;
  GPIO_InitStruct.Pull = GPIO_NOPULL;
  HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);
  HAL_GPIO_WritePin(GPIOA, SEG_A_PIN | SEG_B_PIN | SEG_C_PIN | SEG_D_PIN
                      | SEG_E_PIN | SEG_F_PIN | SEG_G_PIN | SEG_DP_PIN, GPIO_PIN_RESET);

  //Display digitos PB8-PB11
  GPIO_InitStruct.Pin = DIG1_PIN | DIG2_PIN | DIG3_PIN | DIG4_PIN;
  GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_PP;
  GPIO_InitStruct.Pull = GPIO_NOPULL;
  HAL_GPIO_Init(GPIOB, &GPIO_InitStruct);
  HAL_GPIO_WritePin(GPIOB, DIG1_PIN | DIG2_PIN | DIG3_PIN | DIG4_PIN, GPIO_PIN_RESET);

  // EXTI
  HAL_NVIC_SetPriority(EXTI0_1_IRQn, 2, 0);
  HAL_NVIC_EnableIRQ(EXTI0_1_IRQn);
  HAL_NVIC_SetPriority(EXTI2_3_IRQn, 2, 0);
  HAL_NVIC_EnableIRQ(EXTI2_3_IRQn);
}

//Set active column for keypad scanning
void set_column(uint8_t col)
{
  HAL_GPIO_WritePin(GPIOB, COL1_PIN | COL2_PIN | COL3_PIN | COL4_PIN, GPIO_PIN_SET);
  if (col <= 3) {
    uint16_t pins[4] = {COL1_PIN, COL2_PIN, COL3_PIN, COL4_PIN};
    HAL_GPIO_WritePin(GPIOB, pins[col], GPIO_PIN_RESET);
    current_col = col;
  } else {
    current_col = 255;
  }
}

char map_key(uint8_t row, uint8_t col)
{
  const char keys[4][4] = {
    {'1','2','3','A'},
    {'4','5','6','B'},
    {'7','8','9','C'},
    {'*','0','#','D'}
  };
  if (row >= 1 && row <= 4 && col >= 1 && col <= 4) return keys[row-1][col-1];
  return 0;
}

void process_key(char key)
{
  if (key >= '0' && key <= '9') {
	  input_code[code_index++] = key;

	      // Activamos el modo de mostrar la contraseña ingresada
	      show_entered_code = 1;

	      // Convertimos el código ingresado hasta ahora a número
	      entered_number = 0;
	      for (int i = 0; i < code_index; i++) {
	          entered_number = entered_number * 10 + (input_code[i] - '0');
	      }

	      // Mostramos el número en el display
	      display_show_number(entered_number);
    if (entry_len < 4) {
      entry_buffer[entry_len++] = key;
      entry_buffer[entry_len] = '\0';
    }
    return;
  }

  switch (key) {
    case 'A':  // Enter contraseña
      if (entry_len == 4) {
        if (strcmp(entry_buffer, stored_password) == 0) {
          HAL_GPIO_WritePin(GPIOA, LED_GREEN_PIN | LED_BLUE_PIN, GPIO_PIN_SET);
          servo_set_angle(90);
        } else {
          HAL_GPIO_WritePin(GPIOA, LED_RED_PIN, GPIO_PIN_SET);
          buzzer_on();
          wrong_count++;
          update_display();
        }
        entry_len = 0;
        memset(entry_buffer, 0, sizeof(entry_buffer));
      }
      break;

    case 'B':  // Cerrar puerta
      servo_set_angle(0);
      HAL_GPIO_WritePin(GPIOA, LED_GREEN_PIN, GPIO_PIN_RESET);
      break;

    case 'C':  // Entrar en modo cambio de contraseña
      change_mode = true;
      entry_len = 0;
      memset(entry_buffer, 0, sizeof(entry_buffer));
      break;

    case '#':  // Confirma contraseña
      if (change_mode && entry_len == 4) {
        strcpy(stored_password, entry_buffer);
        change_mode = false;
        entry_len = 0;
        memset(entry_buffer, 0, sizeof(entry_buffer));
      }
      break;

    case 'D':  //Encender LED azul
      HAL_GPIO_WritePin(GPIOA, LED_BLUE_PIN, GPIO_PIN_SET);
      break;

    case '*':  //Apagar todos los componentes
      HAL_GPIO_WritePin(GPIOA, LED_GREEN_PIN | LED_RED_PIN | LED_YELLOW_PIN | LED_BLUE_PIN, GPIO_PIN_RESET);
      buzzer_off();
      break;

    default:
      break;
  }
}

void servo_set_angle(uint8_t angle)
{
  if (angle > 180) angle = 180;
  uint16_t pulse = 25 + (angle * 100 / 180);  //(180°)
  __HAL_TIM_SET_COMPARE(&htim2, TIM_CHANNEL_1, pulse);
}

void update_display(void)
{
  uint32_t val = wrong_count > 9999 ? 9999 : wrong_count;
  disp_digits[3] = val % 10;
  val /= 10;
  disp_digits[2] = val % 10;
  val /= 10;
  disp_digits[1] = val % 10;
  val /= 10;
  disp_digits[0] = val % 10;
}

void display_show_number(uint16_t number)
{
    // Descompone el número en 4 dígitos individuales
    disp_digits[0] = (number / 1000) % 10;
    disp_digits[1] = (number / 100) % 10;
    disp_digits[2] = (number / 10) % 10;
    disp_digits[3] = number % 10;
}

void set_segments(uint8_t digit)
{
  uint8_t pattern = seg_patterns[digit];
  HAL_GPIO_WritePin(GPIOA, SEG_A_PIN, (pattern & 0x01) ? GPIO_PIN_SET : GPIO_PIN_RESET);
  HAL_GPIO_WritePin(GPIOA, SEG_B_PIN, (pattern & 0x02) ? GPIO_PIN_SET : GPIO_PIN_RESET);
  HAL_GPIO_WritePin(GPIOA, SEG_C_PIN, (pattern & 0x03) ? GPIO_PIN_SET : GPIO_PIN_RESET);
  HAL_GPIO_WritePin(GPIOA, SEG_D_PIN, (pattern & 0x04) ? GPIO_PIN_SET : GPIO_PIN_RESET);
  HAL_GPIO_WritePin(GPIOA, SEG_E_PIN, (pattern & 0x05) ? GPIO_PIN_SET : GPIO_PIN_RESET);
  HAL_GPIO_WritePin(GPIOA, SEG_F_PIN, (pattern & 0x06) ? GPIO_PIN_SET : GPIO_PIN_RESET);
  HAL_GPIO_WritePin(GPIOA, SEG_G_PIN, (pattern & 0x07) ? GPIO_PIN_SET : GPIO_PIN_RESET);
  HAL_GPIO_WritePin(GPIOA, SEG_DP_PIN, GPIO_PIN_RESET);  // Decimal apagado
}

//Control del Buzzer
void buzzer_on(void) { HAL_GPIO_WritePin(GPIOA, BUZZER_PIN, GPIO_PIN_SET); }
void buzzer_off(void) { HAL_GPIO_WritePin(GPIOA, BUZZER_PIN, GPIO_PIN_RESET); }

//Escaner del TIM6 del Keypad columnas
void TIM6_IRQHandler(void)
{
  if (__HAL_TIM_GET_FLAG(&htim6, TIM_FLAG_UPDATE) != RESET) {
    __HAL_TIM_CLEAR_IT(&htim6, TIM_IT_UPDATE);
    static uint8_t col = 0;
    set_column(col);
    col = (col + 1) % 4;  // Ciclo  0-3
  }
}

//Multiplexacion del display en TIM21
void TIM21_IRQHandler(void)
{
    if (__HAL_TIM_GET_FLAG(&htim21, TIM_FLAG_UPDATE) != RESET)
    {
        __HAL_TIM_CLEAR_IT(&htim21, TIM_IT_UPDATE);
        static uint8_t digit_idx = 0;
        uint16_t dig_pins[4] = {DIG1_PIN, DIG2_PIN, DIG3_PIN, DIG4_PIN};
        //Apagar TODOS los dígitos
        HAL_GPIO_WritePin(GPIOB, DIG1_PIN | DIG2_PIN | DIG3_PIN | DIG4_PIN, GPIO_PIN_RESET);
        //Apagar TODOS los segmentos
        HAL_GPIO_WritePin(GPIOA,
            SEG_A_PIN | SEG_B_PIN | SEG_C_PIN | SEG_D_PIN | SEG_E_PIN | SEG_F_PIN | SEG_G_PIN | SEG_DP_PIN, GPIO_PIN_RESET);
        //Cargar los segmentos del dígito actual
        set_segments(disp_digits[digit_idx]);
        // Encender el dígito actual
        HAL_GPIO_WritePin(GPIOB, dig_pins[digit_idx], GPIO_PIN_SET);
        //Pasar al próximo dígito
        digit_idx = (digit_idx + 1) % 4;
    }
}


//EXTI para las filas del display:  PB0-PB1
void EXTI0_1_IRQHandler(void)
{
  if (__HAL_GPIO_EXTI_GET_IT(ROW1_PIN) != RESET) {
    __HAL_GPIO_EXTI_CLEAR_IT(ROW1_PIN);
    HAL_GPIO_EXTI_Callback(ROW1_PIN);
  }
  if (__HAL_GPIO_EXTI_GET_IT(ROW2_PIN) != RESET) {
    __HAL_GPIO_EXTI_CLEAR_IT(ROW2_PIN);
    HAL_GPIO_EXTI_Callback(ROW2_PIN);
  }
}

//EXTI para las filas PB2-PB3
void EXTI2_3_IRQHandler(void)
{
  if (__HAL_GPIO_EXTI_GET_IT(ROW3_PIN) != RESET) {
    __HAL_GPIO_EXTI_CLEAR_IT(ROW3_PIN);
    HAL_GPIO_EXTI_Callback(ROW3_PIN);
  }
  if (__HAL_GPIO_EXTI_GET_IT(ROW4_PIN) != RESET) {
    __HAL_GPIO_EXTI_CLEAR_IT(ROW4_PIN);
    HAL_GPIO_EXTI_Callback(ROW4_PIN);
  }
}

//EXTI llamado de vuelta del keypad
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
{
  uint32_t now = HAL_GetTick();
  if (now - last_key_time < 200) return;  // Debounce
  last_key_time = now;

  uint8_t row = 0;
  if (GPIO_Pin == ROW1_PIN) row = 1;
  else if (GPIO_Pin == ROW2_PIN) row = 2;
  else if (GPIO_Pin == ROW3_PIN) row = 3;
  else if (GPIO_Pin == ROW4_PIN) row = 4;

  // Detectar la columna activa actual
  uint8_t col = active_col;

  // Obtener la tecla presionada
  char key = map_key(row, col + 1);
  if (key) process_key(key);

  // Cambiar a la siguiente columna (rotación rápida)
  active_col = (active_col + 1) % 4;
  set_column(active_col);
}

/* USER CODE END 4 */

/**
  * @brief  This function is executed in case of error occurrence.
  * @retval None
  */
void Error_Handler(void)
{
  /* USER CODE BEGIN Error_Handler_Debug */
  /* User can add his own implementation to report the HAL error return state */
  __disable_irq();
  while (1)
  {
  }
  /* USER CODE END Error_Handler_Debug */
}

#ifdef USE_FULL_ASSERT
/**
  * @brief  Reports the name of the source file and the source line number
  *         where the assert_param error has occurred.
  * @param  file: pointer to the source file name
  * @param  line: assert_param error line source number
  * @retval None
  */
void assert_failed(uint8_t *file, uint32_t line)
{
  /* USER CODE BEGIN 6 */
  /* User can add his own implementation to report the file name and line number,
     ex: printf("Wrong parameters value: file %s on line %d\r\n", file, line) */
  /* USER CODE END 6 */
}
#endif /* USE_FULL_ASSERT */





## Nota importante:
El problema siguie siendo el mismo del anterior, cuando le doy a ejecutar la aplicación, sale que el target no fue detectado o encontrado. Estuve investigando las posibles causas para ver a que se debe eso. Estuve primero revisando las configuraciones de los pines cada uno. Tambien el codigo y los posibles errores que podria tener. Pero por el momento todo esta correcto. Eh realizado un promt de prueba para ver si efectivamente no hay una comunicación entre la placa y el dispositovo PC. No hubo ningun error pero sin embargo aun sigue dando problema de comunicacion entre el target. Al principio tambien podria ser que algunos de los componentes o incluso el USB no funcionen bien. Eh visto que ese problema es muy comun al operar con un STM32. Pero a medida que avanzaba no encontrara una solución al aplicar acciones opcionables para que el codigo se ejecute en la maqueta. Ya que el codigo funciona correctamente pero por alguna razon no se esta comunicando con el nucleo. 
