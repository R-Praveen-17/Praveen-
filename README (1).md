/* 
 * Non-Invasive Smart Error Alert System
 * ARM Cortex-M / STM32
 *
 * Functions:
 * 1. Read machine display/fault signals through optocouplers
 * 2. Convert binary pattern into fault code
 * 3. Identify fault using lookup table
 * 4. Send fault alert through LoRa
 */

#include "main.h"
#include <string.h>
#include <stdio.h>

/* ---------------- GPIO DEFINITIONS ---------------- */

/*
 * Four optocoupler outputs are connected to STM32 GPIO pins.
 *
 * Example:
 * PC0 -> Optocoupler 1
 * PC1 -> Optocoupler 2
 * PC2 -> Optocoupler 3
 * PC3 -> Optocoupler 4
 */

#define FAULT_INPUT_PORT GPIOC

#define FAULT_BIT0 GPIO_PIN_0
#define FAULT_BIT1 GPIO_PIN_1
#define FAULT_BIT2 GPIO_PIN_2
#define FAULT_BIT3 GPIO_PIN_3

/* ---------------- UART ---------------- */

extern UART_HandleTypeDef huart2;

/* ---------------- FAULT TABLE ---------------- */

typedef struct
{
    uint8_t pattern;
    const char *fault_name;
} FaultPattern;

/*
 * Pattern lookup table
 *
 * Modify these values according to the actual
 * machine display signal patterns.
 */

FaultPattern faultTable[] =
{
    {0x00, "NO FAULT"},
    {0x01, "MOTOR FAULT"},
    {0x02, "OVER TEMPERATURE"},
    {0x03, "OVER CURRENT"},
    {0x04, "LOW VOLTAGE"},
    {0x05, "HIGH VOLTAGE"},
    {0x06, "SENSOR FAULT"},
    {0x07, "COMMUNICATION FAULT"},
    {0x08, "EMERGENCY STOP"},
    {0x09, "UNKNOWN MACHINE FAULT"}
};

#define FAULT_TABLE_SIZE \
    (sizeof(faultTable) / sizeof(faultTable[0]))

/* ---------------- FUNCTION PROTOTYPES ---------------- */

uint8_t ReadFaultPattern(void);
const char* GetFaultName(uint8_t pattern);
void SendLoRaAlert(const char *fault);
void SystemDelay(void);

/* ---------------- MAIN ---------------- */

int main(void)
{
    HAL_Init();

    SystemClock_Config();

    MX_GPIO_Init();
    MX_USART2_UART_Init();

    const char *fault;

    while (1)
    {
        /* Read optocoupler signals */
        uint8_t pattern = ReadFaultPattern();

        /* Convert pattern into fault name */
        fault = GetFaultName(pattern);

        /* Send alert only when fault exists */
        if (pattern != 0x00)
        {
            SendLoRaAlert(fault);

            /* Avoid repeated alerts */
            HAL_Delay(2000);
        }

        HAL_Delay(100);
    }
}

/* ---------------- READ FAULT PATTERN ---------------- */

uint8_t ReadFaultPattern(void)
{
    uint8_t pattern = 0;

    if (HAL_GPIO_ReadPin(FAULT_INPUT_PORT, FAULT_BIT0))
        pattern |= 0x01;

    if (HAL_GPIO_ReadPin(FAULT_INPUT_PORT, FAULT_BIT1))
        pattern |= 0x02;

    if (HAL_GPIO_ReadPin(FAULT_INPUT_PORT, FAULT_BIT2))
        pattern |= 0x04;

    if (HAL_GPIO_ReadPin(FAULT_INPUT_PORT, FAULT_BIT3))
        pattern |= 0x08;

    return pattern;
}

/* ---------------- FAULT LOOKUP ---------------- */

const char* GetFaultName(uint8_t pattern)
{
    for (uint8_t i = 0; i < FAULT_TABLE_SIZE; i++)
    {
        if (faultTable[i].pattern == pattern)
        {
            return faultTable[i].fault_name;
        }
    }

    return "UNKNOWN FAULT";
}

/* ---------------- LORA ALERT ---------------- */

void SendLoRaAlert(const char *fault)
{
    char message[100];

    snprintf(message,
             sizeof(message),
             "ALERT: Machine Fault Detected -> %s\r\n",
             fault);

    /*
     * UART2 is connected to the LoRa module.
     *
     * Example:
     * STM32 TX -> LoRa RX
     * STM32 RX -> LoRa TX
     */

    HAL_UART_Transmit(
        &huart2,
        (uint8_t*)message,
        strlen(message),
        HAL_MAX_DELAY
    );
}
