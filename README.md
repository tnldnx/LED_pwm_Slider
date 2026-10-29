# 창의설계입문 미션 - Arduino와 p5.js를 활용한 LED 제어
# Arduino-p5js-Web-Serial

## 1. 미션 소개

Arduino UNO 보드와 p5.js를 Web Serial 통신으로 연결하여 LED를 제어하고 가변저항의 입력값을 실시간으로 확인하는 미션이다.

Arduino의 푸시 버튼과 p5.js 웹 인터페이스를 통해 LED의 ON/OFF 상태를 제어하고, 가변저항의 아날로그 입력값을 웹 화면에서 확인할 수 있다.

추가적으로 PWM(Pulse Width Modulation) 기능을 활용하여 두 번째 LED의 밝기를 조절하는 기능을 구현했다. p5.js 화면 왼쪽에 세로 슬라이더를 배치하였으며, 슬라이더를 움직이면 Arduino에 PWM 값이 전송되어 LED의 밝기가 변경된다.

## 2. 사용 부품

- Arduino UNO R3
- LED 2개
- 저항
- 푸시 버튼
- 가변저항
- 16×2 I2C LCD
- 브레드보드
- 점퍼선
- USB 케이블

## 3. 회로 연결

Arduino UNO R3에 LED 2개, 푸시 버튼, 가변저항, I2C LCD를 연결했다.

첫 번째 LED는 디지털 출력 핀에 연결하여 ON/OFF를 제어하고, 두 번째 LED는 PWM 출력 핀에 연결하여 밝기를 조절하도록 구성했다.

| Arduino UNO R3 | 연결 부품 | 기능 |
|---|---|---|
| D13 | LED 1 | ON/OFF 제어 |
| D5 | LED 2 | PWM 밝기 제어 |
| D2 | 푸시 버튼 | LED 상태 전환 |
| A0 | 가변저항 | 아날로그 입력 |
| A4 | LCD SDA | I2C 데이터 |
| A5 | LCD SCL | I2C 클록 |
| 5V | LCD VCC | 전원 공급 |
| GND | LCD GND | 접지 |

LED에는 전류 제한용 저항을 직렬로 연결한다.

### Fritzing 연결도
<img width="1024" height="592" alt="image" src="https://github.com/user-attachments/assets/d9d6be0f-e4a3-485c-8975-d82b9d837925" />


## 4. 주요 구현 기능

### 4.1 LED ON/OFF 제어

p5.js 화면에 원형 버튼을 생성하여 LED의 ON/OFF 상태를 제어한다. 

사용자가 원형 버튼을 클릭하면 현재 LED 상태에 따라 `ON` 또는 `OFF` 명령이 Arduino로 전송된다.

LED가 켜져 있을 때는 원형 버튼이 초록색으로 표시되고, 꺼져 있을 때는 회색으로 표시된다.

또한 Arduino에서 전달받은 LED 상태가 p5.js 화면에 반영된다.

### 4.2 가변저항 입력값 표시

Arduino의 A0 핀에 연결된 가변저항을 통해 아날로그 입력값을 측정한다.

측정된 값은 0~1023 범위이며, 시리얼 통신을 통해 p5.js로 전달된다.

p5.js에서는 수신한 값을 숫자와 막대그래프로 표시하여 가변저항의 변화량을 직관적으로 확인할 수 있다.

### 4.3 PWM을 이용한 LED 밝기 조절

PWM(Pulse Width Modulation)은 디지털 신호의 HIGH와 LOW 상태를 빠르게 반복하면서 HIGH 상태가 유지되는 시간의 비율을 조절하는 방식이다.

p5.js에 0~1023 범위의 슬라이더를 추가하고, 슬라이더 값을 Arduino의 PWM 출력 범위인 0~255로 변환하여 전송한다.

슬라이더는 웹 화면 왼쪽에 세로 방향으로 배치하여 기존 LED ON/OFF 버튼 및 가변저항 표시 영역과 구분했다.

### 4.4 Web Serial 양방향 통신

`SerialHelper`를 사용해 Arduino와 p5.js 사이의 시리얼 통신이 처리되었다.

통신 속도는 9600 baud이다.

p5.js에서는 LED 제어 명령과 PWM 값을 Arduino로 전송하고, Arduino에서는 LED 상태와 가변저항 값을 p5.js로 전달한다.

| 통신 데이터 | 기능 |
|---|---|
| `ON` | LED 켜기 |
| `OFF` | LED 끄기 |
| `PWM:128` | PWM 값 128 전송 |
| `LED: ON` | LED 켜짐 상태 수신 |
| `LED: OFF` | LED 꺼짐 상태 수신 |
| `POT:512` | 가변저항 값 512 수신 |

### 4.5 코드 동작 원리

**1) 시리얼 통신 객체 및 변수 선언**

```javascript
let serial, connectButton;
let ledOn = false;
let potValue = 0;
let statusText = 'Arduino를 연결해 주세요.';
let s;
```

`serial`은 Arduino와의 시리얼 통신을 담당하는 객체이며, `connectButton`은 시리얼 포트의 연결 및 해제에 사용한다.

`ledOn`은 LED의 ON/OFF 상태를 저장하고, `potValue`는 Arduino에서 수신한 가변저항 값을 저장한다.

`statusText`는 연결 상태를 화면에 표시하는 문자열이며, `s`는 PWM 제어를 위한 슬라이더 객체이다.

**2) Web Serial 연결 및 슬라이더 생성**

```javascript
function setup() {
  createCanvas(600, 400);
  textFont('sans-serif');

  serial = new SerialHelper(
    9600,
    receiveLine,
    message => statusText = message
  );

  connectButton = createButton('Arduino 연결 / 해제');
  connectButton.mousePressed(() => serial.toggle());

  s = createSlider(0, 1023, 100);
  s.position(-55, 150);
  s.style('width', '160px');
  s.style('transform', 'rotate(-90deg)');
  s.input(changeled);
}
```

`createCanvas()`를 사용하여 600×400 크기의 웹 화면을 생성.

`SerialHelper`를 이용하여 통신 속도를 9600 baud로 설정하고, 데이터를 수신하면 `receiveLine()` 함수가 실행되도록 구성.

또한 Arduino 연결 및 해제를 위한 버튼을 생성.

`createSlider()`를 이용하여 PWM 제어용 슬라이더를 생성하였으며, CSS의 `rotate(-90deg)`를 적용하여 세로 방향으로 표시.

슬라이더 값이 변경되면 `changeled()` 함수가 실행.

**3) LED ON/OFF 버튼 출력**

```javascript
fill(ledOn ? '#22c55e' : '#64748b');
circle(160, 175, 110);

fill(255);
textSize(22);
text(ledOn ? 'ON' : 'OFF', 160, 183);
```

`circle()` 함수를 사용하여 LED 상태를 나타내는 원형 버튼을 생성.

`ledOn` 값이 `true`이면 초록색과 ON 문구를 표시하고, `false`이면 회색과 OFF 문구를 표시.

**4) 마우스 클릭을 이용한 LED 제어**

```javascript
async function mousePressed() {
  if (dist(mouseX, mouseY, 160, 175) > 55) return;

  const next = !ledOn;

  if (await serial.send(next ? 'ON' : 'OFF')) {
    ledOn = next;
  }
}
```

`dist()` 함수를 사용하여 마우스 클릭 위치가 원형 버튼 내부에 있는지 확인한다.

원형 버튼의 반지름인 55픽셀 이내에서 클릭이 발생하면 현재 LED 상태를 반전한다.

이후 `serial.send()`를 통해 Arduino로 `ON` 또는 `OFF` 명령을 전송한다.

명령 전송이 성공하면 `ledOn` 값을 변경하여 화면에 반영한다.

**5) 가변저항 값 시각화**

```javascript
fill('#dbe3ed');
rect(310, 155, 220, 40, 10);

fill('#3b82f6');
rect(
  310, 155,
  map(potValue, 0, 1023, 0, 220),
  40, 10
);

fill(25, 40, 60);
textSize(24);
text(potValue + ' / 1023', 420, 235);
```

`rect()` 함수를 이용하여 가변저항 값을 나타내는 막대그래프 생성.

`map()` 함수를 사용하여 0~1023 범위의 아날로그 입력값을 0~220픽셀 범위로 변환한다.

가변저항 값이 증가하면 파란색 막대의 길이가 증가하며, 하단에는 현재 입력값이 숫자로 표시된다.

**6) Arduino 데이터 수신 및 처리**

```javascript
function receiveLine(line) {
  const message = line.trim().toUpperCase();

  if (message === 'LED: ON' || message === 'ON') {
    ledOn = true;
  }
  else if (message === 'LED: OFF' || message === 'OFF') {
    ledOn = false;
  }
  else {
    const match = message.match(/^(?:POT\s*[:=]\s*)?(\d+)$/);

    if (match) {
      potValue = constrain(Number(match[1]), 0, 1023);
    }
  }
}
```

`receiveLine()` 함수는 Arduino에서 전달받은 문자열을 분석한다.

`trim()`을 통해 문자열 앞뒤의 공백을 제거하고, `toUpperCase()`를 이용하여 대문자로 변환한다.

LED 상태 정보가 수신되면 `ledOn` 변수를 변경하고, 가변저항 값이 수신되면 `potValue` 변수에 저장한다.

`constrain()` 함수는 가변저항 값이 0~1023 범위를 벗어나지 않도록 처리한다.

**7) PWM 값 변환 및 전송**

```javascript
async function changeled() {
  let value = s.value();
  let brightness = floor(map(value, 0, 1023, 0, 255));

  await serial.send("PWM:" + brightness);
}
```

`s.value()`를 통해 현재 슬라이더 값을 읽는다.

`map()` 함수를 사용하여 슬라이더 값인 0~1023을 PWM 출력 범위인 0~255로 변환한다.

이후 `floor()` 함수를 이용하여 정수로 변환하고, `serial.send()`를 통해 Arduino로 전송한다.

Arduino에서는 수신한 PWM 값을 이용하여 D5 핀에 연결된 LED의 밝기를 제어한다.

## 5. 실행 결과

Arduino와 p5.js를 Web Serial로 연결하면 웹 화면에서 LED의 ON/OFF 상태와 가변저항 입력값을 확인할 수 있다.

원형 버튼을 클릭하면 Arduino의 LED 상태가 변경되고, 가변저항을 회전하면 웹 화면의 막대그래프와 숫자가 변경된다.

또한 왼쪽 세로 슬라이더를 움직이면 Arduino로 PWM 제어 명령이 전송되어 두 번째 LED의 밝기를 조절할 수 있게 다.

### 실행 화면

<img width="312" height="255" alt="image" src="https://github.com/user-attachments/assets/42ea8b73-6918-4fa5-a9c9-64538297c949" />


### 실행 영상

시연 영상 추가 예정

## 6. 제작 과정 및 고찰

기존 프로젝트에서는 p5.js의 원형 버튼을 통해 LED의 ON/OFF 상태를 제어하고, Arduino에서 측정한 가변저항 값을 웹 화면에 표시할 수 있었다.

이번 프로젝트에서는 기존의 기능을 유지하면서 추가적으로 LED의 밝기를 조절하는 PWM 기능을 구현하였다.

처음에는 단순히 LED를 켜고 끄는 기능만 있었지만, PWM을 활용하면 디지털 출력 신호를 이용하여 LED의 밝기를 단계적으로 변경할 수 있다는 점에 주목하였다.

이를 구현하기 위해 p5.js에 슬라이더를 추가하고, 슬라이더의 입력 범위인 0~1023을 Arduino의 PWM 출력 범위인 0~255로 변환하였다.

또한 기존 웹 화면의 구성 요소와 겹치지 않도록 슬라이더를 세로 방향으로 회전시키고, 화면 왼쪽에 배치하였다.

Web Serial 통신 과정에서는 Arduino IDE의 시리얼 모니터와 웹 브라우저가 동일한 시리얼 포트를 동시에 사용할 수 없다는 점을 확인하였다. 이를 통해 시리얼 포트 연결 상태를 확인하고 관리하는 과정의 중요성을 알 수 있었다.

이번 프로젝트를 통해 Arduino의 디지털 입출력과 아날로그 입력, PWM 출력의 차이를 이해할 수 있었으며, p5.js를 활용하여 하드웨어의 상태를 실시간으로 표시하고 제어하는 방법을 학습하였다.
