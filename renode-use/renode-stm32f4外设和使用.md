```
(stm32f407_platform) peripherals
Available peripherals:
  sysbus (SystemBus)
  │   
  ├── bitbandPeripherals (BitBanding)
  │       <0x42000000, 0x43FFFFFF>
  │       
  ├── bitbandSram (BitBanding)
  │       <0x22000000, 0x221FFFFF>
  │       
  ├── can1 (STMCAN)
  │       <0x40006400, 0x400067FF>
  │       
  ├── can2 (STMCAN)
  │       <0x40006800, 0x40006BFF>
  │       
  ├── cpu (CortexM)
  │       Slot: 0
  │       
  ├── crc (STM32_CRC)
  │       <0x40023000, 0x400233FF>
  │       
  ├── dma1 (STM32DMA)
  │       <0x40026000, 0x400263FF>
  │       
  ├── dma2 (STM32DMA)
  │       <0x40026400, 0x400267FF>
  │       
  ├── ethernet (SynopsysEthernetMAC)
  │       <0x40028000, 0x400293FF>
  │       
  ├── exti (STM32F4_EXTI)
  │       <0x40013C00, 0x40013FFF>
  │       
  ├── flash (MappedMemory)
  │       <0x08000000, 0x081FFFFF>
  │       
  ├── flash_controller (STM32F4_FlashController)
  │       <0x40023C00, 0x40023FFF>
  │       <0x1FFFC000, 0x1FFFC0FF> [region: optionBytes]
  │       
  ├── fsmcBank1 (MappedMemory)
  │       <0x60000000, 0x6FFFFFFF>
  │       
  ├── gpioPortA (STM32_GPIOPort)
  │   │   <0x40020000, 0x400203FF>
  │   │   
  │   └── UserButton (Button)
  │           
  ├── gpioPortB (STM32_GPIOPort)
  │       <0x40020400, 0x400207FF>
  │       
  ├── gpioPortC (STM32_GPIOPort)
  │       <0x40020800, 0x40020BFF>
  │       
  ├── gpioPortD (STM32_GPIOPort)
  │   │   <0x40020C00, 0x40020FFF>
  │   │   
  │   └── UserLED (LED)
  │           
  ├── gpioPortE (STM32_GPIOPort)
  │       <0x40021000, 0x400213FF>
  │       
  ├── gpioPortF (STM32_GPIOPort)
  │       <0x40021400, 0x400217FF>
  │       
  ├── gpioPortG (STM32_GPIOPort)
  │       <0x40021800, 0x40021BFF>
  │       
  ├── gpioPortH (STM32_GPIOPort)
  │       <0x40021C00, 0x40021FFF>
  │       
  ├── gpioPortI (STM32_GPIOPort)
  │       <0x40022000, 0x400223FF>
  │       
  ├── gpioPortJ (STM32_GPIOPort)
  │       <0x40022400, 0x400227FF>
  │       
  ├── gpioPortK (STM32_GPIOPort)
  │       <0x40022800, 0x40022BFF>
  │       
  ├── i2c1 (STM32F4_I2C)
  │       <0x40005400, 0x400057FF>
  │       
  ├── i2c2 (STM32F4_I2C)
  │       <0x40005800, 0x40005BFF>
  │       
  ├── i2c3 (STM32F4_I2C)
  │       <0x40005C00, 0x40005FFF>
  │       
  ├── iwdg (STM32_IndependentWatchdog)
  │       <0x40003000, 0x400033FF>
  │       
  ├── nvic (NVIC)
  │       <0xE000E000, 0xE000EFFF>
  │       
  ├── pwr (STM32_PWR)
  │       <0x40007000, 0x400073FF>
  │       
  ├── rcc (STM32F4_RCC)
  │       <0x40023800, 0x40023BFF>
  │       
  ├── rng (STM32F4_RNG)
  │       <0x50060800, 0x50060BFF>
  │       
  ├── rom1 (MappedMemory)
  │       <0x1FFF0000, 0x1FFFBFFF>
  │       
  ├── rom2 (MappedMemory)
  │       <0x1FFFC400, 0x1FFFFFFF>
  │       
  ├── rtc (STM32F4_RTC)
  │       <0x40002800, 0x40002BFF>
  │       
  ├── spi1 (STM32SPI)
  │       <0x40013000, 0x400133FF>
  │       
  ├── spi2 (STM32SPI)
  │       <0x40003800, 0x40003BFF>
  │       
  ├── spi3 (STM32SPI)
  │       <0x40003C00, 0x40003FFF>
  │       
  ├── sram (MappedMemory)
  │       <0x20000000, 0x2003FFFF>
  │       
  ├── timer1 (STM32_Timer)
  │       <0x40010000, 0x400103FF>
  │       
  ├── timer10 (STM32_Timer)
  │       <0x40014400, 0x400147FF>
  │       
  ├── timer11 (STM32_Timer)
  │       <0x40014800, 0x40014BFF>
  │       
  ├── timer12 (STM32_Timer)
  │       <0x40001800, 0x40001BFF>
  │       
  ├── timer13 (STM32_Timer)
  │       <0x40001C00, 0x40001FFF>
  │       
  ├── timer14 (STM32_Timer)
  │       <0x40002000, 0x400023FF>
  │       
  ├── timer2 (STM32_Timer)
  │       <0x40000000, 0x400003FF>
  │       
  ├── timer3 (STM32_Timer)
  │       <0x40000400, 0x400007FF>
  │       
  ├── timer4 (STM32_Timer)
  │       <0x40000800, 0x40000BFF>
  │       
  ├── timer5 (STM32_Timer)
  │       <0x40000C00, 0x40000FFF>
  │       
  ├── timer6 (STM32_Timer)
  │       <0x40001000, 0x400013FF>
  │       
  ├── timer7 (STM32_Timer)
  │       <0x40001400, 0x400017FF>
  │       
  ├── timer8 (STM32_Timer)
  │       <0x40010400, 0x400107FF>
  │       
  ├── timer9 (STM32_Timer)
  │       <0x40014000, 0x400143FF>
  │       
  ├── uart4 (STM32_UART)
  │       <0x40004C00, 0x40004CFF>
  │       
  ├── uart5 (STM32_UART)
  │       <0x40005000, 0x400050FF>
  │       
  ├── usart1 (STM32_UART)
  │       <0x40011000, 0x400110FF>
  │       
  ├── usart2 (STM32_UART)
  │       <0x40004400, 0x400044FF>
  │       
  └── usart3 (STM32_UART)
          <0x40004800, 0x400048FF>
          
```
