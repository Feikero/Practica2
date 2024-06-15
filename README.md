# Practica 2

## Part A
### Codi en línea
```cpp
#include <Arduino.h>

 struct Button { 
  const uint8_t PIN; 
  uint32_t numberKeyPresses; 
  bool pressed; 
}; 
Button button1 = {18, 0, false}; 
void IRAM_ATTR isr() { 
  button1.numberKeyPresses += 1; 
  button1.pressed = true; 
} 
void setup() { 
  Serial.begin(115200); 
  pinMode(button1.PIN, INPUT_PULLUP); 
  attachInterrupt(button1.PIN, isr, FALLING); 
} 
void loop() { 
  if (button1.pressed) { 
      Serial.printf("Button 1 has been pressed %u times\n", 
button1.numberKeyPresses); 
      button1.pressed = false; 
  } 
  //Detach Interrupt after 1 Minute 
  static uint32_t lastMillis = 0; 
  if (millis() - lastMillis > 60000) { 
    lastMillis = millis(); 
    detachInterrupt(button1.PIN); 
     Serial.println("Interrupt Detached!"); 
  } 
} 
```
### Explicació del codi
`1.Inclusió de llibreries`
```cpp
#include <Arduino.h>
```
Aquí s'inclou la llibreria d'Arduino.h la qual ens permet utilitzar les funcions bàsiques del microcontrolador.

`2.Definició d'estructura "Button"`
```cpp
struct Button { 
  const uint8_t PIN; 
  uint32_t numberKeyPresses; 
  bool pressed; 
};
```
**Aquí es defineix l'estructura "Button" de la següent manera:**
- **PIN:** El pin al que está conectat el botó.
- **numberKeyPresses:** Un contador per el número de vegades les quals el botó ha sigut presionat.
- **pressed:** Un boolea per saber si el botó està presionat.

`3.Inicialització de una instància de "Button"`
```cpp
Button button1 = {18, 0, false}; 
```
**Es crea una instancia button1 de l'estructura Button, inicialitzant:**
- **'PIN'** amb el valor 18.
- **'numberKeyPresses'** amb 0.
- **'pressed'** amb false.

`4.Funció d'interrupció`
```cpp
void IRAM_ATTR isr() { 
  button1.numberKeyPresses += 1; 
  button1.pressed = true; 
} 
```
**La funció isr es una rutina d'interrupción que s'executa cada vegada que s'activa la interrupció del botó. Aquí:**
- S'incrementa el contador **'numberKeyPresses'** en 1.
- S'estableix **'pressed'** en true.

`5.Setup`
```cpp
void setup() { 
  Serial.begin(115200); 
  pinMode(button1.PIN, INPUT_PULLUP); 
  attachInterrupt(button1.PIN, isr, FALLING); 
} 
```
La funció setup es fa servir una vegada a l'inici del programa. Aquí es fa servir primer el **'Serial.begin'** per inicialitzar la comunicació serial a una velocitat de 115200 bauds. 
Segon el **'pinMode'** que configura el pin del botó com una entrada amb resistència pull-up interna.
Y per últim adjunta una interrupció al pin del botó que s'aciva amb una transició d'alt a baix (FALLING).

`6.Loop`
```cpp
void loop() { 
  if (button1.pressed) { 
      Serial.printf("Button 1 has been pressed %u times\n", button1.numberKeyPresses); 
      button1.pressed = false; 
  } 
  //Detach Interrupt after 1 Minute 
  static uint32_t lastMillis = 0; 
  if (millis() - lastMillis > 60000) { 
    lastMillis = millis(); 
    detachInterrupt(button1.PIN); 
    Serial.println("Interrupt Detached!"); 
  } 
} 
```
El loop és una funció que s'executa infinitament en bucle, també és la acció principal del programa.
- Si **'button1.pressed'** és 'true', dona com a sortida el número de vegades que se ha presionat el botó y es reinicia **'button1.pressed'** a 'false'.
- S'utilitza una variable estàtica **'lastMillis'** per contar el temps transcorregut.
- Si ha pasat més d'un minut (60000 milisegons) des de l'últim cop que s'ha verificat el temps, es desactiva la interrupció del botó amb  **'detachInterrupt(button1.PIN)'** i es dona com a sortida un missatge informant que s'ha desactivat la interrupció.

## Part B
### Codi en línea
```cpp
#include <Arduino.h>

volatile int interruptCounter;
int totalInterruptCounter;

hw_timer_t * timer = NULL;
portMUX_TYPE timerMux = portMUX_INITIALIZER_UNLOCKED;

void IRAM_ATTR onTimer() {
  portENTER_CRITICAL_ISR(&timerMux);
  interruptCounter++;
  portEXIT_CRITICAL_ISR(&timerMux);
}

void setup() {
  Serial.begin(115200);

  timer = timerBegin(0, 80, true);
  timerAttachInterrupt(timer, &onTimer, true);
  timerAlarmWrite(timer, 1000000, true);
  timerAlarmEnable(timer);
}

void loop() {
  if (interruptCounter > 0) {
    portENTER_CRITICAL(&timerMux);
    interruptCounter--;
    portEXIT_CRITICAL(&timerMux);

    totalInterruptCounter++;

    Serial.print("An interrupt as occurred. Total number: ");
    Serial.println(totalInterruptCounter);
  }
}
```

### Explicació del codi
`1.Inclusió de llibreries`
```cpp
#include <Arduino.h>
```
Aquí s'inclou la llibreria d'Arduino.h la qual ens permet utilitzar les funcions bàsiques del microcontrolador.

`2.Declaració de variables`
```cpp
volatile int interruptCounter;
int totalInterruptCounter;

hw_timer_t * timer = NULL;
portMUX_TYPE timerMux = portMUX_INITIALIZER_UNLOCKED;
```
- **'volatile int interruptCounter;:'** Variable que s'incrementa a la rutina d'interrupció. 'volatile' s'utiliza per indicar que la variable pot ser modificada fora del flux normal del programa (en una ISR, per exemple).
- **'int totalInterruptCounter;:'** Variable que conta el número de interrupcions processades.
- **'hw_timer_t * timer = NULL;:'** Punter al temporitzador de hardware.
- **'portMUX_TYPE timerMux = portMUX_INITIALIZER_UNLOCKED;:'** Inicialitzador per el tipús de port múltiple ('mux'), que s'utiliza per sincronitzar l'accés a 'interruptCounter'.

`3.onTimer`
```cpp
void IRAM_ATTR onTimer() {
  portENTER_CRITICAL_ISR(&timerMux);
  interruptCounter++;
  portEXIT_CRITICAL_ISR(&timerMux);
}
```
Aquesta funció s'executa cada cop que s'activa la interrupció del temporitzador:

- **'portENTER_CRITICAL_ISR(&timerMux);:'** Entra a una secció crítica per evitar que altres interrupcions interfereixin.
- **'interruptCounter++;:'** Incrementa el comptador d'interrupcions.
- **'portEXIT_CRITICAL_ISR(&timerMux);:'** Surt de la secció crítica.

`4.Setup`
```cpp
void setup() {
  Serial.begin(115200);

  timer = timerBegin(0, 80, true);
  timerAttachInterrupt(timer, &onTimer, true);
  timerAlarmWrite(timer, 1000000, true);
  timerAlarmEnable(timer);
}
```
La funció setup es fa servir una vegada a l'inici del programa. Aquí es fa servir primer el **'Serial.begin'** per inicialitzar la comunicació serial a una velocitat de 115200 bauds. 
Després es fa servir **'timer = timerBegin(0, 80, true);'** que inicialitza un comptador de hardware (0) amb un prescaler de 80 (sent la freqüència base de 80MHz, el temporitzador anirà comptant cada un microsegon.
Seguidament s'adjunta la rutina d'interrupció 'onTimer' amb **'timerAttachInterrupt(timer, &onTimer, true);'**.
Amb **'timerAlarmWrite(timer, 1000000, true);'** es configura el temporitzador perquè dispari una interrupció cada segon.
Per últim s'habilita l'alarma del temporitzador amb **'timerAlarmEnable(timer);'**.

`5.Loop`
```cpp
void loop() {
  if (interruptCounter > 0) {
    portENTER_CRITICAL(&timerMux);
    interruptCounter--;
    portEXIT_CRITICAL(&timerMux);

    totalInterruptCounter++;

    Serial.print("An interrupt has occurred. Total number: ");
    Serial.println(totalInterruptCounter);
  }
}
```
El loop és una funció que s'executa infinitament en bucle, també és la acció principal del programa.
- Si **'interruptCounter'** es major que 0, significa que com a mínim hi ha hagut una interrupció.
- Entra a una secció crítica, decrementa 'interruptCounter' y surt de la secció crítica.
- Incrementa **'totalInterruptCounter'** per portar un registre del número total de interrupcions processades.
- Dona com a sortida el missatge "An interrupt has occurred. Total number: " seguit del número total de interrupcions processades en el monitor serial.
  
