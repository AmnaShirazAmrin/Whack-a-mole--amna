/* USER CODE BEGIN Header */
/**
  ******************************************************************************
- @file           : main.c
- @brief          : Whack-A-Mole Game (Slower Speed + 3-Wrong Penalty Flash)
  ******************************************************************************
  */
/* USER CODE END Header */

/* Includes ------------------------------------------------------------------*/
#include "main.h"
#include <stdint.h>
#include <stdlib.h>

/* Private variables ---------------------------------------------------------*/
// LEDs: PB9 (Mole 0), PB8 (Mole 1), PB7 (Mole 2), PB5 (Mole 3), PB4 (Mole 4)
GPIO_TypeDef* ledPorts[5] = {GPIOB, GPIOB, GPIOB, GPIOB, GPIOB};
uint16_t ledPins[5] = {GPIO_PIN_9, GPIO_PIN_8, GPIO_PIN_7, GPIO_PIN_5, GPIO_PIN_4};

// Buttons: PB13 (Btn 0), PA10 (Btn 1), PA9 (Btn 2), PB14 (Btn 3), PB15 (Btn 4)
GPIO_TypeDef* btnPorts[5] = {GPIOB, GPIOA, GPIOA, GPIOB, GPIOB};
uint16_t btnPins[5] = {GPIO_PIN_13, GPIO_PIN_10, GPIO_PIN_9, GPIO_PIN_14, GPIO_PIN_15};

/* Private function prototypes -----------------------------------------------*/
void SystemClock_Config(void);
static void MX_GPIO_Init(void);
void Play_Startup_Animation(void);
void Play_Penalty_Animation(void);

int main(void)
{
  HAL_Init();
  SystemClock_Config();
  MX_GPIO_Init();

  // Seed random number generator
  srand(HAL_GetTick());

  // Visual startup sequence
  Play_Startup_Animation();

  int score = 0;
  int wrong_count = 0;
  int current_mole = 0;
  uint32_t mole_delay = 2000; // Relaxed starting speed: 2 seconds per mole

  /* Infinite loop */
  /* USER CODE BEGIN WHILE */
  while (1)
  {
      // 1. Pick a random mole (0 to 4)
      current_mole = rand() % 5;

      // Turn on the selected mole LED
      HAL_GPIO_WritePin(ledPorts[current_mole], ledPins[current_mole], GPIO_PIN_SET);

      // 2. Wait for player input within the time window
      uint32_t start_time = HAL_GetTick();
      int hit_registered = 0;

      while ((HAL_GetTick() - start_time) < mole_delay)
      {
          // Check all buttons
          for (int i = 0; i < 5; i++)
          {
              // Active-Low check (Button pressed connects pin to GND)
              if (HAL_GPIO_ReadPin(btnPorts[i], btnPins[i]) == GPIO_PIN_RESET)
              {
                  // Debounce delay
                  HAL_Delay(50);
                  while(HAL_GPIO_ReadPin(btnPorts[i], btnPins[i]) == GPIO_PIN_RESET); // Wait for release

                  if (i == current_mole)
                  {
                      // Correct button hit!
                      score++;
                      hit_registered = 1;

                      // Gradually speed up as score increases (minimum delay 800ms)
                      if (mole_delay > 800) {
                          mole_delay -= 50;
                      }
                  }
                  else
                  {
                      // Wrong button pressed
                      wrong_count++;
                      score = (score > 0) ? score - 1 : 0;

                      // If 3 wrong answers are given, trigger all-LED shine penalty
                      if (wrong_count >= 3)
                      {
                          Play_Penalty_Animation();
                          wrong_count = 0; // Reset wrong counter
                      }
                  }
                  break;
              }
          }

          if (hit_registered)
          {
              break; // Exit timer loop early on successful hit
          }
      }

      // Turn off the mole LED before next round
      HAL_GPIO_WritePin(ledPorts[current_mole], ledPins[current_mole], GPIO_PIN_RESET);

      // Short pause between moles
      HAL_Delay(300);
  }
  /* USER CODE END WHILE */
}

void Play_Startup_Animation(void)
{
    // Flash all LEDs twice
    for(int j = 0; j < 2; j++) {
        for(int i = 0; i < 5; i++) {
            HAL_GPIO_WritePin(ledPorts[i], ledPins[i], GPIO_PIN_SET);
        }
        HAL_Delay(200);
        for(int i = 0; i < 5; i++) {
            HAL_GPIO_WritePin(ledPorts[i], ledPins[i], GPIO_PIN_RESET);
        }
        HAL_Delay(200);
    }
}

void Play_Penalty_Animation(void)
{
    // Shine all LEDs together when 3 wrong answers are reached
    for(int i = 0; i < 5; i++) {
        HAL_GPIO_WritePin(ledPorts[i], ledPins[i], GPIO_PIN_SET);
    }
    HAL_Delay(600); // Hold all on for 600ms
    for(int i = 0; i < 5; i++) {
        HAL_GPIO_WritePin(ledPorts[i], ledPins[i], GPIO_PIN_RESET);
    }
    HAL_Delay(200);
}

void SystemClock_Config(void)
{
  RCC_OscInitTypeDef RCC_OscInitStruct = {0};
  RCC_ClkInitTypeDef RCC_ClkInitStruct = {0};

  __HAL_RCC_PWR_CLK_ENABLE();
  __HAL_PWR_VOLTAGESCALING_CONFIG(PWR_REGULATOR_VOLTAGE_SCALE2);

  RCC_OscInitStruct.OscillatorType = RCC_OSCILLATORTYPE_HSI;
  RCC_OscInitStruct.HSIState = RCC_HSI_ON;
  RCC_OscInitStruct.HSICalibrationValue = RCC_HSICALIBRATION_DEFAULT;
  RCC_OscInitStruct.PLL.PLLState = RCC_PLL_NONE;
  if (HAL_RCC_OscConfig(&RCC_OscInitStruct) != HAL_OK)
  {
    Error_Handler();
  }

  RCC_ClkInitStruct.ClockType = RCC_CLOCKTYPE_HCLK|RCC_CLOCKTYPE_SYSCLK
                              |RCC_CLOCKTYPE_PCLK1|RCC_CLOCKTYPE_PCLK2;
  RCC_ClkInitStruct.SYSCLKSource = RCC_SYSCLKSOURCE_HSI;
  RCC_ClkInitStruct.AHBCLKDivider = RCC_SYSCLK_DIV1;
  RCC_ClkInitStruct.APB1CLKDivider = RCC_HCLK_DIV1;
  RCC_ClkInitStruct.APB2CLKDivider = RCC_HCLK_DIV1;

  if (HAL_RCC_ClockConfig(&RCC_ClkInitStruct, FLASH_LATENCY_0) != HAL_OK)
  {
    Error_Handler();
  }
}

static void MX_GPIO_Init(void)
{
  GPIO_InitTypeDef GPIO_InitStruct = {0};

  __HAL_RCC_GPIOA_CLK_ENABLE();
  __HAL_RCC_GPIOB_CLK_ENABLE();

  // Configure LEDs (PB9, PB8, PB7, PB5, PB4)
  HAL_GPIO_WritePin(GPIOB, GPIO_PIN_9|GPIO_PIN_8|GPIO_PIN_7|GPIO_PIN_5|GPIO_PIN_4, GPIO_PIN_RESET);
  GPIO_InitStruct.Pin = GPIO_PIN_9|GPIO_PIN_8|GPIO_PIN_7|GPIO_PIN_5|GPIO_PIN_4;
  GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_PP;
  GPIO_InitStruct.Pull = GPIO_NOPULL;
  GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_LOW;
  HAL_GPIO_Init(GPIOB, &GPIO_InitStruct);

  // Configure Buttons PA10, PA9 with Pull-Up
  GPIO_InitStruct.Pin = GPIO_PIN_10|GPIO_PIN_9;
  GPIO_InitStruct.Mode = GPIO_MODE_INPUT;
  GPIO_InitStruct.Pull = GPIO_PULLUP;
  HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);

  // Configure Buttons PB13, PB14, PB15 with Pull-Up
  GPIO_InitStruct.Pin = GPIO_PIN_13|GPIO_PIN_14|GPIO_PIN_15;
  GPIO_InitStruct.Mode = GPIO_MODE_INPUT;
  GPIO_InitStruct.Pull = GPIO_PULLUP;
  HAL_GPIO_Init(GPIOB, &GPIO_InitStruct);
}

void Error_Handler(void)
{
  while(1) {}
}
