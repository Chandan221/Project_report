Date: 10 March, 2022

## Problem 1 : Charging Circuit

Sir, in this problem i have checked the broken pad charging circuit part and come to find the Vcc (5V) pin of USB C (output) have a black spot and looks like its track is broken, and So, i tried to get the output directly from the IP5303 output pins and cleaned the short circuit part, and it works. This short circuit and broken pad mainly arisen due to the twisting of the connecting wires while opening and closing of the lid of the pedometer box as the track size near the usb c connector is thin (approx. 30-40 mils), so as to avoid this to happens in future now, i am trying to get the output directly from the IP5303 output pins, as there is even space for soldering near the output pads and it somewhat inward so there are less chances that the track or pad come out/broke due to opening and closing of lid while charging.

The images of the charging circuit module when came from Dr. Mohanty Sir is below :
![Before Changes](image\project_report\g1794.png)

The image of the charging circuit module after the changes :
![After changes](image\project_report\IMG_20220310_162252.jpg)

---

## Problem 2 : Generation of random files

Sir, now i am trying to use both internal rtc of esp32 of M5atom matrix, and external rtc DS3231 to checking for date and time. First, when the code runs, it check what was the date stored in the eeprom memory (by the internal rtc date before going to sleep), and checks if its matches external rtc date, and then checks is the data from external rtc has any significane (checkin for day, month and year) or not and if it has meaning and different from the stored value, this new value is being updated in eeprom and assign to internal rtc and now using time from this internal rtc the data are recorded as accelerometer detects any movement in certain (z axis) and before going back to sleep, the internal and external rtc date and time are being checked. to check for random date and time due to volatge drops at low battery.

The Code before making any changes is shown below:
```Arduino
// Including Important libraries
#include "M5Atom.h"         //M5Atom library for initialising M5Atom matrix, using its built-in accelerometer, and led matrix
#include <SPI.h>        //For communicating with SD Card  
#include "FS.h"
#include "SD.h"         //For SD card function usages 
#include <Wire.h>       //For I2C communication with external RTC
#include "RTClib.h"     //External RTC library to access and set time, date
#include <EEPROM.h>     //for using EEPROM of esp32
RTC_DS3231 rtc;        //rtc object to access RTC_DS3231 functions    
#define uS_TO_S_FACTOR 1000000  //Conversion factor for micro seconds to seconds 
#define TIME_TO_SLEEP 30
float ax, ay, az, px, py, pz;    //acceleration values of x, y, and z coordinates
int c_tot, force_stop=0;        //c_tot means count total steps in a day, forec_stop is when there is no movement for some second to turn off the modules
int mm, yy,dd, pd;          // mm for month number, yy is for year, dd for current day, pd is day stored in eeprom
int t=0;
/*
Functions for writing 2 bytes in EEPROM memory, and reading 2 bytes from EEPROM memory are written below
*/
void writeInt2byte(int address, int number)
{ 
  byte byte1 = number >> 8;
  byte byte2 = number & 0xFF;

  EEPROM.write(address, byte1);
  EEPROM.commit();
  EEPROM.write(address + 1, byte2);
  EEPROM.commit();
}
int readInt2byte(int address)
{
  byte byte1 = EEPROM.read(address);
  byte byte2 = EEPROM.read(address + 1);

  return (byte1 << 8) + byte2;
}
void setup(){
    M5.begin(true,false,true);
    Serial.begin(9600);
    EEPROM.begin(6);
    Wire.begin(32,26);
    rtc.begin();
    delay(1000);
    M5.dis.drawpix(0, 0x00f000); 
    // rtc.adjust(DateTime(F(__DATE__), F(__TIME__)));  for time update
    esp_sleep_enable_ext0_wakeup(GPIO_NUM_37, 1); //1 = High, 0 = Low
    SPI.begin(23,33,19,-1);
    M5.IMU.Init();  //Init IMU sensor.  
    if(!SD.begin(-1, SPI,   10000000)){
      Serial.println("Card Mount Failed");
        return;
    }
    uint8_t cardType = SD.cardType();
    if(cardType == CARD_NONE){
        Serial.println("No SD card attached");
        return;
    }
    Serial.print("SD Card Type: ");
    if(cardType == CARD_MMC){
        Serial.println("MMC");
    } 
    else if(cardType == CARD_SD){
        Serial.println("SDSC");
    } 
    else if(cardType == CARD_SDHC){
        Serial.println("SDHC");
    } 
    else {
        Serial.println("UNKNOWN");
    }
    uint64_t cardSize = SD.cardSize() / (1024 * 1024);
    Serial.printf("SD Card Size: %lluMB\n", cardSize);
    DateTime now = rtc.now();
    pd = EEPROM.read(0);
    if(pd == now.day()){
      Serial.println("same day ,starts from pevious count");
      c_tot = readInt2byte(1); 
      Serial.println(String(c_tot));
    }
    else{
      pd = now.day();
      Serial.println("New day, step count starts at 0");
      c_tot=0;
      writeInt2byte(1,c_tot);
      delay(100);
    }
    dd = now.day();
    EEPROM.write(0,dd);
    EEPROM.commit();
    Serial.print("Print eeprom date store : ");
    Serial.println(EEPROM.read(0));
    String fdate = String(now.day())+String(now.month())+String(now.year());
    String fdate2 = String(now.day())+"/"+String(now.month())+"/"+String(now.year());
    String ftime = String(now.hour())+":"+String(now.minute())+":"+String(now.second());
    esp_sleep_enable_timer_wakeup(TIME_TO_SLEEP * uS_TO_S_FACTOR);
    Serial.println("Date and Time : "+fdate2+","+ftime);
    M5.IMU.getAccelData(&px,&py,&pz);
    delay(500);
    File f_count = SD.open("/test_work_"+fdate+".csv");
      if(f_count){
        Serial.println("/test_work_date.csv present and opened");
        while(f_count.available()){
          Serial.write(f_count.read());
        }
        f_count.close();
      }
      else{
        Serial.println("file not present");
        writeInt2byte(1,0);
        c_tot =0;
        File mfile = SD.open("/test_work_"+fdate+".csv", FILE_WRITE);
        if(mfile){
          Serial.print(" file created and Writing for the first time today");
          mfile.println(String(0)+","+String(0)+","+String(0)+","+String(0)+","+String(0)+","+String(0));
          // close the file:
          mfile.close();
          delay(200);
          Serial.println("done.");
        }
        else{
          Serial.println("Failed to write ");
        }
      }
    while(1){
     delay(100);
     M5.IMU.getAccelData(&ax,&ay,&az);
     DateTime now = rtc.now();
     if(abs(pz-az)>0.25){
      Serial.printf("Previous values %.2f, %.2f, %.2f and new values %.2f, %.2f,  %.2f\n", px,py,pz,ax,ay,az);  
       px= ax;
       py=ay;
       pz=az;
       delay(200);
       c_tot +=1;
       t +=1;
       force_stop =0;
       String x = String(ax,3);
       String y = String(ay,3);
       String z = String(az,3);
       String cc = String(c_tot);
       String fdate = String(now.day())+String(now.month())+String(now.year());
       String fdate2 = String(now.day())+"/"+String(now.month())+"/"+String(now.year());
       String ftime = String(now.hour())+":"+String(now.minute())+":"+String(now.second());
       Serial.print("Datas shows :");
       Serial.println(fdate2+","+ftime+","+x+","+y+","+z+","+cc);
       Serial.printf("Value of c_tot is %d\n", c_tot);
       File myFile = SD.open("/test_work_"+fdate+".csv", FILE_APPEND);
       if (myFile) {
       Serial.print("Writing to test_work_date.csv...");
       myFile.println(fdate2+","+ftime+","+x+","+y+","+z+","+cc);
       // close the file:
       myFile.close();
       delay(200);
       Serial.println("done.");
        } 
        else {
          // if the file didn't open, print an error:
            Serial.println("error opening test.txt");
            }
      }
      else{
        px=ax;
        py=ay;
        pz=az;
        force_stop +=1;
        delay(100);
      }
     writeInt2byte(1,c_tot);
     int re = readInt2byte(1);
     Serial.printf("Value from EEPROM is %d and force_stop value is %d\n", re, force_stop );
     if(force_stop > 20){
      force_stop=0;
      break;
     }
    delay(200);
    }
    delay(100);
    Serial.println("Now M5 is going into sleep mode");
    M5.dis.drawpix(0, 0x000000); 
    esp_deep_sleep_start();
  
}
void loop(){

}
```

The code after making changes and using internal RTC of the M5atom's built-in esp32 is listed below.

```Arduino
#include "M5Atom.h"
#include <SPI.h>
#include "FS.h"
#include "SD.h"
#include <Wire.h>
#include "RTClib.h"
#include <EEPROM.h>
#include <ESP32Time.h>
RTC_DS3231 rtc;
ESP32Time rtc2;
#define uS_TO_S_FACTOR 1000000 /* Conversion factor for micro seconds to seconds */ 
#define TIME_TO_SLEEP 30
float ax, ay, az, px, py, pz;
int c_tot, force_stop=0;
int mm, yy, dd, pd, pm, py, ps, pmin, ph;
int t=0;
void writeInt2byte(int address, int number)
{ 
  byte byte1 = number >> 8;
  byte byte2 = number & 0xFF;

  EEPROM.write(address, byte1);
  EEPROM.commit();
  EEPROM.write(address + 1, byte2);
  EEPROM.commit();
}
int readInt2byte(int address)
{
  byte byte1 = EEPROM.read(address);
  byte byte2 = EEPROM.read(address + 1);

  return (byte1 << 8) + byte2;
}
void setup(){
    M5.begin(true,false,true);
    Serial.begin(9600);
    EEPROM.begin(6);
    Wire.begin(32,26);
    rtc.begin();
    delay(1000);
    M5.dis.drawpix(0, 0x00f000); 
   //rtc.adjust(DateTime(F(__DATE__), F(__TIME__)));
    esp_sleep_enable_ext0_wakeup(GPIO_NUM_37, 1); //1 = High, 0 = Low
    SPI.begin(23,33,19,-1);
    M5.IMU.Init();  //Init IMU sensor.  初始化姿态传感器
    if(!SD.begin(-1, SPI,   10000000)){
      Serial.println("Card Mount Failed");
        return;
    }
    uint8_t cardType = SD.cardType();
    if(cardType == CARD_NONE){
        Serial.println("No SD card attached");
        return;
    }
    Serial.print("SD Card Type: ");
    if(cardType == CARD_MMC){
        Serial.println("MMC");
    } else if(cardType == CARD_SD){
        Serial.println("SDSC");
    } else if(cardType == CARD_SDHC){
        Serial.println("SDHC");
    } else {
        Serial.println("UNKNOWN");
    }
    uint64_t cardSize = SD.cardSize() / (1024 * 1024);
    Serial.printf("SD Card Size: %lluMB\n", cardSize);
    DateTime now = rtc.now();
    if ((now.day() > 0 && now.day() <= 31) and (now.month()> 0 && now.month() <= 12) and (now.year() > 2021)){    // eliminating past date and time or random date and time
      Serial.println("Everything seems to be working");
      rtc2.setTime(now.second(), now.minute(), now.hour(), now.day(), now.month(), now.year());
    pd = EEPROM.read(0);
    if(pd == now.day()){
      Serial.println("same day ,starts from pevious count");
      c_tot = readInt2byte(1); 
      Serial.println(String(c_tot));
    }
    else{
      pd = now.day();
      Serial.println("New day, step count starts at 0");
      c_tot=0;
      writeInt2byte(1,c_tot);
      delay(100);
    }
    dd = rtc2.getDay();
    mm = rtc2.getMonth();
    yy = rtc2.getYear();
    EEPROM.write(0,dd);
    EEPROM.write(8,mm);
    writeInt2byte(10,yy);
    EEPROM.commit();
    Serial.print("Print eeprom date store : ");
    Serial.println(EEPROM.read(0));
    String fdate = String(rtc2.getDay())+String(rtc2.getMonth())+String(rtc2.getYear());
    String fdate2 = rtc2.getDate();
    String ftime = rtc2.getTime();
    esp_sleep_enable_timer_wakeup(TIME_TO_SLEEP * uS_TO_S_FACTOR);
    //Serial.println("Date and Time : "+fdate2+","+ftime);
    M5.IMU.getAccelData(&px,&py,&pz);
    delay(500);
    File f_count = SD.open("/test_work_"+fdate+".csv");
      if(f_count){
       // Serial.println("/test_work_date.csv present");
     /*   while(f_count.available()){
          Serial.write(f_count.read());
        }
        f_count.close(); */
      }
      else{
        Serial.println("file not present");
        writeInt2byte(1,0);
        c_tot =0;
        File mfile = SD.open("/test_work_"+fdate+".csv", FILE_WRITE);
        if(mfile){
        //  Serial.print(" file created and Writing for the first time today");
          mfile.println(String(0)+","+String(0)+","+String(0)+","+String(0)+","+String(0)+","+String(0));
    // close the file:
      mfile.close();
          delay(200);
      //Serial.println("done.");
        }
        else{
          //Serial.println("Failed to write ");
        }
      }
   while(1){
    delay(100);
    M5.IMU.getAccelData(&ax,&ay,&az);
    DateTime now = rtc.now();
    if(abs(pz-az)>0.25){
     //Serial.printf("Previous values %.2f, %.2f, %.2f and new values %.2f, %.2f, %.2f\n", px,py,pz,ax,ay,az);
       px= ax;
       py=ay;
       pz=az;
       delay(200);
       c_tot +=1;
       t +=1;
       force_stop =0;
       String x = String(ax,3);
       String y = String(ay,3);
       String z = String(az,3);
       String cc = String(c_tot);
       String ftime = rtc2.getTime();
       //Serial.print("Datas shows :");
       //Serial.println(fdate2+","+ftime+","+x+","+y+","+z+","+cc);
       //Serial.printf("Value of c_tot is %d\n", c_tot);
       File myFile = SD.open("/test_work_"+fdate+".csv", FILE_APPEND);
       if (myFile) {
       //Serial.print("Writing to test_work_date.csv ...");
       myFile.println(fdate2+","+ftime+","+x+","+y+","+z+","+cc);
       // close the file:
       myFile.close();
       delay(200);
       //Serial.println("done.");
        } else {
          // if the file didn't open, print an error:
          // Serial.println("error opening test_work_date.csv ...");
    }
      }
      else{
        px=ax;
        py=ay;
        pz=az;
        force_stop +=1;
        delay(100);
      }
     writeInt2byte(1,c_tot);
     //int re = readInt2byte(1);
     //Serial.printf("Value from EEPROM is %d and force_stop value is %d\n", re, force_stop );
     if(force_stop > 20){
      force_stop=0;
      break;
     }
    delay(200);
    }
    delay(100);
    EEPROM.write(12, rtc2.getHour(true));
    EEPROM.write(13, rtc2.getMinute());
    EEPROM.write(14, rtc2.getSecond());
    EEPROM.commit();
    }
    else {
      Serial.println("Some rando date and time. So, loading the time from EEPROM Date and Time");
      pm = EEPROM.read(8);
      py = readInt2byte(10);
      ph = EEPROM.read(12);
      pmin = EEPROM.read(13);
      ps = EEPROM.read(14);
      rtc.adjust(DateTime(py, pm, pd, ph, pmin, ps));   //Resetting External rtc clock w
    }
    Serial.println("Now M5 is going into sleep mode");
    M5.dis.drawpix(0, 0x000000); 
    esp_deep_sleep_start();
}
void loop(){

}

```
Sir, in this way when, there is a drift in external clock, we can reset using the last saved date and time and files are being generated.