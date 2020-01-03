# Enabling Jablotron security sensors to MQTT
Past year we bought house where was installed Jablotron JA-80 alarm system. As normal security system, it is wired in close box with no API. It is nice that they have App, web, but without API I was struggling to connect it to my Smart Home running with HomeKit and Homebridge.
This enable me turning on/off heating based on open window, run alert during rain if window is open, turning on light with 10% brightness on stairs during night if someone is comming etc. using sensors which was already in place.

Later, as a following step I was connect this MQTT and Jablotron itself to Homebridge - read lower is nice to have.

## Components
Arduino Board with enough analog pins - I chose Arduino Mega to have 16 analog pins
Arduino Ethernet Shield
Ethernet cable
Power source for Arduino
Enough wires to connect Arduino to Jablotron

## Prerequisities
You already setup MQTT. If not, I can recommend setup described here https://www.baldengineer.com/mqtt-tutorial.html

## Code
First, include the library in your sketch along with wires, mqtt and ethernet
```
#include <SPI.h>
#include <Ethernet.h>
#include <PubSubClient.h>
```

Change your MQTT server url and setup your mac address
```
byte mac[] = { 0x00, 0xAA, 0xBB, 0xCC, 0xDE, 0x02 };   //mac address for Arduino
IPAddress server(192, 168, 1, 100); //your mqtt server
EthernetClient ethClient;
PubSubClient client(ethClient);
```

I am setuping regular delays for every device connected to internet. Wire your digital Pin to restart Pin
```
const long restartInterval = 3600000; //how ofter will be Arduino restared
unsigned long restartMillis = 0;
int Restart = 21; //wire pin 21 to reset pin
```

Setup arrays for all Pins which you will use
```
unsigned long last_sub[] = {0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0};
float previousvoltage[] = {0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0};
```

Generating callback for mqtt
```
void callback(char* topic, byte* payload, unsigned int length) {
  Serial.println("callback start");
}
```

Reconnect if there is problem with connectivity
```
boolean reconnect() {
  if (client.connect("jablotron")) {
    // Once connected, publish an announcement...
    client.publish("jablotron","hello from home");
    // ... and resubscribe
    client.subscribe("jablotron");
  }
  return client.connected();
}
```

Setup will start your connectivity and connect to MQTT
```
void setup(){
  Serial.begin(9600);
  while (!Serial) {
    ; // wait for serial port to connect. Needed for native USB port only
  }

  // start the Ethernet connection:
  Serial.println("Initialize Ethernet with DHCP:");
  if (Ethernet.begin(mac) == 0) {
    Serial.println("Failed to configure Ethernet using DHCP");
    if (Ethernet.hardwareStatus() == EthernetNoHardware) {
      Serial.println("Ethernet shield was not found.  Sorry, can't run without hardware. :(");
    } else if (Ethernet.linkStatus() == LinkOFF) {
      Serial.println("Ethernet cable is not connected.");
    }
    // no point in carrying on, so do nothing forevermore:
    while (true) {
      delay(1);
    }
  }
  // print your local IP address:
  Serial.print("My IP address: ");
  Serial.println(Ethernet.localIP());
  
  client.setServer(server, 1883);
  client.setCallback(callback);
}
```

Setup timer and check if serial is working. If not, reboot
```
 unsigned long currentMillis = millis(); 
 if (Serial.available() > 0){
    if (Serial.read() == '@')
    {
      Serial.println("Rebooting. . .");
      delay(100); // Give the computer time to receive the "Rebooting. . ." message, or it won't show up
      void (*reboot)(void) = 0; // Creating a function pointer to address 0 then calling it reboots the board.
      reboot();
    }
  }
  ```
  
Restarting based on time interval
```  
if (currentMillis - restartMillis >= restartInterval) {
      restartMillis = currentMillis;
      client.publish("jablotron", "Restart");
      Serial.println("Restart");
      delay(1000);
      digitalWrite(Restart, LOW);
}
```

Maintaining connectivity
```
switch (Ethernet.maintain()) {
    case 1:
      //renewed fail
      Serial.println("Error: renewed fail");
      break;

    case 2:
      //renewed success
      Serial.println("Renewed success");
      //print your local IP address:
      Serial.print("My IP address: ");
      Serial.println(Ethernet.localIP());
      break;

    case 3:
      //rebind fail
      Serial.println("Error: rebind fail");
      break;

    case 4:
      //rebind success
      Serial.println("Rebind success");
      //print your local IP address:
      Serial.print("My IP address: ");
      Serial.println(Ethernet.localIP());
      break;

    default:
      //nothing happened
      break;
}
```

Checking if connection is working. If yes, progress
```
if (!client.connected()) {
    if (now - lastReconnectAttempt > 10000) {
      lastReconnectAttempt = now;
      // Attempt to reconnect
      if (reconnect()) {
        lastReconnectAttempt = 0;
      }
        Serial.println("reconnection");
    }
  } 
```

Reading values from analog pins and publishing them to mqtt
```
    if (currentMillis - previousMillis >= interval) {
      previousMillis = currentMillis;
      
      byte Pins[] = {A0, A1, A2, A3, A4, A5, A6, A7, A8, A9, A10, A11, A12, A13, A14, A15};
      int sensors[] = {0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15};
      int last_activity = 300000; //how often we want to get update if no change with sensor state

      for (int k=0; k <= 15; k++){
        sensors[k] = analogRead(Pins[k]); //reading Pin value
        float voltage = sensors[k] * (5.0 / 1024.0);  //calculating it to volts
        char mes[50];
        sprintf(mes, "jablotron/%d", k); //preparing meessage to char for mqtt
        char mes2[50];
        if(voltage > 1 && (previousvoltage[k] < 1 || (now - last_sub[k] > last_activity))){  //if sensor is active, and last command was non-active or no responce for long time
          last_sub[k] = now;
          sprintf(mes2, "jablotron/%d - OPEN", k);
          client.publish(mes, "OPEN");
          client.publish("jablotron", mes2);
          Serial.print(mes);
          Serial.print(" - pin output > 0 - ");
          Serial.println(voltage);
          Serial.print(previousvoltage[k]);
          Serial.println("previous");
          previousvoltage[k] = voltage;
        }else if (voltage < 1 && (previousvoltage[k] > 1 || (now - last_sub[k] > last_activity))) { //if sensor is non-active, and last command was active or no responce
          last_sub[k] = now;
          sprintf(mes2, "jablotron/%d - CLOSED", k);
          client.publish(mes, "CLOSED");
          client.publish("jablotron", mes2);
          Serial.print(mes);
          Serial.print(" - pin output < 0 - ");
          Serial.println(voltage);
          previousvoltage[k] = voltage;
        } 
      } 
    }
    client.loop();
```

## Wiring 
Wiring itself will take a while, be patience and prepare some coffee.

1. To start wiring, put your Jablotron to the Service mode - \* 0 \<YOUR PIN\>
2. It is recommended to turn off Jablotron before wiring
3. Connect wires to Jablotron Pins to Arduino Mega and Arduino Ethernet Shield- keep there current wire and add there one you want to connect to Arduino. In my case 1-16 and GND
4. Connect wires to Arduino Pins - idealy 0-15 and GND in the same order
5. Plug in Arduino to electricity and ethernet
5. Close box with Jablotron, turn electricity on and turn off Service mode with \#

## Optional - connecting to Homebridge
To be able to automatisation and operating rules and scenes at home I am using Homebridge. 
- Just install Homebridge to some server or even Raspberry - https://homebridge.io
- Connect Arduino to your Homebridge - I am using MQTT Thing - https://github.com/arachnetech/homebridge-mqttthing#readme
- For connecting Alarm to your home, I am using also Homebridge with this plugin - https://github.com/F4stFr3ddy/homebridge-jablotron-alarm

## TODO
- [x] Window sensor
- [x] Motion sensor
- [ ] Smoke sensor

Enjoy!
