# Oline_Device
基于双ARM的局部放电在线检测设备

`文件说明: `
* [1] Hardware     对应硬件设计文件 
* [2] Software     对应软件设计文件
* [3] Serial Chart 波形测试上位机

---

### Hardware硬件设计问题汇总
`历史问题1:` 集成运放的压摆率不够导致波形存在一定失真,体现在脉冲的幅度存在较大是衰减.

`历史问题2:` LM2596电源的设计,存在问题使得电感存在啸叫.

---

### Software软件设计问题汇总
* 使用STM32H750 和 STM32L151C8T6进行设计,使用HAL库进行开发.STM32L151C8T6固有的低功耗检测程序不做改动
* STM32H750的ADC采样率设置为2Msps 对应工频周期20ms 采样40000个数据点

`HAL构建问题1:` stm32h7xx_hal_conf.h文件修改
``` 
#修改1 99行:

#if !defined (HSE_VALUE) 
#define HSE_VALUE ((uint32_t)8000000) //外部高速振荡器的值 单位Hz
#endif /* HSE_VALUE */
```
`HAL构建问题2:` system32h7xx_stm32h7xx.h文件修改
``` 
#修改2 224行: 修改为SCB->VTOR = FLASH_BANK1_BASE | VECT_TAB_OFFSET;

/* Configure the Vector Table location add offset address ------------------*/
#ifdef VECT_TAB_SRAM
  SCB->VTOR = D1_AXISRAM_BASE  | VECT_TAB_OFFSET; /* Vector Table Relocation in Internal SRAM */
#else
  SCB->VTOR = FLASH_BANK1_BASE | VECT_TAB_OFFSET; /* Vector Table Relocation to APPLICATION_ADDRESS in preprocessor defines */
#endif  
```
`HAL构建问题3:` H750数据传输问题 cache一致性问题
``` 
sys_cache_enable();  //开启L1-Cache
...
SCB_InvalidateDCache(); //数据更新至实际物理内存
```

### H750软件测试程序（自定义）
```
//开启DMA数据传输
delay_ms(100);
adc_dma_enable(ADC_DMA_BUF_SIZE);
  //测试ADC DMA数据传输
  for (;;){
  if(g_adc_dma_sta == 1){
    if(sample_cycle < CYCLE_SAMPLE){
      pd_data_get(sample_cycle);
      for (i = 0; i < WIN_SPLIT;++i){
        printf("%f\n", pd_pulse_buf[sample_cycle][i] * 3.3 / 16384);
        delay_ms(10);
      }
      ++sample_cycle;
      adc_dma_enable(ADC_DMA_BUF_SIZE);
    }else{
      sample_cycle = 0;
    }
  }
}
```
```
//串口1数据传输测试
for (;;){
  uart1_send_buffer[0] = 1;
  usart1_send_data((uint8_t *)uart1_send_buffer,1);
  delay_ms(100);
}
```
```
//串口2数据传输测试 + 串口中断接收测试LED DEUBG
for (;;){
   uart2_send_buffer[0] = 1;
  usart2_send_data((uint8_t *)uart2_send_buffer,1);
  delay_ms(100);
}
```
