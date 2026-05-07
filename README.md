# Sistema-Inteligente-de-Control-Dual-con-ESP32-e-Interrupciones
//Enlace WOKWI:https://wokwi.com/projects/463327584130075649
#define PIN_LED1       25   
#define PIN_LED2       26  
#define PIN_POT1       34   
#define PIN_POT2       35   
#define PIN_BTN1       18   
#define PIN_BTN2       19   

#define LEDC_CHANNEL   0
#define LEDC_FREQ_HZ   5000
#define LEDC_BITS      8  

volatile bool timerFlag = false;   
volatile bool estadoSistema  = true;    
volatile int  modo = 1;      
volatile unsigned long lastBtn1 = 0;
volatile unsigned long lastBtn2 = 0;
#define DEBOUNCE_US 200000UL      
volatile bool estadoLed2 = false;

hw_timer_t* timer = NULL;

void IRAM_ATTR onTimer() 
{
  timerFlag = true;
}
void IRAM_ATTR isrBtn1() {
  unsigned long ahora = micros();
  if (ahora - lastBtn1 > DEBOUNCE_US) 
  {
    lastBtn1 = ahora;
    if (estadoSistema) 
    {
      modo = (modo == 1) ? 2 : 1;
    }
  }
}
void IRAM_ATTR isrBtn2() {
  unsigned long ahora = micros();
  if (ahora - lastBtn2 > DEBOUNCE_US) 
  {
    lastBtn2 = ahora;
    estadoSistema = !estadoSistema;
  }
}

void configurarTimer(uint32_t freqHz) 
{
  if (timer != NULL) 
  {
    timerEnd(timer);
    timer = NULL;
  }

  timer = timerBegin(1000000);          

  timerAttachInterrupt(timer, &onTimer);
  uint64_t ticks = 1000000UL / freqHz;
  timerAlarm(timer, ticks, true, 0);    
}

void setup() 
{
  Serial.begin(115200);
  Serial.println("=== SISTEMA ESP32 INICIADO ===");

  pinMode(PIN_LED1, OUTPUT);
  pinMode(PIN_LED2, OUTPUT);
  pinMode(PIN_BTN1, INPUT_PULLUP);
  pinMode(PIN_BTN2, INPUT_PULLUP);
  ledcAttach(PIN_LED1, LEDC_FREQ_HZ, LEDC_BITS);
  attachInterrupt(digitalPinToInterrupt(PIN_BTN1), isrBtn1, FALLING);
  attachInterrupt(digitalPinToInterrupt(PIN_BTN2), isrBtn2, FALLING);
  configurarTimer(2);
  Serial.println("Hardware Timer configurado: 2 Hz");
  Serial.println("Modo inicial: 1 (Control PWM)");
}

void loop() 
{

  if (!estadoSistema) {
    ledcWrite(PIN_LED1, 0);     
    digitalWrite(PIN_LED2, LOW); 
    Serial.println("SISTEMA DESACTIVADO");

    while (!estadoSistema)
    {
  
    }
    Serial.println("=== SISTEMA REACTIVADO ===");
    return;
  }

  if (modo == 1) 
  {
    digitalWrite(PIN_LED2, LOW);

    int valorADC = analogRead(PIN_POT1);
    int valorPWM = map(valorADC, 0, 4095, 0, 255);

    ledcWrite(PIN_LED1, valorPWM);

    // Monitor Serial
    Serial.print("MODO 1 | ADC: ");
    Serial.print(valorADC);
    Serial.print("  |  PWM: ");
    Serial.println(valorPWM);
  }

  else if (modo == 2) 
  {

    ledcWrite(PIN_LED1, 0);

    int valorPot2 = analogRead(PIN_POT2);
    int freqParpadeo = map(valorPot2, 0, 4095, 20, 1);

    static int freqAnterior = 0;
    if (freqParpadeo != freqAnterior) 
    {
      configurarTimer(freqParpadeo);
      freqAnterior = freqParpadeo;
    }
    if (timerFlag) 
    {
      timerFlag = false;              
      estadoLed2 = !estadoLed2;          
      digitalWrite(PIN_LED2, estadoLed2 ? HIGH : LOW);

      // Monitor Serial
      Serial.print("MODO 2 | ADC: ");
      Serial.print(valorPot2);
      Serial.print("  |  Frecuencia: ");
      Serial.print(freqParpadeo);
      Serial.print(" Hz  |  LED2: ");
      Serial.println(estadoLed2 ? "ON" : "OFF");
    }
  }
}
