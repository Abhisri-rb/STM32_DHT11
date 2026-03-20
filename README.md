# STM32 DHT11 Temperature and Humidity Monitor with I2C LCD

This project demonstrates interfacing a DHT11 temperature and humidity sensor with an STM32 microcontroller and displaying the measured values on a 16x2 LCD using an I2C interface. The implementation uses STM32 HAL drivers.

---

### Features

- DHT11 single-wire communication protocol
- Microsecond delay using timer
- I2C-based 16x2 LCD display
- Real-time temperature and humidity monitoring
- HAL-based implementation

---

### Hardware Required

- STM32 microcontroller (F4 series)
- DHT11 sensor
- 16x2 LCD with I2C module
- Connecting wires

---

### Working Principle

1. The microcontroller sends a start signal to the DHT11 sensor.
2. The sensor responds with an acknowledgment signal.
3. The DHT11 transmits 40 bits of data:
   - Humidity integer part
   - Humidity decimal part
   - Temperature integer part
   - Temperature decimal part
   - Checksum
4. The microcontroller reads the data using timing-based decoding.
5. Temperature and humidity values are displayed on the LCD.

---


### Code Implementation

### Start Peripherals
```c
HAL_TIM_Base_Start(&htim1);   // for microsecond delay
lcd_init();                   // initialize LCD
```
**Functionality:**
- Starts the timer required for microsecond delays
- Initializes the LCD so it’s ready to display data

---

### Set Pin as Input
```c
void Set_Pin_Input(GPIO_TypeDef *GPIOx, uint16_t GPIO_Pin)
{
    GPIO_InitTypeDef GPIO_InitStruct = {0};
    GPIO_InitStruct.Pin = GPIO_Pin;
    GPIO_InitStruct.Mode = GPIO_MODE_INPUT;
    GPIO_InitStruct.Pull = GPIO_NOPULL;
    HAL_GPIO_Init(GPIOx, &GPIO_InitStruct);
}
```
**Functionality:**
- Configures GPIO pin as input
- Used to receive data from DHT11

---

### Microsecond Delay
```c
void delay_us(uint16_t us)
{
    __HAL_TIM_SET_COUNTER(&htim1, 0);
    while (__HAL_TIM_GET_COUNTER(&htim1) < us);
}
```
**Functionality:**
- Resets timer counter
- Waits until the specified microseconds pass
- Ensures precise timing required by DHT11 protocol

  ---
  
### DHT11 Start Signal
```c
void DHT11_Start(void)
{
    Set_Pin_Output(GPIOA, GPIO_PIN_1);
    HAL_GPIO_WritePin(GPIOA, GPIO_PIN_1, GPIO_PIN_RESET);
    HAL_Delay(18);

    HAL_GPIO_WritePin(GPIOA, GPIO_PIN_1, GPIO_PIN_SET);
    delay_us(20);

    Set_Pin_Input(GPIOA, GPIO_PIN_1);
}
```
**Functionality:**
- Sends start signal to DHT11
- Pulls data line LOW for 18ms to wake sensor
- Pulls HIGH briefly and switches to input mode

---

### Check Response
```c
uint8_t DHT11_Check_Response(void)
{
    uint8_t Response = 0;
    delay_us(40);

    if (!(HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_1)))
    {
        delay_us(80);
        if ((HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_1))) Response = 1;
        else Response = 0;
    }

    while ((HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_1)));

    return Response;
}
```
**Functionality:**
- Waits for sensor acknowledgment
- Detects LOW → HIGH response from DHT11
- Returns 1 if sensor responds correctly

---

### Read Data
```c
uint8_t DHT11_Read(void)
{
    uint8_t i = 0, j;
    for (j = 0; j < 8; j++)
    {
        while (!(HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_1)));
        delay_us(40);

        if (!(HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_1)))
        {
            i &= ~(1 << (7 - j));
        }
        else
        {
            i |= (1 << (7 - j));
        }
        while ((HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_1)));
    }
    return i;
}
```
**Functionality:**
- Reads 8 bits (1 byte) from DHT11
- Determines bit value based on signal duration
- Combines bits to form humidity/temperature data
  
---

### Main Loop (DHT11 + LCD)
```c
uint8_t Rh_byte1, Rh_byte2, Temp_byte1, Temp_byte2;
uint16_t SUM;
char buffer[20];

while (1)
{
    DHT11_Start();
    DHT11_Check_Response();

    Rh_byte1 = DHT11_Read();
    Rh_byte2 = DHT11_Read();
    Temp_byte1 = DHT11_Read();
    Temp_byte2 = DHT11_Read();
    SUM = DHT11_Read();

    lcd_clear();

    sprintf(buffer, "Temp: %d C", Temp_byte1);
    lcd_put_cur(0, 0);
    lcd_send_string(buffer);

    sprintf(buffer, "Hum: %d %%", Rh_byte1);
    lcd_put_cur(1, 0);
    lcd_send_string(buffer);

    HAL_Delay(1000);
}
```
**Functionality:**
- Initiates communication with DHT11
- Reads humidity and temperature values
- Displays temperature on first row of LCD
- Displays humidity on second row
- Updates data every 1 second
