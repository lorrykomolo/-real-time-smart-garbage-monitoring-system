# -real-time-smart-garbage-monitoring-system
#define BLYNK_PRINT Serial
#include <TinyGPS++.h>
#include <SoftwareSerial.h>// The serial connection to the GPS module
#include <SimpleTimer.h>
#include <ESP8266WiFi.h>
#include <BlynkSimpleEsp8266.h>
#include "DHT.h"         
#include <SimpleTimer.h>  
#include<Servo.h>
#define DHTTYPE DHT11    
#define dht_dpin 14//D5
DHT dht(dht_dpin, DHTTYPE);
    Servo servo;
int trigPin = D3;  
int echoPin = D4; 
int const strigPin = D7;//servo pin for sensing  human
int const sechoPin = D6;
int sduration, sdistance;
long duration, cm,level,zero,ledl,full;
char auth[] = "ZBvATAY_GpA0a2DyyrbO8KGADrhffBx8"; // Blynk authentication key
char ssid[] = "Samsung"; // Name of your WiFi SSID
char pass[] = "09876543"; // WiFi Password
char server[] = "blynk-cloud.com";
float t;    
float h;
/*  virtual pins:
  # V3: MAP
  # V4: LCD screen in Advanced
*/
// Blynk LCD and map widgets
WidgetLCD lcd(V4);
WidgetMap myMap(V3);
String GPSLabel = "Garbage Bin"; //Labeling location on MAP
SimpleTimer timer;
static const int RXPin = D2, TXPin = D1;   // GPIO 4=D2(connect Tx of GPS) and GPIO 5=D1(Connect Rx of GPS)
static const uint32_t GPSBaud = 9600;

TinyGPSPlus gps;                    
SoftwareSerial ss(RXPin, TXPin);  
void setup() {
  pinMode(strigPin, OUTPUT); 
  pinMode(sechoPin, INPUT); 
    servo.attach(D8);
  Serial.begin(9600);       // serial connection for debugging
  ss.begin(GPSBaud);
  //Connect Blynk
  Blynk.begin(auth, ssid, pass, server, 8080);
      //Define inputs and outputs
  pinMode(trigPin, OUTPUT);
  pinMode(echoPin, INPUT);
  delay(1500);//Delay to let system boot
    dht.begin();
    timer.setInterval(2000, sendUptime);
    while (Blynk.connect() == false) {
    // Wait until connected
  }
 pinMode(14, INPUT); // gpio 14 input button 
  Serial.println("Activating GPS");
  timer.setInterval(1000L, periodicUpdate);
  timer.setInterval(60*1000, reconnectBlynk);
}
void sendUptime()
 {
  digitalWrite(trigPin, LOW);
  delayMicroseconds(10);
  digitalWrite(trigPin, HIGH);
  delayMicroseconds(10);
  digitalWrite(trigPin, LOW);
  // Read the signal from the sensor: a HIGH pulse whose
  // duration is the time (in microseconds) from the sending
  // of the ping to the reception of its echo off of an object.
  pinMode(echoPin, INPUT);
  duration = pulseIn(echoPin, HIGH);
  // Convert the time into a distance
  cm = (duration/2) / 29.1;     // Divide by 29.1 or multiply by 0.0343
  level=(100-(8+cm*(100/22)));
 
  //Serial.print(cm );
  //Serial.print ("cm ");
  //Serial.print ("\n");
  zero=0;
  full=100; 
  if (level<100){
    Serial.print("Current Trash Level = ");
   Serial.print(level );
   Serial.print (" % ");
  }
  if(level>=100){
    Serial.print("BIN FULL!");
   if(level<5)
     {Serial.print("BIN EMPTY");};}
  Serial.println();
  delay(1500);
  
  float h = dht.readHumidity();
  float t = dht.readTemperature(); 
  Serial.println("\n");
  Serial.print("Current humidity = ");
  Serial.print(h);
  Serial.print("%  ");
  Serial.print("temperature = ");
  Serial.print(t);
  Serial.println("C  ");
    
  if(t>27){
  Serial.print("POSSIBLE FIRE!");
  Serial.print("ACTION NEEDED!");
  Serial.print ("\n");  
  };
  
  if (level<100){
  Blynk.virtualWrite(V2, level);
     };
   if (level<=0){
  Blynk.virtualWrite(V2, zero);
     };
   if (level>=100){
  Blynk.virtualWrite(V2, full);
   }
   delay(1500);
  Blynk.virtualWrite(V0, t);
  Blynk.virtualWrite(V1, h);
  Blynk.virtualWrite(V5, ledl);
}

//Show GPS lat and lng on LCD
void periodicUpdate() {
  String line1, line2;
  //LCD
  lcd.clear();
  if (gps.location.isValid() && (gps.location.age() < 3000)) {
    //position current
    line1 = String("lat: ") + String(gps.location.lat(), 6);
    line2 = String("lng: ") + String(gps.location.lng(), 6);
    lcd.print(0, 0, line1);
    lcd.print(0, 1, line2);
    //update location on map
    myMap.location(2, gps.location.lat(), gps.location.lng(), GPSLabel);
  } else {
    //position is lost
    lcd.print(0, 0, "GPS Reconnecting");
  }
}

void updateGPS() {
  //read data from GPS module
  while (ss.available() > 0) {
    gps.encode(ss.read());
  }
}

void reconnectBlynk() {
  if (!Blynk.connected()) {
    Serial.println("Lost connection");
    if(Blynk.connect()) Serial.println("Reconnected");
    else Serial.println("Not reconnected");
  }
}
void loop() {
digitalWrite(strigPin, HIGH); 
delay(0);
digitalWrite(strigPin, LOW);
sduration = pulseIn(sechoPin, HIGH);
sdistance = (sduration/2) / 29.1;
 // if distance less than 0.5 meter and more than 0 (0 or less means over range) 
if (sdistance <= 50 && sdistance >= 0) {
  servo.write(0);
    delay(300);
} else {
  
  servo.write(80);
}
delay(60);
  timer.run();
  if(Blynk.connected()) { Blynk.run(); }
  updateGPS();
}
