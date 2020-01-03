# Enabling Jablotron security sensors to MQTT
Past year we bought house where was installed Jablotron JA-80 alarm system. As normal security system, it is wired in close box with no API. It is nice that they have App, web, but without API I was struggling to connect it to my Smart Home running with HomeKit and Homebridge.

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


## Wiring 
Wiring itself will take a while, be patience and prepare some coffee.

1. To start wiring, put your Jablotron to the Service mode - \* 0 \<YOUR PIN\>
2. It is recommended to turn off Jablotron before wiring
3. Connect wires to Jablotron Pins to Arduino Mega and Arduino Ethernet Shield- keep there current wire and add there one you want to connect to Arduino. In my case 1-16 and GND
4. Connect wires to Arduino Pins - idealy 0-15 and GND in the same order
5. Close box with Jablotron, turn electricity on and turn off Service mode with \#

## Optional - connecting to Homebridge
- running homebridge mqttthing
- running homebridge
- running jablotron
