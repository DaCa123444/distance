# sr04 거리 측정

### 결과
 - 출력값
</br>
<img src="/pic/Result.png" width="40%" height="30%" title="opamp_1_off"></img></br>
 - 핀 설정
</br>
<img src="/pic/core.png" width="40%" height="30%" title="opamp_1_off"></img></br>
 - tim1,2 설정
</br>
<img src="/pic/tim1.png" width="25%" height="25%" title="opamp_1_off" ></img></br>
 - clk 설정
</br>
<img src="/pic/clock_configuration.png" width="40%" height="30%" title="opamp_1_off"  ></img></br>

### 칩에서 stm32 간 처리 과정 정리
1. Trigger : ranging을 위해 trigger가 발생해야하며, 10us 이후 종료

2. echo  
 - 시작시, 자동으로 초음파가 발사되며, echo가 High를 출력합니다. 
 - 초음파가 되돌아옴을 확인하면 low가 출력됩니다.
 - 즉, echo가 활성화된 시간 동안이 초음파가 대상까지 왕복한 시간

### 논리

1. 트리거 발생시킬 경우,**구형파의 펄스간격을 PWM을 이용**하며 주기는 10us으로 설정합니다.</br>
(tim1의 1펄스 단위 : sysclk : 84MHz , PSR : 84 로 설정하여 1펄스당 1us 이다.)</br>
(카운터 주기 : count period : 10,000 -1 ; 주기 = 1/f = 10us 이다.)
  
2. echo의 시작시, 상승 펄스를 인식하는 것과 하강펄스를 인식해서 high가 유지된 시간을 확인해야합니다.</br>
상승 펄스를 인식하기 위해 tim2의 ch1을 Input Capture(**Rising edge**)로 설정하였으며 ch2는 Input Capture(**Falling edge**)로 설정하였다. (tim2도 tim1과 같은 설정)

3. Rising edge시 수행 기능은 타이머를 초기화합니다.

4. Falling edge는 타이머의 카운트값을 읽고 거리값을 출력합니다.
` D = 340m/s * t = 0.34[cm/us] * t[us] ` 

#### 코드

3. Rising edge시 수행 기능은 타이머를 초기화합니다.
```
if(htim->Channel==HAL_TIM_ACTIVE_CHANNEL_1)
		{
			// rising edge
			__HAL_TIM_SET_COUNTER(htim,0);
		}
```
4. Falling edge는 타이머의 카운트값을 읽고 거리값을 출력합니다.

```
if(htim->Channel==HAL_TIM_ACTIVE_CHANNEL_1)
		{
			// rising edge
			__HAL_TIM_SET_COUNTER(htim,0);
		}
```


#### 시행착오
1. system clock configuration에서 sysclk이 84MHz임을 깜박하고 42Mhz로 진행하여 데이터값이 계속 2배로 나온 착오를 경험.

2. pwm의 경우 callback을 사용하지 않기 때문에 NVIC setting이 필요없었으나, echo에서 Callback을 통해 타이머 카운트 값을 측정하기 위해 NVIC에서 interrupt를 활성화해야합니다.
 - 추상화IRQ -> 구체화(GPIO등)IRQ -> Callback 이 자동으로 설정되지만 활성화는 직접해줘야합니다.

3. 거리값을 계산하는 위치를 상승 엣지와 하강 엣지에서 카운터 값에서 저장한 후, 주기마다 distance를 계산하는 함수를 구현하려하였으나, 상승 엣지가 하강엣지보다 큰 값을 갖는 상황이 있으며, 카운터값을 초기화 하지 않을 경우, 일정 시간 이후 자동으로 초기화되는 문제에 직면할 수 있을 것이라 판단했다. 이를 직접 시도해보고 싶었으나, interrupt 활성화 문제로 헤메다 시도해보지 못했다.
