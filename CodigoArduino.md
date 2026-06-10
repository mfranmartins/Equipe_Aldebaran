//Código utilizado no arduino

#include <Adafruit_LiquidCrystal.h> // Inclusão de biblioteca
Adafruit_LiquidCrystal lcd(0);

// Pinos dos Sensores
const int pinTMP = A0;          // Temperatura
const int pinPOT_Energia = A1;  // Bateria
const int pinLDR = A2;          // Luminosidade
const int pinPOT_Pressao = A3;  // Pressão
const int pinTILT_Estabil = 9;  // Tilt Switch
const int pinC_Retorno = 8;     // Retorno do CI NOT

// Pinos de Saída para os CIs e LEDs
const int PIN_A = 2;   
const int PIN_B = 3;   
const int PIN_D = 5;   
const int PIN_E = 6;   
const int PIN_IA = 7;  

// Variáveis para controle do efeito piscar do LCD
unsigned long tempoAnteriorBlink = 0;
bool estadoBlink = false; 
const long intervaloBlink = 100; 

void setup() {
  Serial.begin(9600);
  
  pinMode(pinTILT_Estabil, INPUT_PULLUP); 
  pinMode(pinC_Retorno, INPUT); 
  pinMode(PIN_A, OUTPUT);
  pinMode(PIN_B, OUTPUT);
  pinMode(PIN_D, OUTPUT);
  pinMode(PIN_E, OUTPUT); 
  pinMode(PIN_IA, OUTPUT);
  
  lcd.begin(16, 2);
  lcd.setBacklight(1);
  lcd.setCursor(0, 0);
  lcd.print("Mission Control");
  lcd.setCursor(2, 1);
  lcd.print("Alert System");
  delay(2000);
  lcd.clear();
}

void loop() {
  //Leitura dos sensores em grandezas reais
  float tempC = ((analogRead(pinTMP) * (5.0 / 1023.0)) - 0.5) * 100.0;
  int bateria = map(analogRead(pinPOT_Energia), 0, 1023, 0, 100);
  int pressao = map(analogRead(pinPOT_Pressao), 0, 1023, 0, 100);
  int luzInterna = map(analogRead(pinLDR), 1, 169, 0, 100);
  if (luzInterna < 0) luzInterna = 0;
  if (luzInterna > 100) luzInterna = 100;

  bool inclinou = digitalRead(pinTILT_Estabil) == LOW; 
  bool C = digitalRead(pinC_Retorno) == HIGH; 
  bool A = (bateria < 20);              
  bool B = (tempC > 45.0);              
  bool D = (pressao < 30);              
  bool E = inclinou;                    

  //Análise da IA (zonas de atenção)
  bool iaAtencaoBateria = (bateria >= 20 && bateria <= 35);
  bool iaAtencaoTemp    = (tempC > 38.0 && tempC <= 45.0);
  bool iaAtencaoPressao = (pressao >= 30 && pressao <= 45);
  bool alertaIA = iaAtencaoBateria || iaAtencaoTemp || iaAtencaoPressao;
  digitalWrite(PIN_IA, alertaIA ? HIGH : LOW);

  //Gerenciador do tempo do blink (LCD)
  unsigned long tempoAtual = millis();
  if (tempoAtual - tempoAnteriorBlink >= intervaloBlink) {
    tempoAnteriorBlink = tempoAtual;
    estadoBlink = !estadoBlink; 
  }

  //Controle dos leds
  digitalWrite(PIN_A, A ? HIGH : LOW);
  digitalWrite(PIN_B, B ? HIGH : LOW);
  digitalWrite(PIN_D, D ? HIGH : LOW);
  digitalWrite(PIN_E, E ? HIGH : LOW);

  //Cálculo da equação booleana
  bool X_calc = (A && E) || (D && B) || (B && C) || (C && !A);

  //Diagnóstico no monitor serial
  Serial.print("Telemetry -> B: "); Serial.print(bateria); Serial.print("% | ");
  Serial.print("T: "); Serial.print(tempC, 0); Serial.print("C | ");
  Serial.print("C: "); Serial.print(C ? "1" : "0"); Serial.print(" | ");
  Serial.print("P: "); Serial.print(pressao); Serial.print("% | ");
  Serial.print("V: "); Serial.print(E ? "1" : "0"); Serial.print(" | ");
  Serial.print("L: "); Serial.print(luzInterna); Serial.print("%"); 
  
  if (X_calc)        Serial.println(" -> [EMERGENCY: HARDWARE CRITICAL]");
  else if (alertaIA) Serial.println(" -> [WARNING: AI DETECTED POTENTIAL RISK]");
  else               Serial.println(" -> [STATUS: OK]");

  //Exibição no LCD
  lcd.setCursor(0, 0); lcd.print("B:");
  lcd.setCursor(7, 0); lcd.print("T:");
  lcd.setCursor(13, 0); lcd.print("C:");
  lcd.setCursor(2, 0);
  if ((iaAtencaoBateria || A) && estadoBlink) {
    lcd.print("    "); 
  } else {
    lcd.print(bateria); lcd.print("%");
    if (bateria < 100) lcd.print(" "); 
    if (bateria < 10)  lcd.print(" ");
  }
  lcd.setCursor(9, 0);
  if ((iaAtencaoTemp || B) && estadoBlink) {
    lcd.print("   "); 
  } else {
    lcd.print((int)tempC); lcd.print("C");
    if (tempC >= 0 && tempC < 10) lcd.print(" ");
    if (tempC < 0 && tempC > -10) lcd.print(" "); 
  }
  lcd.setCursor(15, 0);
  if (C && estadoBlink) {
    lcd.print(" ");
  } else {
    lcd.print(C ? "1" : "0");
  }
  lcd.setCursor(0, 1);  lcd.print("P:");
  lcd.setCursor(7, 1);  lcd.print("V:");
  lcd.setCursor(11, 1); lcd.print("L:");
  lcd.setCursor(2, 1);
  if ((iaAtencaoPressao || D) && estadoBlink) {
    lcd.print("    "); 
  } else {
    lcd.print(pressao); lcd.print("%");
    if (pressao < 100) lcd.print(" ");
    if (pressao < 10)  lcd.print(" ");
  }
  lcd.setCursor(9, 1);
  if (E && estadoBlink) {
    lcd.print(" ");
  } else {
    lcd.print(E ? "1" : "0");
  }
  lcd.setCursor(13, 1);
  lcd.print(luzInterna); lcd.print("%");
  if (luzInterna < 100) lcd.print(" ");
  if (luzInterna < 10)  lcd.print(" ");
  delay(20); 
}
