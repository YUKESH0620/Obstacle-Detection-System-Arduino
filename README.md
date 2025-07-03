# Obstacle Detection System – Arduino Project 🚧

This project is a basic yet fully functional **Obstacle Detection System** developed as the **final project** of an Arduino course on **Udemy**. It demonstrates real-time sensing, user interaction, persistent settings, and modular embedded programming using **C++**.

## 🎯 Features

- 🔍 **Ultrasonic Sensor** to detect obstacles and measure real-time distance
- 📟 **LCD Display** with multi-screen navigation (distance, settings, light level)
- 🎮 **IR Remote Control** to switch units, reset settings, and navigate modes
- 💾 **EEPROM** to store user preferences even after reboot
- 🌗 **Photoresistor (LDR)** to auto-adjust LED brightness based on ambient light
- 🚨 **Warning & Error LEDs** based on proximity logic
- 🎚️ **Potentiometer** to adjust LCD contrast
- 🔘 **Push Button** for user input

## 🧠 What I Learned

- Writing non-blocking logic using `millis()` (no `delay()` used)
- Using **interrupts** for precise ultrasonic readings
- Handling **IR remote commands** and user inputs
- Managing **persistent memory** with EEPROM
- Designing intuitive user interfaces on simple LCDs
- Structuring embedded code into **clean, scalable modules**

## 🔧 Components Used

| Component            | Description                     |
|----------------------|---------------------------------|
| Arduino Uno          | Microcontroller Board           |
| Ultrasonic Sensor    | HC-SR04                         |
| LCD Display          | 16x2 with I2C or parallel pins  |
| IR Receiver & Remote | For input navigation            |
| EEPROM (internal)    | Data persistence                |
| LDR                  | Light sensing                   |
| Potentiometer        | LCD contrast control            |
| LEDs (Warning/Error) | Visual feedback                 |
| Push Button          | Additional input                |

## 🔗 Demo Video & Certificate

A short demo video will be uploaded soon.  
The **course certificate** (Udemy) is used as the **thumbnail** in the video.

## 🤝 Acknowledgements

Special thanks to my **Udemy instructor** for providing a clear, hands-on learning experience that inspired this project.

## 📌 Future Improvements

- Motor driver for robotic movement
- Buzzer alerts
- Wi-Fi (IoT) integration
- App or web interface

## 📜 License

This project is open-source under the MIT License.

## My Arduino Code
```cpp
#include <LiquidCrystal.h>
#include<IRremote.h>
#include<EEPROM.h>
//Ultrasonic Sensor
#define ECHO_PIN 3
#define TRIGGER_PIN 4
//Ir Receiver
#define IR_RECEIVER_PIN 5
//LEDs
#define WARNING_LED 11
#define ERROR_LED 12
#define LUMINOUS_LED 10
#define LOCK_DISTANCE 10.00
#define WARNING_DISTANCE 60.00
//button
#define BUTTON_PIN 2
//lcd
#define LCD_RS_PIN A5
#define LCD_E_PIN  A4
#define LCD_D4_PIN 6
#define LCD_D5_PIN 7
#define LCD_D6_PIN 8
#define LCD_D7_PIN 9
//Ir Button Mapping
#define IR_BUTTON_PLAY  1
#define IR_BUTTON_OFF  18
#define IR_BUTTON_EQ    4
#define IR_BUTTON_RIGHT 3
#define IR_BUTTON_LEFT  2
#define DIST_IN_CM 0
#define DIST_IN_INCHES 1
#define CM_TO_INCHES 0.393701
#define EEPROM_ADDRESS_DIST_UNIT 80
#define PHOTORESISTOR_PIN A0
//Lcd Mode
#define LCD_MODE_DISTANCE 0
#define LCD_MODE_SETTINGS 1
#define LCD_MODE_LUMINOSITY 2

unsigned long LastTimeUltrasonicTrigger = millis();
unsigned long UltrasonicDelay = 60;
volatile bool newDistanceAvailable = false ;

volatile long PulseInBegin;
volatile long PulseInEnd;
unsigned int PreviousDistance = 380;

unsigned long LastTimeWarningLedBlinked = millis();
unsigned long BlinkDelay = 500;
byte WarningLedState = LOW;
unsigned long LastTimeErrorLedBlinked = millis();
unsigned long ErrorBlinkDelay = 500;
byte ErrorLedState = LOW;

unsigned long LastTimeButtonPressed = millis();
unsigned long DebounceDelay = 50;
byte ButtonState;

unsigned long LastTimeReadLuminosity = millis();
unsigned long ReadingDelay = 100;

bool isLocked = false;
int DistanceUnit = DIST_IN_CM;
int LcdMode = LCD_MODE_DISTANCE;
LiquidCrystal lcd( LCD_RS_PIN ,LCD_E_PIN ,LCD_D4_PIN ,LCD_D5_PIN ,LCD_D6_PIN ,LCD_D7_PIN );

void triggerUltrasonicSensor()
{
  digitalWrite(TRIGGER_PIN, LOW);
  delayMicroseconds(2);
  digitalWrite(TRIGGER_PIN,HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIGGER_PIN,LOW);
}

void UltrasonicInterrupt()
{
   if(digitalRead(ECHO_PIN) == HIGH){
   PulseInBegin = micros();
   }
   else{
    PulseInEnd = micros();
    newDistanceAvailable = true;
   }
}

double UltrasonicDistance()
{
  double TimeDurationMicros = PulseInEnd - PulseInBegin ;
  double Distance = TimeDurationMicros / 58.00;
  if(Distance > 300.0){
    return PreviousDistance;
  }
  Distance = PreviousDistance *0.8 + Distance *0.2 ;
  PreviousDistance = Distance;
  return Distance;
}

void toggleWarningLed(){
  WarningLedState =(WarningLedState == HIGH )? LOW: HIGH ;
  digitalWrite(WARNING_LED, WarningLedState);
}

void toggleErrorLed(){
  ErrorLedState = (ErrorLedState == HIGH )? LOW: HIGH ;
  digitalWrite(ERROR_LED, ErrorLedState);
}

void SetWarningLedBlinkRateFromDistance(double distance)
{
  BlinkDelay = distance*4 ;
  //Serial.println(BlinkDelay); 
}

void lock(){
  if(!isLocked){
    isLocked = true;
    WarningLedState = LOW ;
    ErrorLedState = LOW ;
  }
}

void Unlock()
{
  if(isLocked){
    isLocked = false;
    ErrorLedState = LOW ;
    digitalWrite(ERROR_LED, ErrorLedState);
    lcd.clear();
  }
}

void PrintDistOnLed(double distance)
{
  if(isLocked){
     lcd.setCursor(0,0);
     lcd.print("!!! Obstacle !!!   ");
     lcd.setCursor(0,1);
     lcd.print("Press to Unlock. ");
  }else if(LcdMode ==LCD_MODE_DISTANCE){
    lcd.setCursor(0,0);
    lcd.print("Dist:");
    if(DistanceUnit == DIST_IN_INCHES){
       lcd.print(distance * CM_TO_INCHES);
       lcd.print("in      ");
    }else{
        lcd.print(distance);
        lcd.print("cm        ");
    }
    lcd.setCursor(0,1);
    if(distance > WARNING_DISTANCE){
       lcd.print("No Obstacle.            ");
    }else{
       lcd.print(" !! Warning !!   ");
    }
  }
}

void toggleDistanceUnit()
{
  if(DistanceUnit == DIST_IN_CM){
    DistanceUnit = DIST_IN_INCHES;
  }else{
    DistanceUnit = DIST_IN_CM;
  }
  EEPROM.write(EEPROM_ADDRESS_DIST_UNIT, DistanceUnit); 
}

void toggleLcdScreen(bool Next)
{
  switch(LcdMode) {
    case LCD_MODE_DISTANCE : {
       LcdMode = (Next)? LCD_MODE_SETTINGS: LCD_MODE_LUMINOSITY;
       break;
    }
    case LCD_MODE_SETTINGS : {
      LcdMode = (Next)? LCD_MODE_LUMINOSITY: LCD_MODE_DISTANCE;
    }
    case LCD_MODE_LUMINOSITY : {
      LcdMode = (Next)? LCD_MODE_DISTANCE: LCD_MODE_SETTINGS;
    }
    default : {
      LcdMode = LCD_MODE_DISTANCE;
    }
  }

  lcd.clear();
  if(LcdMode == LCD_MODE_SETTINGS){
    lcd.setCursor(0,0);
    lcd.print("Press on OFF to :    ");
    lcd.setCursor(0,1);
    lcd.print("reset Settings.     ");
  }
}

void resetSettingdToDefault()
{
  if(LcdMode == LCD_MODE_SETTINGS){
     DistanceUnit = DIST_IN_CM;
     EEPROM.write(EEPROM_ADDRESS_DIST_UNIT, DistanceUnit);
     lcd.clear();
     lcd.setCursor(0,0);
     lcd.print("Settings Have  ");
     lcd.setCursor(0,1);
     lcd.print("Been Reset.    ");
  }
}

void setLuminosityLedBrightness(int Luminosity)
{
  byte Brightness = 255 - Luminosity / 4;
  analogWrite(LUMINOUS_LED, Brightness);
}

void printLuminosityOnLcd(int Luminosity)
{
  if( !isLocked && LcdMode==LCD_MODE_LUMINOSITY ){
    lcd.clear();
     lcd.setCursor(0,0);
     lcd.print("Luminosity: ");
     lcd.print(Luminosity);
     lcd.print("         ");
  }
}

void handleIrRemote(long command)
{
   switch (command) {
      case IR_BUTTON_PLAY : {
        Unlock();
        break;
      }
      case IR_BUTTON_OFF : {
        resetSettingdToDefault();
        break;
      }
      case IR_BUTTON_EQ : {
        toggleDistanceUnit();
        break;
      }
      case IR_BUTTON_RIGHT : {
        toggleLcdScreen(true);
        break;
      }
      case IR_BUTTON_LEFT : {
        toggleLcdScreen(false);
        break;
      }
      default: {
      }
   }
}

void setup() {
  Serial.begin(115200);
  lcd.begin(16,2);
  IrReceiver.begin(IR_RECEIVER_PIN);
  pinMode(ECHO_PIN, INPUT);
  pinMode(TRIGGER_PIN, OUTPUT);
  attachInterrupt(digitalPinToInterrupt(ECHO_PIN), UltrasonicInterrupt,CHANGE);
  pinMode(WARNING_LED, OUTPUT);
  pinMode(ERROR_LED, OUTPUT);
  pinMode(LUMINOUS_LED, OUTPUT);
  pinMode(BUTTON_PIN,INPUT);

  ButtonState = digitalRead(BUTTON_PIN);
  lcd.print("Initializing...");
  delay(1000);
  lcd.clear();

  DistanceUnit = EEPROM.read(EEPROM_ADDRESS_DIST_UNIT);
  if(DistanceUnit == 255){
    DistanceUnit = DIST_IN_CM;
  }
}

void loop() {
  unsigned long Timenow = millis();

  if(IrReceiver.decode()){
    IrReceiver.resume();
    long command = IrReceiver.decodedIRData.command;
    handleIrRemote(command);
    Serial.println(command);
  }

  if(Timenow - LastTimeReadLuminosity > ReadingDelay){
    LastTimeReadLuminosity += ReadingDelay;
    int Luminosity = analogRead(PHOTORESISTOR_PIN);
    setLuminosityLedBrightness(Luminosity);
    printLuminosityOnLcd(Luminosity);
  }

  if(Timenow - LastTimeUltrasonicTrigger > UltrasonicDelay)
  {
    LastTimeUltrasonicTrigger += UltrasonicDelay;
    triggerUltrasonicSensor();
  }
  if(isLocked){
    if(Timenow - LastTimeErrorLedBlinked > ErrorBlinkDelay)
      {
        LastTimeErrorLedBlinked += ErrorBlinkDelay;
        toggleWarningLed();
        toggleErrorLed(); 
      }
    if(Timenow - LastTimeButtonPressed > DebounceDelay)
      {
        byte NewButtonState = digitalRead(BUTTON_PIN);
        if(NewButtonState != ButtonState)
        {
          LastTimeButtonPressed = Timenow;
          ButtonState = NewButtonState;
          if(ButtonState == LOW){
            Unlock();
          }
        }
      }
  }else{
     if(Timenow - LastTimeWarningLedBlinked > BlinkDelay)
      {
        LastTimeWarningLedBlinked += BlinkDelay;
        toggleWarningLed();
      }
  }

  if(newDistanceAvailable){
    newDistanceAvailable = false;
    double distance = UltrasonicDistance();
    SetWarningLedBlinkRateFromDistance(distance);
    PrintDistOnLed(distance);
    if(distance < LOCK_DISTANCE)
    {
      lock();
    }
  }
}
```

*Built with curiosity, passion, and a mindset for systems thinking.*

