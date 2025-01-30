# dc-motor-control-CJ
DC motor control with PWM STM32

### Power supply design
This board received power from a 12V DC supply that supplies the onboard devices and also supplies 
the DC motor. I then step it down using a TPS54302 SMPS IC to 3.3V, 3A. This is enough to power the STM32F4 MCU.

### Current detection and proctection design
To be able to do overcurent proctection, we have to have a means of measuring 
current. I utilized the ACS712 current sensing IC for non-invasive current measurement. According to the motor this board is designed for, I limited the current to 20A.   
This is not a generic board for all motors, therefore the motor specs have been listed in the schematic.

ACS712 is connected in series with the motor and can be used to measure accurately the current being used by the motor, this value then compared to a threshold to detect overcurrent apart from INRUSH CURRENT or MAX-LOAD CURRENT.

### MCU design
I selected STM32F401CCU6 MCU for this design. It is powerful enough and has enough pins and peripherals such as ADC which I heavily utilized for current and PWM motor control 

### Motor design 
The motor that can be used with design is listed below: 

|SPEC|VALUE|
|---|---|
|MAX NOMINAL VOLTAGE| 12V|
|IN-RUSH CURRENT|8A, <45A|
|MAX LOAD CURRENT|<45A|

### MOSFET selection 
The currents stated above were determined by the MOSFET I decided to use. I used IRLZ44N, which is a N-channel MOSFET with a MAX Id of 45A, more than capable for the selected motor.

### Schematic files 
Schematic PDF file can be found in the ```schematic``` folder

## PCB Design Files 
Following the schematic above, the following PCB was designed.
### PCB top 
![pcb](./pcb-images/pcb-3d.png)

### PCB Top tracks
![pcb](./pcb-images/pcb-tracks.png)

### PCB top only outline
![pcb](./pcb-images/top-outline.png)

## Firmware design
The firmware I wrote is in the ```firmware``` folder.
### General flow

### START/STOP
I assumed a momentary push button for this logic. When the used presses the button the first time, the motor starts. 

Pressing the push button the second time stops the motor. This is represented in the flowchart below:

![start-stop logic](./misc/start-stop.png)

### Speed control input signal and PWM control signal
For this part I perform a SINGLE MODE SCAN on the ADC CHANNEL 4 which is connected to the potentiometer pin. This value is then LEFT shifted by 4 to fit within the 16-bit TIMER value. After this I write the value to the Timer CCR1 register which sets the PWM. The snippet is shown below:

```c

	  // configure first channel - potentiometer
	  ADC_CH_Cfg.Channel = ADC_CHANNEL_4;
	  HAL_ADC_ConfigChannel(&hadc1, &ADC_CH_Cfg);
	  // start conversion
	  HAL_ADC_Start(&hadc1);

	  // poll for converted result
	  // with a timeout of 1 second
	  HAL_ADC_PollForConversion(&hadc1, 1);

	  // read ADC
	  AD_POT_RES = HAL_ADC_GetValue(&hadc1);

	  // this part here controls the motor PWM signal,
	  // varying the signal based on the Potentiometer value
	  TIM2->CCR1 = (AD_POT_RES << 4);
	  HAL_Delay(1);

```


### Overcurrent detection 
For overcurrent detection, I used the ACS712 to read motor current. I read this value using the ADC and perfom conversion. SINGLE SCAN MODE is used t read the ADC1_CHANNEL_6 to which the CURRENT_SENSE pin is connected. 

```c
 //===== over-current protection ===============
	  // configure second channel - ACS current meter
	  ADC_CH_Cfg.Channel = ADC_CHANNEL_6;
	  HAL_ADC_ConfigChannel(&hadc1, &ADC_CH_Cfg);
	  // read the ADC value from the current pin
	  HAL_ADC_PollForConversion(&hadc1, 1);
	  ADC_CURRENT = HAL_ADC_GetValue(&hadc1);


```

This value is then converted as follows: Since STM32 ADC Vref is 3.3V, the midpoint is 1.65V. The resolution per Amp of the ACS712 is 0.185V, which means we get 0.185V for every ampere read. I perfomed the following conversion to convert DC readings to real current values.

```c
    // convert to real current value
    // assuming ADC midpoint is 1.65V
    CONVERTED_CURRENT = (1.65 - (ADC_CURRENT * 3.3/4095)) * 0.185;
```

The conveted value is checked against a set current THRESHOLD to check for fault.

```c
    // check against the threshold
    if(CONVERTED_CURRENT > CURRENT_THRESHOLD) {
        // fault, turn off the motor
        // set PWM to 0% duty cycle
        TIM2->CCR1 = 0;
        fault_detected= 1;
    } else {
        fault_detected = 0; // reset flag
    }

```

### Status indication 
For status indication, I two LEDS that turn ON/OFF based on a set ```fault_detected``` flag. If the flag is SET, the RED led turns ON, green turns OFF. and vice versa. 

```c
/*
 * @brief Handles the status LEDs
 */
void LEDControl() {
	if(fault_detected) {
		HAL_GPIO_WritePin(GPIOA, STATUS_LED_GRN_Pin, GPIO_PIN_RESET);
		HAL_GPIO_WritePin(GPIOA, STATUS_LED_RED_Pin, GPIO_PIN_SET);
	} else {
		HAL_GPIO_WritePin(GPIOA, STATUS_LED_GRN_Pin, GPIO_PIN_SET);
		HAL_GPIO_WritePin(GPIOA, STATUS_LED_RED_Pin, GPIO_PIN_RESET);
	}
}


```
