#include <SPI.h>
#include <Ethernet.h>
#include <PubSubClient.h>

// Update these with values suitable for your hardware/network.
byte mac[] = { 0x00, 0xAA, 0xBB, 0xCC, 0xDE, 0x02 };   //mac address for Arduino
IPAddress server(192, 168, 1, 50); //your mqtt server
EthernetClient ethClient;
PubSubClient client(ethClient);

//delay setup
unsigned long previousMillis = 0;
const long interval = 5000; //update interval
long lastReconnectAttempt = 0;

//restart
const long restartInterval = 3600000; //how ofter will be Arduino restared
unsigned long restartMillis = 0;
int Restart = 21; //wire pin 21 to reset pin

//arrays
unsigned long last_sub[] = {0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0};
float previousvoltage[] = {0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0};

void callback(char* topic, byte* payload, unsigned int length) {
  Serial.println("callback start");
}

boolean reconnect() {
  if (client.connect("jablotron")) {
    // Once connected, publish an announcement...
    client.publish("jablotron","hello from satna");
    // ... and resubscribe
    client.subscribe("jablotron");
  }
  return client.connected();
}

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


void loop(){
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

  if (currentMillis - restartMillis >= restartInterval) {
      restartMillis = currentMillis;
      client.publish("jablotron", "Restart");
      Serial.println("Restart");
      delay(1000);
      digitalWrite(Restart, LOW);
  }

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
  
  long now = millis();
  if (!client.connected()) {
    if (now - lastReconnectAttempt > 10000) {
      lastReconnectAttempt = now;
      // Attempt to reconnect
      if (reconnect()) {
        lastReconnectAttempt = 0;
      }
        Serial.println("reconnection");
    }
  } else {
    // Client connected
    if (currentMillis - previousMillis >= interval) {
      previousMillis = currentMillis;
      
      byte Pins[] = {A0, A1, A2, A3, A4, A5, A6, A7, A8, A9, A10, A11, A12, A13, A14, A15};
      int sensors[] = {0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15};
      int last_activity = 300000; //how often we want to get update if no change with sensor state

      for (int k=0; k <= 15; k++){
        sensors[k] = analogRead(Pins[k]); //reading Pin value
        float voltage = sensors[k] * (5.0 / 1024.0);  //calculating it to volts
        char message_prep[50];
        sprintf(message_prep, "jablotron/%d", k); //preparing meessage to char for mqtt
        char message[50];
        if(voltage > 1 && (previousvoltage[k] < 1 || (now - last_sub[k] > last_activity))){  //if sensor is active, and last command was non-active or no responce for long time
          last_sub[k] = now;
          sprintf(message, "jablotron/%d - OPEN", k);
          client.publish(message_prep, "OPEN");
          client.publish("jablotron", message);
          Serial.print(message_prep);
          Serial.print(" - pin output > 0 - ");
          Serial.println(voltage);
          Serial.print(previousvoltage[k]);
          Serial.println("previous");
          previousvoltage[k] = voltage;
        }else if (voltage < 1 && (previousvoltage[k] > 1 || (now - last_sub[k] > last_activity))) { //if sensor is non-active, and last command was active or no responce
          last_sub[k] = now;
          sprintf(message, "jablotron/%d - CLOSED", k);
          client.publish(message_prep, "CLOSED");
          client.publish("jablotron", message);
          Serial.print(message_prep);
          Serial.print(" - pin output < 0 - ");
          Serial.println(voltage);
          previousvoltage[k] = voltage;
        } 
      } 
    }
    client.loop();
  }
}
