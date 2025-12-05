# STM32 LoRa Ra-02 (SX1278) Integration Guide

This project demonstrates how to implement Long Range (LoRa) wireless communication between two STM32F103 (Blue Pill) microcontrollers using the **LoRa Ra-02 (SX1278)** module. This guide covers wiring, STM32CubeMX configuration, and source code for both the **Transmitter** and **Receiver**.

## 🚀 Introduction

This library is a port based on Wojciech Domski's driver, adapted for the STM32 HAL library. It supports essential LoRa features:
* SPI Communication.
* Packet Transmission and Reception.
* Configuration of Frequency, Bandwidth, and Spreading Factor.

## 🛠️ Hardware & Pinout

Connect the LoRa Ra-02 module to the STM32 Blue Pill via SPI1.
**Important:** The LoRa Ra-02 module operates at **3.3V**. Connecting it to 5V will damage the module.

| LoRa Ra-02 Pin | STM32 Pin | Function | CubeMX Configuration |
| :--- | :--- | :--- | :--- |
| **3.3V** | 3.3V | Power | - |
| **GND** | GND | Ground | - |
| **NSS** (CS) | **PA4** | Chip Select | `GPIO_Output` |
| **MOSI** | **PA7** | SPI Data | `SPI1_MOSI` |
| **MISO** | **PA6** | SPI Data | `SPI1_MISO` |
| **SCK** | **PA5** | SPI Clock | `SPI1_SCK` |
| **RST** | **PB0** | Reset | `GPIO_Output` |
| **DIO0** | **PB1** | Interrupt | `GPIO_Input` (or `EXTI`) |



## ⚙️ STM32CubeMX Configuration

1.  **Connectivity -> SPI1:**
    * Mode: **Full-Duplex Master**
    * Prescaler: Select a value such that Baud Rate < 10 MBits/s (e.g., Prescaler 4 or 8).
2.  **System Core -> GPIO:**
    * **PA4 (NSS):** Output Level: `High`, Mode: `Output Push Pull`.
    * **PB0 (RST):** Output Level: `High`, Mode: `Output Push Pull`.
    * **PB1 (DIO0):** Mode: `Input` (or `External Interrupt` if using IRQ).
3.  **Project Manager:**
    * Generate Code for Keil C (MDK-ARM) or STM32CubeIDE.

## 📂 Library Integration

Copy the following files into your project folder:
* `SX1278.c`, `SX1278_hw.c` -> copy to `Src/` folder.
* `SX1278.h`, `SX1278_hw.h` -> copy to `Inc/` folder.

## 💻 Source Code

Add the following initialization code to `main.c` (Common for both Transmitter and Receiver).

### 1. Includes & Variables

Place this under `/* USER CODE BEGIN Includes */`:

```c
#include "SX1278.h"
#include <stdio.h> // For sprintf

/* Global variables */
SX1278_hw_t SX1278_hw;
SX1278_t SX1278;
```  

### 2. Initialization

Place this inside the `main()` function, before the `while(1)` loop:

```c
SX1278_hw.dio0.port = GPIOB;
SX1278_hw.dio0.pin = GPIO_PIN_1;
SX1278_hw.nss.port = GPIOA;
SX1278_hw.nss.pin = GPIO_PIN_4;
SX1278_hw.reset.port = GPIOB;
SX1278_hw.reset.pin = GPIO_PIN_0;
SX1278_hw.spi = &hspi1; // Ensure hspi1 is initialized before this

SX1278.hw = &SX1278_hw;

/* Initialize LoRa Module */
// Frequency: 433MHz, Power: 17dBm, SF: 7, BW: 125kHz
printf("Configuring LoRa Module...\r\n");
SX1278_init(&SX1278, 433000000, SX1278_POWER_17DBM, SX1278_LORA_SF_7,
            SX1278_LORA_BW_125KHZ, SX1278_LORA_CR_4_5, SX1278_LORA_CRC_EN, 10);

/* Check SPI Connection */
uint8_t ret = SX1278_SPIRead(&SX1278, 0x42); // Read version register
if (ret == 0x12) {
    printf("LoRa module OK! Version: 0x%02X\r\n", ret);
} else {
    printf("LoRa Connection Error! Read: 0x%02X\r\n", ret);
    Error_Handler(); // Stop if hardware error
}
```  

### 3. Code for TRANSMITTER
The module sends the string "Hello Master" along with an incrementing counter every second.

```c
char buffer[64];
int message_length;
int count = 0;

while (1) {
    // 1. Prepare data
    message_length = sprintf(buffer, "Hello Master %d", count++);

    // 2. Set module to TX mode
    SX1278_LoRaEntryTx(&SX1278, message_length, 2000);

    // 3. Transmit packet
    printf("Sending: %s\r\n", buffer);
    int status = SX1278_LoRaTxPacket(&SX1278, (uint8_t *)buffer, message_length, 2000);

    if (status > 0) {
        printf("Sent successfully!\r\n");
        HAL_GPIO_TogglePin(GPIOC, GPIO_PIN_13); // Toggle LED PC13
    } else {
        printf("Send failed (Timeout)\r\n");
    }

    HAL_Delay(1000);
      ```  

### 4.Code for RECEIVER
The module constantly listens for incoming packets and prints the data.

```c
SX1278_LoRaEntryRx(&SX1278, 16, 2000);

uint8_t rx_buffer[64];
int rx_len;

/* Inside the while(1) loop */
while (1) {
    // 1. Check if data is available
    rx_len = SX1278_available(&SX1278);

    if (rx_len > 0) {
        // 2. Read data from buffer
        SX1278_read(&SX1278, rx_buffer, rx_len);
        
        // Null-terminate the string for printing
        rx_buffer[rx_len] = '\0'; 
        
        printf("Received (%d bytes): %s\r\n", rx_len, rx_buffer);
        printf("RSSI: %d\r\n", SX1278_RSSI_LoRa(&SX1278)); // Signal Strength

        HAL_GPIO_TogglePin(GPIOC, GPIO_PIN_13); // Toggle LED on receive
    }
    
    HAL_Delay(10); // Small delay to prevent MCU freeze
}
