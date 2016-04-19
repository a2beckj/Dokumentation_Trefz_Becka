<img src="https://github.com/sensebox/OER/blob/master/senseBox_edu/images/sensebox_logo_neu.png" width="200"/>


## Projekt 'Messstation am Münsteraner Hafen'
Unsere senseBox soll Temperatur-, Feuchtigkeits-, Feinstaub- und UV-Werte messen und im 10-Sekunden Takt auf die OSM-Plattform übertragen.
 
### Ziel
Durch Erfassen der verschiedenen Werte sollen mögliche Korrelationen zwischen den Naturerscheinungen festgestellt werden. Auf Basis der Werte soll im Anschluss eine  Karte erstellt werden, um die zuvor erhobenen Messwerte besser darstellen zu können.
 
### Materialien
#### Aus der senseBox:edu
*   Genuino UNO
*   Ethernet Shield v1.0
*   senseBox-Shield
*    Temperatur- und Luftfeuchtigkeitssensor
*    UV-Sensor
*    Jumpwires
 
#### Zusätzliche Hardware
*    Base Shield v2.0
*    Feinstaubsensor
*    Verbindungskabel
*    Netzwerkkabel
 
 
## Setup Beschreibung
#### Hardwarekonfiguration
 
* Ethernet Shield v1.0 auf das Genuino UNO stecken.
* Base Shield v2.0 auf das Ethernet Shield v1.0 stecken.
* Anschließen von Temperatur-/bzw. Luftfeuchtigkeits- und UV-Sensor über das Genunio Board an das Base Shield v2.0.
* Anschließen des Feinstaubsensors an Port D8 des Base Shield v2.0.
* Netzwerkkabel an Ethernet Shield v1.0 anschliessen und mit dem Router(sowie der Stromzufuhr) verbinden.
 
 
#### Softwaresketch
<image src="https://raw.githubusercontent.com/a2beckj/images/49ce4b9b83c80e7c470e2b88ae5b82810c5653b9/Untitled%20Sketch%204_Steckplatin_v2.jpg"/>
 
 
 
Zunächst sind folgende Bibliotheken herunterzuladen und in die Arduino-Dateistruktur zu integrieren:
* https://github.com/RFgermany/HDC100X_Arduino_Library
 
* https://github.com/watterott/Arduino-Libs
 
Diese Bibliotheken müssen anschließend im Programmcode wie folgt implementiert werden:
``` c
#include <SPI.h>
#include <Ethernet.h>
#include <Wire.h>  
#include <HDC100X.h>
```
Hier sieht man den Programmcode mit den ID's, die uns von der opensensemap nach Einrichtung des Messstandortes überliefert wurden:
(Die individuell überlieferte Sensor-ID wurde im Folgenden durch 'SENSORID' ersetzt)
 
```
//SenseBox ID
#define SENSEBOX_ID "SENSEBOXID"
 
//Sensor IDs
#define TEMPSENSOR_ID "SENSORID" //Temperatur
#define SENSOR1_ID "SENSORID" // Luftfeuchtigkeit
#define UVSENSOR_ID "SENSORID" // UV-Strahlung
#define SENSOR2_ID "SENSORID" // Feinstaub
 
//Ethernet-Parameter
char server[] = "www.opensensemap.org";
byte mac[] = { 0xDE, 0xAD, 0xBE, 0xEF, 0xFE, 0xED };
// Diese IP Adresse nutzen falls DHCP nicht möglich
IPAddress myIP(192, 168, 0, 42);
EthernetClient client;
 
//Messparameter
int postInterval = 10000; //Uploadintervall in Millisekunden
long oldTime = 0;
```
Die Deklaration der Variablen der verschiedenen Sensoren:
 
```
//Temperatursensor
float Humi = 0;
float Temp = 0;
HDC100X HDC(0x43);
 
//UV Sensor
#define I2C_ADDR 0x38
#define IT_1_2 0x0 //1/2T
#define IT_1   0x1 //1T
#define IT_2   0x2 //2T
#define IT_4   0x3 //4T
 
//Feinstaubsensor
int pin = 8;
unsigned long duration;
unsigned long starttime;
unsigned long sampletime_ms = 2000;
unsigned long lowpulseoccupancy = 0;
float ratio = 0;
float concentration = 0;
```
 
Im setup werden die verschiedenen Sensoren sowie Bibliotheken und Anschlüsse und Verbindungen (z.B. Ethernet) initialisiert:
```
void setup()
{
  Serial.begin(9600);
````
Hier erfolgt die Initialisierung der eingesetzen Sensoren:
````
  //Temperatursensor
  HDC.begin(HDC100X_TEMP_HUMI, HDC100X_14BIT, HDC100X_14BIT, DISABLE);
 
  //UV Sensor
  Wire.begin();
  Wire.beginTransmission(I2C_ADDR);
  Wire.write((IT_1<<2) | 0x02);
  Wire.endTransmission();
  delay(500);
 
  //Feinstaubsensor
  pinMode(8,INPUT);
````
Der von OSM überlieferte Code um die Ethernet-Verbindung zu initialisieren:
````
  Serial.print("Starting network...");
  //Ethernet Verbindung mit DHCP ausführen..
  if (Ethernet.begin(mac) == 0)
  {
    Serial.println("DHCP failed!");
    //Falls DHCP fehltschlägt, mit manueller IP versuchen
    Ethernet.begin(mac, myIP);
  }
  Serial.println("done!");
  delay(1000);
  Serial.println("Starting loop.");
 
 
  starttime = millis();//get the current time;
}
`````
Nach der Initialisierung startet der loop mit der Berechnung der verschiedenen Messwerte der Sensoren: (Eränzt um die postFloatValue-Funktion welche die Messwerte mit der OSM-Plattform synchronisiert)
```
void loop()
{
  //Upload der Daten mit konstanter Frequenz
  if (millis() - oldTime >= postInterval)
  {
    oldTime = millis();
 
  //Temperatur/Feuchtigkeit loop
  Humi = HDC.getHumi();
  postFloatValue(Humi, 1, SENSOR1_ID);
  Temp = HDC.getTemp();
  postFloatValue(Temp, 1, TEMPSENSOR_ID);
 
   //UV Sensor loop
  byte msb=0, lsb=0;
  uint16_t uv;
  Wire.requestFrom(I2C_ADDR+1, 1); //MSB
  delay(1);
  if(Wire.available())
    msb = Wire.read();
  Wire.requestFrom(I2C_ADDR+0, 1); //LSB
  delay(1);
  if(Wire.available())
    lsb = Wire.read();
  uv = (msb<<8) | lsb;
  uv = uv*5.625;        //Multiplizieren mit Umrechnungsfaktor
  delay(1000);
   postFloatValue(uv, 1, UVSENSOR_ID);
 
 
  //Feinstaubsensorloop
  duration = pulseIn(pin, LOW);
  lowpulseoccupancy = lowpulseoccupancy+duration;
 
  if ((millis()-starttime) >= sampletime_ms)//if the sampel time = = 30s
  {
    ratio = lowpulseoccupancy/(sampletime_ms*10.0);  // Integer percentage 0=&gt;100
    concentration = 1.1*pow(ratio,3)-3.8*pow(ratio,2)+520*ratio+0.62; // using spec sheet curve
 
    lowpulseoccupancy = 0;
    starttime = millis();
  }
  postFloatValue(concentration, 1, SENSOR2_ID);
}
}
```
 
Im weiterführenden Programmcode des loops stehen die Anweisungen, die uns von OSM übergeben wurden:
```
void postFloatValue(float measurement, int digits, String sensorId)
{
  //Float zu String konvertieren
  char obs[10];
  dtostrf(measurement, 5, digits, obs);
  //Json erstellen
  String jsonValue = "{\"value\":";
  jsonValue += obs;
  jsonValue += "}"; 
  //Mit OSeM Server verbinden und POST Operation durchführen
  Serial.println("-------------------------------------");
  Serial.print("Connectingto OSeM Server...");
  if (client.connect(server, 8000))
  {
    Serial.println("connected!");
    Serial.println("-------------------------------------");    
    //HTTP Header aufbauen
    client.print("POST /boxes/");client.print(SENSEBOX_ID);client.print("/");client.print(sensorId);client.println(" HTTP/1.1");
    client.println("Host: www.opensensemap.org");
    client.println("Content-Type: application/json");
    client.println("Connection: close"); 
    client.print("Content-Length: ");client.println(jsonValue.length());
    client.println();
    //Daten senden
    client.println(jsonValue);
  }else
  {
    Serial.println("failed!");
    Serial.println("-------------------------------------");
  }
  //Antwort von Server im seriellen Monitor anzeigen
  waitForServerResponse();
}
 
void waitForServerResponse()
{
  //Ankommende Bytes ausgeben
  boolean repeat = true;
  do{
    if (client.available())
    {
      char c = client.read();
      Serial.print(c);
    }
    //Verbindung beenden
    if (!client.connected())
    {
      Serial.println();
      Serial.println("--------------");
      Serial.println("Disconnecting.");
      Serial.println("--------------");
      client.stop();
      repeat = false;
    }
  }while (repeat);
}
```
 
## OpenSenseMap Registrierung
Die Registrierung erfolgt über Angabe des Namens und der E-Mail-Adresse auf dem OSM-Portal. Im Anschluss kann man die Art der Station, die Eigenschaften der benutzten Sensoren und den Standort der geplanten Station festgelegen. Daraufhin wird der individuelle Sketch für die Verbindung der Messstation zum OSM-Portal versandt.
 
## Stationsaufbau
Die Station sollte so aufgebaut werden, dass die Verfälschung der Daten möglichst gering ausfällt und alle Sensoren funktionieren können. In unseren Fall  haben wir den Feinstaubsensor und den Temperatur- und Luftfeuchtigkeitssensor so angebracht, dass sie keinen Schatten auf unseren UV-Sensor werfen. Diese beiden Sensoren sind außerdem außerhalb der Box angebracht, damit die Luftzufuhr gewährleistet ist. Jedoch müssen sie trotzdem vor Nässe geschützt sein.

<image
src=https://raw.githubusercontent.com/Alina1995/Bilder1/master/13000535_1006850769384804_1440555269_o.jpg width="200"/> <image
src=https://raw.githubusercontent.com/Alina1995/Bilder1/master/13009932_1006850796051468_131693779_o.jpg width="200"/> <image
src=https://raw.githubusercontent.com/Alina1995/Bilder1/master/13010003_1006850779384803_1388410211_o.jpg width="200"/>
 
 

 

 
## Kontakt
Alina Trefz, a_tref01@uni-muenster.de
 
Judith Becka, j_beck60@uni-muenster.de
 
Münster,
08.04.2016
