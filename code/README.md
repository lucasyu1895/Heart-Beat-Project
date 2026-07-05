#include <LiquidCrystal_PCF8574.h>
#include <Wire.h>
#include "MAX30105.h"
#include "heartRate.h"
 
MAX30105 particleSensor;

const byte RATE_SIZE = 12; //Increase this for more averaging. 4 is good.
byte rates[RATE_SIZE]; //Array of heart rates
byte rateSpot = 0;
long lastBeat = 0; //Time at which the last beat occurred

float beatsPerMinute;
int beatAvg;
LiquidCrystal_PCF8574 lcd(0x3F);  // 設定i2c位址，一般情況就是0x27和0x3F兩種
const int Red = 11;
const int Green = 10;
const int Blue = 9;


void setup()
{
  pinMode(Red, OUTPUT);
  pinMode(Green, OUTPUT);
  pinMode(Blue, OUTPUT);
  lcd.begin(16, 2);
  Serial.begin(115200);
  Serial.println("Initializing...");
  lcd.setBacklight(255);
  lcd.clear();
  lcd.setCursor(0, 0);  //設定游標位置 (字,行)
  // Initialize sensor
  if (!particleSensor.begin(Wire, I2C_SPEED_FAST)) //Use default I2C port, 400kHz speed
  {
    Serial.println("MAX30105 was not found. Please check wiring/power. ");
    while (1);
  }
  Serial.println("Place your index finger on the sensor with steady pressure.");

  particleSensor.setup(33,4,3,400,411,4096); //Configure sensor with default settings, first is LED, second is 4 times 1 data
 // particleSensor.setPulseAmplitudeRed(0x0A); //Turn Red LED to low to indicate sensor is running
  particleSensor.setPulseAmplitudeGreen(0); //Turn off Green LED

}

void loop()
{
  long irValue = particleSensor.getIR();

  if (checkForBeat(irValue) == true)
  {
    //We sensed a beat!
    long delta = millis() - lastBeat;
    lastBeat = millis();

    beatsPerMinute = 60 / (delta / 1000.0);

    if (beatsPerMinute < 255 && beatsPerMinute > 20)
    {
      rates[rateSpot++] = (byte)beatsPerMinute; //Store this reading in the array
      rateSpot %= RATE_SIZE; //Wrap variable

      //Take average of readings
      beatAvg = 0;
      for (byte x = 0 ; x < RATE_SIZE ; x++)
        beatAvg += rates[x];
      beatAvg /= RATE_SIZE;
    }
  }
 
  Serial.print("IR=");
  Serial.print(irValue);
  Serial.print(", BPM=");
  Serial.print(beatsPerMinute);
  Serial.print(", Avg BPM=");
  Serial.print(beatAvg);
  lcd.setCursor(0, 0);//設定游標位置 (字,行)
  lcd.print("BPM = ");

  if (irValue < 50000)
    {Serial.print(" No finger?");
    lcd.setCursor(0,0);
    lcd.print("No finger?");
    analogWrite(Blue,50);
    analogWrite(Red,0);
    analogWrite(Green,0);    
    }

if (irValue >= 50000)
{
  if (beatAvg <= 40)
  { lcd.setCursor(6,0);  lcd.print("  "); lcd.print("__");
    analogWrite(Blue, 50);
    analogWrite(Green,0);
    analogWrite(Red,0);    }
  if (beatAvg > 40 && beatAvg < 90)
  { analogWrite(Green, 50);
    analogWrite(Blue,0);
    analogWrite(Red,0);
    lcd.setCursor(6,0); lcd.print("  ");   lcd.print(beatAvg);    }

  if (beatAvg >= 90)
  { analogWrite(Red,50);
    analogWrite(Green, 0);
    analogWrite(Blue,0);
    lcd.setCursor(6,0); lcd.print(" ");  lcd.print(beatAvg);
    if (beatAvg < 100)
    {lcd.setCursor(6,0); lcd.print("  "); lcd.print(beatAvg);}
    if (beatAvg >= 100)
    {lcd.setCursor(6,0); lcd.print(" "); lcd.print(beatAvg);}     }
}

  Serial.println();

}

