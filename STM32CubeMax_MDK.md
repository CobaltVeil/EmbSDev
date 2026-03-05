1. 在STM32CubeMax配置EXTI外部中断项目中，出现设置抢占优先级，造成反向中断嵌套调用造成程序卡死的问题。
系统默认生成System Tick Timer的抢占优先级为15，EXTI 0 Line一般由用户自行设置，在外部中断回调函数中如果写：
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
{
    if(GPIO_Pin == USER_KEY_Pin)
    {
        HAL_Delay(10);
        HAL_GPIO_TogglePin(GREEN_LED_GPIO_Port, GREEN_LED_Pin);
    }
}
HAL库延时函数HAL_Delay使用的是系统滴答定时器作为时间基准，而系统滴答定时器同为中断，在外部中断中触发系统滴答定时器中断会涉及中断优先级的问题，
结论：尽量不要在外部中断回调函数中调用低优先级中断函数，在外部中断回调函数中尽量少的去执行功能，可以通过设标志位再在主循环中执行的方式来处理。
