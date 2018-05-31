---
layout: blog
title: 3d Printing Safety - Fire detection relay
date: '2018-05-30T22:19:19-04:00'
cover: /images/this-is-fine.0.jpg
categories:
  - 3d printing
  - safety
---
## 3d Printing Safety - Fire detection relay

I've been doing some 3d printing lately, and I have some thoughts to share about security and safety.

![This is fine](/images/this-is-fine.0.jpg)

<!--more-->

This is my 3d printer, isn't it beautiful?

![3d printer](/images/img_1357.jpg)

Here's something that isn't really highlighted when you buy a cheap chinese 3d printer. They aren't very safe. 

This is the printer I bought: 

https://www.aliexpress.com/item/Flsun-I3-3d-Printer-Auto-Leveling-Large-Printing-Size-300x300x420mm-DIY-3D-Printer-Kit-Heated-Bed/32874564199.html

A few prints in, this is what happened:

![null](/images/img_0755.jpg)

Ugh. I wasn't doing anything crazy here, but the amount of power routed through the motherboard for the heated bed and the extruder hot-end was too much, and the board caught fire.

Thankfully, I was in the other room and had a smoke detector right on top of the 3d printer. If I had left this unattended, it might have burned down my house.

Not. Cool.

So I fixed the fundamental design flaw by installing a mosfet to route the power for the heated by, bypassing the motherboard:

![](/images/img_1162.jpg)

And while I was at it, added a proper power switch to the power terminal block:

![](/images/img_1163.jpg)

These two upgrades greatly increase the safety of running the printer. I also updated the firmware and enabled some thermal runaway protection. Nevertheless, this thing could still catch fire.

What I needed was a way to kill the power on the printer if things went wrong. I couldn't find any commercially available devices that would do this.

There had been a crowdfunded device, but the project was abandoned.

I came across this URL http://mkme.org/forum/viewtopic.php?t=706 which was pretty close to what I wanted to do, so off I went to buy some parts.

Here's my part list:

[MQ2 smoke sensor](https://www.aliexpress.com/item/Free-Shipping-MQ-5-Methane-Natural-Gas-Sensor-Shield-Liquefied-Electronic-Detector-Module-New/32548466566.html) 0.98$

[DHT11 temperature sensor](https://www.aliexpress.com/item/New-Temperature-and-Relative-Humidity-Sensor-DHT11-Module-with-Cable-for-arduino-Diy-Kit/1941380773.html) 1$

[Arduino Nano](https://www.aliexpress.com/item/Freeshipping-1pcs-lot-Nano-3-0-controller-compatible-for-arduino-nano-CH340-USB-driver-NO-CABLE/32804787481.html) 1.79$

[Passive Buzzer Module KY-006](https://www.aliexpress.com/item/Compatible-ARDUNIO-Sensor-Branch-Yi-small-passive-buzzer-module-KY-006-For-Arduino/1826370256.html) 3$ for 10

Breadboard

[Black IEC C13 female Plug ](https://www.aliexpress.com/item/10pcs-New-Wholesale-Price-10A-250V-Black-IEC-C13-female-Plug-Rewirable-Power-Connector-3-pin/32824382215.html) (2) 6$ for 10

[Rocker switch](https://www.aliexpress.com/item/15A-250V-3pin-AC-power-socket-with-Power-Rocker-Switch-Fused/32812877104.html) 1$

[110 volt AC to 12 volt DC switching power supply](https://www.aliexpress.com/item/5-pcs-lot-400mA-4-8W-90V-240V-to-12V-AC-DC-Power-Buck-cenverters-Isolated/730947518.html) 11$ for 5

[Fotek solid state relay 40 amp](https://www.aliexpress.com/item/1pcs-SSR-40DA-40A-Solid-State-Relay-Module-3-32V-DC-Input-24-380VAC/32681597174.html) 3.40$

All-in, you're looking at about 25$ worth of parts to build this.

```
#include "DHT.h"
#define DHTPIN 2     // what pin we're connected to
#define BUZZERPIN 3 // Alarm buzzer
#define SMOKEPIN 4  // Smoke/gas detector
#define DHTTYPE DHT11   // DHT 11
#define THERMAL_OVERLOAD 40 // Alarm temp High Limit
#define RELAYPIN 13

DHT dht(DHTPIN, DHTTYPE);
// inspired by https://github.com/MKme/3D-Printer-Safety-Shutoff
//https://www.youtube.com/watch?v=LBr6AROebYA
//http://mkme.org/forum/viewtopic.php?f=2&t=706&p=949&hilit=relay#p949

void setup() {
  Serial.begin(9600);
  Serial.println("Initializing... waiting for gas detector to warm up");
  pinMode(BUZZERPIN,OUTPUT); 
  pinMode(SMOKEPIN, INPUT);
  pinMode(RELAYPIN,OUTPUT);

  // Right off the bat, turn off the relay...
  digitalWrite (RELAYPIN, LOW);

  delay(20000); // Need to wait 20 seconds before trying to read smoke sensor.

  Serial.println("Done");
  
  if (digitalRead(SMOKEPIN) == LOW) {
    Serial.println("ON FIRE!");
    alarm();
  }
  dht.begin();
}

void loop() {
  // Wait a few seconds between measurements.
  delay(1000);
  // Reading temperature or humidity takes about 250 milliseconds!
  // Sensor readings may also be up to 2 seconds 'old' (its a very slow sensor)
  float h = dht.readHumidity();
  // Read temperature as Celsius
  float t = dht.readTemperature();
 
  // Check if any reads failed and exit early (to try again).
  if (isnan(h) || isnan(t)) {
    Serial.println("Failed to read from DHT sensor!");
    // Turn off power then return
    alarm();
    return;
  }
 Serial.print("Temperature: ");
 Serial.print(t);
 Serial.print(" *C ");
 Serial.print(" Humidity: ");
 Serial.println(h);
 if (t >= THERMAL_OVERLOAD || digitalRead(SMOKEPIN) == LOW) {
  alarm();
} else { // All good
      digitalWrite(RELAYPIN, HIGH) ;
      Serial.println(" System OKay");
}

 

  }

void alarm() {
  while(1) {
  digitalWrite (RELAYPIN, LOW);
  Serial.println("FIRE!");
  sing(1);
  sing(1);
  sing(2);
  }
}


#define NOTE_B0  31
#define NOTE_C1  33
#define NOTE_CS1 35
#define NOTE_D1  37
#define NOTE_DS1 39
#define NOTE_E1  41
#define NOTE_F1  44
#define NOTE_FS1 46
#define NOTE_G1  49
#define NOTE_GS1 52
#define NOTE_A1  55
#define NOTE_AS1 58
#define NOTE_B1  62
#define NOTE_C2  65
#define NOTE_CS2 69
#define NOTE_D2  73
#define NOTE_DS2 78
#define NOTE_E2  82
#define NOTE_F2  87
#define NOTE_FS2 93
#define NOTE_G2  98
#define NOTE_GS2 104
#define NOTE_A2  110
#define NOTE_AS2 117
#define NOTE_B2  123
#define NOTE_C3  131
#define NOTE_CS3 139
#define NOTE_D3  147
#define NOTE_DS3 156
#define NOTE_E3  165
#define NOTE_F3  175
#define NOTE_FS3 185
#define NOTE_G3  196
#define NOTE_GS3 208
#define NOTE_A3  220
#define NOTE_AS3 233
#define NOTE_B3  247
#define NOTE_C4  262
#define NOTE_CS4 277
#define NOTE_D4  294
#define NOTE_DS4 311
#define NOTE_E4  330
#define NOTE_F4  349
#define NOTE_FS4 370
#define NOTE_G4  392
#define NOTE_GS4 415
#define NOTE_A4  440
#define NOTE_AS4 466
#define NOTE_B4  494
#define NOTE_C5  523
#define NOTE_CS5 554
#define NOTE_D5  587
#define NOTE_DS5 622
#define NOTE_E5  659
#define NOTE_F5  698
#define NOTE_FS5 740
#define NOTE_G5  784
#define NOTE_GS5 831
#define NOTE_A5  880
#define NOTE_AS5 932
#define NOTE_B5  988
#define NOTE_C6  1047
#define NOTE_CS6 1109
#define NOTE_D6  1175
#define NOTE_DS6 1245
#define NOTE_E6  1319
#define NOTE_F6  1397
#define NOTE_FS6 1480
#define NOTE_G6  1568
#define NOTE_GS6 1661
#define NOTE_A6  1760
#define NOTE_AS6 1865
#define NOTE_B6  1976
#define NOTE_C7  2093
#define NOTE_CS7 2217
#define NOTE_D7  2349
#define NOTE_DS7 2489
#define NOTE_E7  2637
#define NOTE_F7  2794
#define NOTE_FS7 2960
#define NOTE_G7  3136
#define NOTE_GS7 3322
#define NOTE_A7  3520
#define NOTE_AS7 3729
#define NOTE_B7  3951
#define NOTE_C8  4186
#define NOTE_CS8 4435
#define NOTE_D8  4699
#define NOTE_DS8 4978
 
#define melodyPin 3
//Mario main theme melody
int melody[] = {
  NOTE_E7, NOTE_E7, 0, NOTE_E7,
  0, NOTE_C7, NOTE_E7, 0,
  NOTE_G7, 0, 0,  0,
  NOTE_G6, 0, 0, 0,
 
  NOTE_C7, 0, 0, NOTE_G6,
  0, 0, NOTE_E6, 0,
  0, NOTE_A6, 0, NOTE_B6,
  0, NOTE_AS6, NOTE_A6, 0,
 
  NOTE_G6, NOTE_E7, NOTE_G7,
  NOTE_A7, 0, NOTE_F7, NOTE_G7,
  0, NOTE_E7, 0, NOTE_C7,
  NOTE_D7, NOTE_B6, 0, 0,
 
  NOTE_C7, 0, 0, NOTE_G6,
  0, 0, NOTE_E6, 0,
  0, NOTE_A6, 0, NOTE_B6,
  0, NOTE_AS6, NOTE_A6, 0,
 
  NOTE_G6, NOTE_E7, NOTE_G7,
  NOTE_A7, 0, NOTE_F7, NOTE_G7,
  0, NOTE_E7, 0, NOTE_C7,
  NOTE_D7, NOTE_B6, 0, 0
};
//Mario main them tempo
int tempo[] = {
  12, 12, 12, 12,
  12, 12, 12, 12,
  12, 12, 12, 12,
  12, 12, 12, 12,
 
  12, 12, 12, 12,
  12, 12, 12, 12,
  12, 12, 12, 12,
  12, 12, 12, 12,
 
  9, 9, 9,
  12, 12, 12, 12,
  12, 12, 12, 12,
  12, 12, 12, 12,
 
  12, 12, 12, 12,
  12, 12, 12, 12,
  12, 12, 12, 12,
  12, 12, 12, 12,
 
  9, 9, 9,
  12, 12, 12, 12,
  12, 12, 12, 12,
  12, 12, 12, 12,
};
//Underworld melody
int underworld_melody[] = {
  NOTE_C4, NOTE_C5, NOTE_A3, NOTE_A4,
  NOTE_AS3, NOTE_AS4, 0,
  0,
  NOTE_C4, NOTE_C5, NOTE_A3, NOTE_A4,
  NOTE_AS3, NOTE_AS4, 0,
  0,
  NOTE_F3, NOTE_F4, NOTE_D3, NOTE_D4,
  NOTE_DS3, NOTE_DS4, 0,
  0,
  NOTE_F3, NOTE_F4, NOTE_D3, NOTE_D4,
  NOTE_DS3, NOTE_DS4, 0,
  0, NOTE_DS4, NOTE_CS4, NOTE_D4,
  NOTE_CS4, NOTE_DS4,
  NOTE_DS4, NOTE_GS3,
  NOTE_G3, NOTE_CS4,
  NOTE_C4, NOTE_FS4, NOTE_F4, NOTE_E3, NOTE_AS4, NOTE_A4,
  NOTE_GS4, NOTE_DS4, NOTE_B3,
  NOTE_AS3, NOTE_A3, NOTE_GS3,
  0, 0, 0
};
//Underwolrd tempo
int underworld_tempo[] = {
  12, 12, 12, 12,
  12, 12, 6,
  3,
  12, 12, 12, 12,
  12, 12, 6,
  3,
  12, 12, 12, 12,
  12, 12, 6,
  3,
  12, 12, 12, 12,
  12, 12, 6,
  6, 18, 18, 18,
  6, 6,
  6, 6,
  6, 6,
  18, 18, 18, 18, 18, 18,
  10, 10, 10,
  10, 10, 10,
  3, 3, 3
};
 

int song = 0;
 
void sing(int s) {
  // iterate over the notes of the melody:
  song = s;
  if (song == 2) {
    Serial.println(" 'Underworld Theme'");
    int size = sizeof(underworld_melody) / sizeof(int);
    for (int thisNote = 0; thisNote < size; thisNote++) {
 
      // to calculate the note duration, take one second
      // divided by the note type.
      //e.g. quarter note = 1000 / 4, eighth note = 1000/8, etc.
      int noteDuration = 1000 / underworld_tempo[thisNote];
 
      buzz(BUZZERPIN, underworld_melody[thisNote], noteDuration);
 
      // to distinguish the notes, set a minimum time between them.
      // the note's duration + 30% seems to work well:
      int pauseBetweenNotes = noteDuration * 1.30;
      delay(pauseBetweenNotes);
 
      // stop the tone playing:
      buzz(BUZZERPIN, 0, noteDuration);
 
    }
 
  } else {
 
    Serial.println(" 'Mario Theme'");
    int size = sizeof(melody) / sizeof(int);
    for (int thisNote = 0; thisNote < size; thisNote++) {
 
      // to calculate the note duration, take one second
      // divided by the note type.
      //e.g. quarter note = 1000 / 4, eighth note = 1000/8, etc.
      int noteDuration = 1000 / tempo[thisNote];
 
      buzz(BUZZERPIN, melody[thisNote], noteDuration);
 
      // to distinguish the notes, set a minimum time between them.
      // the note's duration + 30% seems to work well:
      int pauseBetweenNotes = noteDuration * 1.30;
      delay(pauseBetweenNotes);
 
      // stop the tone playing:
      buzz(BUZZERPIN, 0, noteDuration);
 
    }
  }
}
 
void buzz(int targetPin, long frequency, long length) {
  long delayValue = 1000000 / frequency / 2; // calculate the delay value between transitions
  //// 1 second's worth of microseconds, divided by the frequency, then split in half since
  //// there are two phases to each cycle
  long numCycles = frequency * length / 1000; // calculate the number of cycles for proper timing
  //// multiply frequency, which is really cycles per second, by the number of seconds to
  //// get the total number of cycles to produce
  for (long i = 0; i < numCycles; i++) { // for the calculated length of time...
    digitalWrite(targetPin, HIGH); // write the buzzer pin high to push out the diaphram
    delayMicroseconds(delayValue); // wait for the calculated delay value
    digitalWrite(targetPin, LOW); // write the buzzer pin low to pull back the diaphram
    delayMicroseconds(delayValue); // wait again or the calculated delay value
  } 
}
```
