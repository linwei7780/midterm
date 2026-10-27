/* USER CODE BEGIN Header */
/**
  ******************************************************************************
  * @file           : main.c
  * @brief          : midterm project - 生理監控與多重感官警報系統 (STM32G474RE)
  ******************************************************************************
  */
/* USER CODE END Header */
/* Includes ------------------------------------------------------------------*/
#include "main.h"

/* Private includes ----------------------------------------------------------*/
/* USER CODE BEGIN Includes */

/* USER CODE END Includes */

/* Private typedef -----------------------------------------------------------*/
/* USER CODE BEGIN PTD */
// 生理監控狀態機定義
typedef enum {
  STATE_NORMAL = 0, // 正常狀態：平緩呼吸燈
  STATE_STRESS,     // 心率偏高/情緒起伏
  STATE_ALARM       // 異常警報：急速閃爍警報
} SystemState_t;
/* USER CODE END PTD */

/* Private define ------------------------------------------------------------*/
/* USER CODE BEGIN PD */
#define BUTTON_GPIO_Port GPIOC
#define BUTTON_Pin GPIO_PIN_10          // 外接按鈕 PC10

#define RED_LED_GPIO_Port GPIOA
#define RED_LED_Pin GPIO_PIN_5          // 紅燈 PA5

#define GREEN_LED_GPIO_Port GPIOA
#define GREEN_LED_Pin GPIO_PIN_7        // 綠燈 PA7
/* USER CODE END PD */

/* Private macro -------------------------------------------------------------*/
/* USER CODE BEGIN PM */

/* USER CODE END PM */

/* Private variables ---------------------------------------------------------*/
TIM_HandleTypeDef htim1;
TIM_HandleTypeDef htim3;

/* USER CODE BEGIN PV */
SystemState_t current_state = STATE_NORMAL;

// 時間戳記與非阻塞計時變數
uint32_t last_breath_tick = 0;
uint32_t last_btn_tick = 0;
uint32_t last_press_time = 0;
uint32_t last_alarm_tick = 0;

// 互動狀態旗標
uint8_t button_ever_pressed = 0; // 記錄是否曾按過按鈕，避免開機直接觸發逾時警報

// 呼吸燈與 PWM 變數 (藍燈 PA8 透過 TIM1_CH1 輸出)
int32_t step_val = 0;
int32_t direction = 1;
uint32_t ccr_val = 0;

// 心率計算變數
uint32_t press_interval = 0;
uint32_t bpm = 70; // 預設 70 BPM
uint8_t btn_prev_state = GPIO_PIN_SET;
/* USER CODE END PV */

/* Private function prototypes -----------------------------------------------*/
void SystemClock_Config(void);
static void MX_GPIO_Init(void);
static void MX_TIM1_Init(void);
static void MX_TIM3_Init(void);
/* USER CODE BEGIN PFP */
void Process_Button_HeartRate(void);
void Update_State_Machine(void);
void Handle_Normal_Breathing(void);
void Handle_Alarm_State(void);
/* USER CODE END PFP */

/* Private user code ---------------------------------------------------------*/
/* USER CODE BEGIN 0 */

/* USER CODE END 0 */

/**
  * @brief  The application entry point.
  * @retval int
  */
int main(void)
{
  /* USER CODE BEGIN 1 */

  /* USER CODE END 1 */

  /* MCU Configuration--------------------------------------------------------*/

  /* Reset of all peripherals, Initializes the Flash interface and the Systick. */
  HAL_Init();

  /* USER CODE BEGIN Init */

  /* USER CODE END Init */

  /* Configure the system clock */
  SystemClock_Config();

  /* USER CODE BEGIN SysInit */

  /* USER CODE BEGIN SysInit */

  /* Initialize all configured peripherals */
  MX_GPIO_Init();
  MX_TIM1_Init();
  MX_TIM3_Init();
  /* USER CODE BEGIN 2 */
  // 啟動 TIM1 Channel 1 (藍燈 PA8 呼吸燈) 與 TIM3 Channel 1 (蜂鳴器 PA6) PWM 輸出
  HAL_TIM_PWM_Start(&htim1, TIM_CHANNEL_1);
  HAL_TIM_PWM_Start(&htim3, TIM_CHANNEL_1);
  last_press_time = HAL_GetTick();
  /* USER CODE END 2 */

  /* Infinite loop */
  /* USER CODE BEGIN WHILE */
  while (1)
  {
    /* USER CODE END WHILE */

    /* USER CODE BEGIN 3 */
    // 1. 讀取外接按鈕，模擬心率輸入 (BPM 演算)
    Process_Button_HeartRate();

    // 2. 更新狀態機
    Update_State_Machine();

    // 3. 依據目前狀態執行對應之感官警報控制
    switch (current_state)
    {
      case STATE_NORMAL:
      case STATE_STRESS:
        Handle_Normal_Breathing(); // 執行漸變藍燈呼吸燈，紅綠燈與蜂鳴器適當控制
        break;

      case STATE_ALARM:
        Handle_Alarm_State();     // 執行急速閃爍與蜂鳴器警報
        break;
    }
  }
  /* USER CODE END 3 */
}

/* USER CODE BEGIN 4 */
/**
  * @brief 處理外接按鈕按壓，即時計算模擬心率 BPM
  */
void Process_Button_HeartRate(void)
{
  uint8_t btn_curr_state = HAL_GPIO_ReadPin(BUTTON_GPIO_Port, BUTTON_Pin);

  // 偵測按鈕下壓瞬間 (Falling Edge：從 High 變 Low)
  if (btn_prev_state == GPIO_PIN_SET && btn_curr_state == GPIO_PIN_RESET)
  {
    uint32_t now = HAL_GetTick();
    if ((now - last_btn_tick) > 150) // 去抖動 (150ms)
    {
      button_ever_pressed = 1;        // 標記使用者已開始互動
      press_interval = now - last_press_time;
      last_press_time = now;
      last_btn_tick = now;

      if (press_interval > 0)
      {
        bpm = 60000 / press_interval; // 計算模擬心率 BPM
      }
    }
  }
  btn_prev_state = btn_curr_state;
}

/**
  * @brief 狀態機更新：判斷心率與按壓時間是否進入異常警報狀態
  */
void Update_State_Machine(void)
{
  uint32_t now = HAL_GetTick();

  // 若曾經按過按鈕，且超過 4 秒沒再按（心率驟降/停搏），或 BPM > 150（心率過高），切換至警報狀態
  if ((button_ever_pressed && (now - last_press_time > 4000)) || (bpm > 150))
  {
    current_state = STATE_ALARM;
  }
  else if (bpm > 100)
  {
    current_state = STATE_STRESS;
  }
  else
  {
    current_state = STATE_NORMAL;
  }
}

/**
  * @brief 正常狀態：藍燈 (PA8) 2 秒漸亮 + 2 秒漸暗 平方曲線呼吸燈
  */
void Handle_Normal_Breathing(void)
{
  // 正常狀態下關閉蜂鳴器 (PA6 PWM 設為 0) 與紅綠燈
  __HAL_TIM_SET_COMPARE(&htim3, TIM_CHANNEL_1, 0);
  HAL_GPIO_WritePin(RED_LED_GPIO_Port, RED_LED_Pin, GPIO_PIN_RESET);
  HAL_GPIO_WritePin(GREEN_LED_GPIO_Port, GREEN_LED_Pin, (current_state == STATE_STRESS) ? GPIO_PIN_SET : GPIO_PIN_RESET);

  // 20ms 更新一次，配合 step_val (0~100) 達到 2 秒平滑過渡
  if ((HAL_GetTick() - last_breath_tick) >= 20)
  {
    last_breath_tick = HAL_GetTick();

    step_val += direction;

    if (step_val >= 100)
    {
      step_val = 100;
      direction = -1; // 反轉為漸暗
    }
    else if (step_val <= 0)
    {
      step_val = 0;
      direction = 1;  // 反轉為漸亮
    }

    // 二次方 Gamma 校正：放慢暗部變化，消除視覺跳階感
    ccr_val = step_val * step_val;
    __HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_1, ccr_val);
  }
}

/**
  * @brief 異常狀態：藍燈急速爆閃、紅燈點亮與蜂鳴器同步警報 (PA6)
  */
void Handle_Alarm_State(void)
{
  HAL_GPIO_WritePin(GREEN_LED_GPIO_Port, GREEN_LED_Pin, GPIO_PIN_RESET);

  // 100ms 急速閃爍與鳴叫
  if ((HAL_GetTick() - last_alarm_tick) >= 100)
  {
    last_alarm_tick = HAL_GetTick();

    static uint8_t toggle = 0;
    toggle = !toggle;

    if (toggle)
    {
      __HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_1, 10000); // 藍燈最大亮度
      __HAL_TIM_SET_COMPARE(&htim3, TIM_CHANNEL_1, 5000);  // 蜂鳴器響 (PA6 PWM)
      HAL_GPIO_WritePin(RED_LED_GPIO_Port, RED_LED_Pin, GPIO_PIN_SET); // 紅燈亮
    }
    else
    {
      __HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_1, 0);     // 藍燈熄滅
      __HAL_TIM_SET_COMPARE(&htim3, TIM_CHANNEL_1, 0);     // 蜂鳴器靜音
      HAL_GPIO_WritePin(RED_LED_GPIO_Port, RED_LED_Pin, GPIO_PIN_RESET); // 紅燈滅
    }
  }
}
/* USER CODE END 4 */

/**
  * @brief System Clock Configuration
  * @retval None
  */
void SystemClock_Config(void)
{
  RCC_OscInitTypeDef RCC_OscInitStruct = {0};
  RCC_ClkInitTypeDef RCC_ClkInitStruct = {0};

  HAL_PWREx_ControlVoltageScaling(PWR_REGULATOR_VOLTAGE_SCALE1_BOOST);

  RCC_OscInitStruct.OscillatorType = RCC_OSCILLATORTYPE_HSI;
  RCC_OscInitStruct.HSIState = RCC_HSI_ON;
  RCC_OscInitStruct.HSICalibrationValue = RCC_HSICALIBRATION_DEFAULT;
  RCC_OscInitStruct.PLL.PLLState = RCC_PLL_ON;
  RCC_OscInitStruct.PLL.PLLSource = RCC_PLLSOURCE_HSI;
  RCC_OscInitStruct.PLL.PLLM = RCC_PLLM_DIV4;
  RCC_OscInitStruct.PLL.PLLN = 85;
  RCC_OscInitStruct.PLL.PLLP = RCC_PLLP_DIV2;
  RCC_OscInitStruct.PLL.PLLQ = RCC_PLLQ_DIV2;
  RCC_OscInitStruct.PLL.PLLR = RCC_PLLR_DIV2;
  if (HAL_RCC_OscConfig(&RCC_OscInitStruct) != HAL_OK)
  {
    Error_Handler();
  }

  RCC_ClkInitStruct.ClockType = RCC_CLOCKTYPE_HCLK|RCC_CLOCKTYPE_SYSCLK
                              |RCC_CLOCKTYPE_PCLK1|RCC_CLOCKTYPE_PCLK2;
  RCC_ClkInitStruct.SYSCLKSource = RCC_SYSCLKSOURCE_PLLCLK;
  RCC_ClkInitStruct.AHBCLKDivider = RCC_SYSCLK_DIV1;
  RCC_ClkInitStruct.APB1CLKDivider = RCC_HCLK_DIV1;
  RCC_ClkInitStruct.APB2CLKDivider = RCC_HCLK_DIV1;

  if (HAL_RCC_ClockConfig(&RCC_ClkInitStruct, FLASH_LATENCY_4) != HAL_OK)
  {
    Error_Handler();
  }
}

/**
  * @brief TIM1 Initialization Function (應用於藍燈 PA8 呼吸燈)
  * @param None
  * @retval None
  */
static void MX_TIM1_Init(void)
{
  TIM_ClockConfigTypeDef sClockSourceConfig = {0};
  TIM_MasterConfigTypeDef sMasterConfig = {0};
  TIM_OC_InitTypeDef sConfigOC = {0};
  TIM_BreakDeadTimeConfigTypeDef sBreakDeadTimeConfig = {0};

  htim1.Instance = TIM1;
  htim1.Init.Prescaler = 16;
  htim1.Init.CounterMode = TIM_COUNTERMODE_UP;
  htim1.Init.Period = 9999;
  htim1.Init.ClockDivision = TIM_CLOCKDIVISION_DIV1;
  htim1.Init.RepetitionCounter = 0;
  htim1.Init.AutoReloadPreload = TIM_AUTORELOAD_PRELOAD_DISABLE;
  if (HAL_TIM_Base_Init(&htim1) != HAL_OK)
  {
    Error_Handler();
  }
  sClockSourceConfig.ClockSource = TIM_CLOCKSOURCE_INTERNAL;
  if (HAL_TIM_ConfigClockSource(&htim1, &sClockSourceConfig) != HAL_OK)
  {
    Error_Handler();
  }
  if (HAL_TIM_PWM_Init(&htim1) != HAL_OK)
  {
    Error_Handler();
  }
  sMasterConfig.MasterOutputTrigger = TIM_TRGO_RESET;
  sMasterConfig.MasterSlaveMode = TIM_MASTERSLAVEMODE_DISABLE;
  if (HAL_TIMEx_MasterConfigSynchronization(&htim1, &sMasterConfig) != HAL_OK)
  {
    Error_Handler();
  }
  sConfigOC.OCMode = TIM_OCMODE_PWM1;
  sConfigOC.Pulse = 0;
  sConfigOC.OCPolarity = TIM_OCPOLARITY_HIGH;
  sConfigOC.OCFastMode = TIM_OCFAST_DISABLE;
  if (HAL_TIM_PWM_ConfigChannel(&htim1, &sConfigOC, TIM_CHANNEL_1) != HAL_OK)
  {
    Error_Handler();
  }

  sBreakDeadTimeConfig.OffStateRunMode = TIM_OSSR_DISABLE;
  sBreakDeadTimeConfig.OffStateIDLEMode = TIM_OSSI_DISABLE;
  sBreakDeadTimeConfig.LockLevel = TIM_LOCKLEVEL_OFF;
  sBreakDeadTimeConfig.DeadTime = 0;
  sBreakDeadTimeConfig.BreakState = TIM_BREAK_DISABLE;
  sBreakDeadTimeConfig.BreakPolarity = TIM_BREAKPOLARITY_HIGH;
  sBreakDeadTimeConfig.BreakFilter = 0;
  sBreakDeadTimeConfig.AutomaticOutput = TIM_AUTOMATICOUTPUT_DISABLE;
  if (HAL_TIMEx_ConfigBreakDeadTime(&htim1, &sBreakDeadTimeConfig) != HAL_OK)
  {
    Error_Handler();
  }
  HAL_TIM_MspPostInit(&htim1);
}

/**
  * @brief TIM3 Initialization Function (應用於蜂鳴器 PA6 / TIM3_CH1)
  * @param None
  * @retval None
  */
static void MX_TIM3_Init(void)
{
  TIM_ClockConfigTypeDef sClockSourceConfig = {0};
  TIM_MasterConfigTypeDef sMasterConfig = {0};
  TIM_OC_InitTypeDef sConfigOC = {0};

  htim3.Instance = TIM3;
  htim3.Init.Prescaler = 16;
  htim3.Init.CounterMode = TIM_COUNTERMODE_UP;
  htim3.Init.Period = 9999;
  htim3.Init.ClockDivision = TIM_CLOCKDIVISION_DIV1;
  htim3.Init.AutoReloadPreload = TIM_AUTORELOAD_PRELOAD_DISABLE;
  if (HAL_TIM_Base_Init(&htim3) != HAL_OK)
  {
    Error_Handler();
  }
  sClockSourceConfig.ClockSource = TIM_CLOCKSOURCE_INTERNAL;
  if (HAL_TIM_ConfigClockSource(&htim3, &sClockSourceConfig) != HAL_OK)
  {
    Error_Handler();
  }
  if (HAL_TIM_PWM_Init(&htim3) != HAL_OK)
  {
    Error_Handler();
  }
  sMasterConfig.MasterOutputTrigger = TIM_TRGO_RESET;
  sMasterConfig.MasterSlaveMode = TIM_MASTERSLAVEMODE_DISABLE;
  if (HAL_TIMEx_MasterConfigSynchronization(&htim3, &sMasterConfig) != HAL_OK)
  {
    Error_Handler();
  }
  sConfigOC.OCMode = TIM_OCMODE_PWM1;
  sConfigOC.Pulse = 0;
  sConfigOC.OCPolarity = TIM_OCPOLARITY_HIGH;
  sConfigOC.OCFastMode = TIM_OCFAST_DISABLE;
  if (HAL_TIM_PWM_ConfigChannel(&htim3, &sConfigOC, TIM_CHANNEL_1) != HAL_OK)
  {
    Error_Handler();
  }
  HAL_TIM_MspPostInit(&htim3);
}

/**
  * @brief GPIO Initialization Function
  * @param None
  * @retval None
  */
static void MX_GPIO_Init(void)
{
  GPIO_InitTypeDef GPIO_InitStruct = {0};

  /* GPIO Ports Clock Enable */
  __HAL_RCC_GPIOC_CLK_ENABLE();
  __HAL_RCC_GPIOF_CLK_ENABLE();
  __HAL_RCC_GPIOA_CLK_ENABLE();
  __HAL_RCC_GPIOB_CLK_ENABLE();

  /* 初始輸出狀態設為低電位 */
  HAL_GPIO_WritePin(GPIOA, RED_LED_Pin|GREEN_LED_Pin, GPIO_PIN_RESET);

  /* 設定紅燈 (PA5) 與綠燈 (PA7) 為一般 GPIO 輸出 */
  GPIO_InitStruct.Pin = RED_LED_Pin|GREEN_LED_Pin;
  GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_PP;
  GPIO_InitStruct.Pull = GPIO_NOPULL;
  GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_LOW;
  HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);

  /* 設定外接按鈕 (PC10) 為輸入，並啟用內部上拉確保訊號穩定 */
  GPIO_InitStruct.Pin = BUTTON_Pin;
  GPIO_InitStruct.Mode = GPIO_MODE_INPUT;
  GPIO_InitStruct.Pull = GPIO_PULLUP;
  HAL_GPIO_Init(BUTTON_GPIO_Port, &GPIO_InitStruct);
}

/**
  * @brief  This function is executed in case of error occurrence.
  * @retval None
  */
void Error_Handler(void)
{
  __disable_irq();
  while (1)
  {
  }
}
