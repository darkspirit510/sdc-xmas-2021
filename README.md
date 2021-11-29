# openHAB on Steroids (SDC-XMAS-2021)

Just like last year, this workshop will be "guided try-and-error". To me, all of the following topics are important in
my openHAB instance. I'll explain each topic, present some use cases and show how you can implement them. After that,
you can try everything out and ask for help if anything fails. After this workshop you (hopefully) have a basic
understanding of each topic and how to it might help you.

## Topics

- Persistence: (Re)store information within openHAB or in any database
- HTTP: Create HTTP requests to interact with other systems
- MQTT: Send/Receive data via MQTT
- Notifications: Send data via Telegram

# Preconditions

Create openHAB and MariaDB using the following docker-compose file.

```yaml
version: '2.2'

services:
  openhab:
    container_name: openhab
    image: "openhab/openhab:3.1.0"
    restart: always
    ports:
      - "8080:8080"
      - "8443:8443"
    volumes:
      - "./openhab_addons:/openhab/addons"
      - "./openhab_conf:/openhab/conf"
      - "./openhab_userdata:/openhab/userdata"
    environment:
      OPENHAB_HTTP_PORT: "8080"
      OPENHAB_HTTPS_PORT: "8443"
      EXTRA_JAVA_OPTS: "-Duser.timezone=Europe/Berlin"

  mariadb:
    image: mariadb
    container_name: mariadb
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: root
```

After this, create the admin account and install

- HTTP Binding
- MQTT Binding
- Telegram Binding


- MapDB Persistence
- JDBC Persistence MariaDB

# Topic 1 - Persistence

# Topic 2 - HTTP

# Topic 3 - MQTT

## Basics

"MQTT is an OASIS standard messaging protocol for the Internet of Things (IoT)." (mqtt.org)

- Lots of open source and commercial products support MQTT to send/receive data to/from a given MQTT server (broker)
- Has a small implementation footprint (can run on most microcontrollers)
- Uses almost zero CPU, even for hundreds of messages
- Clients can publish data to specific topic (/home/basement/kitchen/lights/1/brightness = 100)
- Clients can subscribe data from specific or wildcard topic (/home/basement/kitchen/lights/1/brightness, /home/basement/kitchen/lights/+/brightness)
    - \+ = single level wildcard (/a/+/c matches /a/b/c, but not /a/b1/b2/c)
    - \# = multi level wildcard (/a/# matches /a/b/c and /a/b1/b2/c)
    - \# is nice for debugging, but "dangerous" on root level
- openHAB comes with both MQTT client and broker, but we'll just use the client

## openHAB Thing(s)

```json
Bridge mqtt:broker:MqttBroker @ "Installation room" [
  host="broker.hivemq.com",
  clientId="openHAB demo 1234"
] {
  Thing topic Raspberry "Raspberry" @ "Wohnzimmer" {
    Channels:
      Type number : watt "Watt" [ stateTopic="sdc-xmas-2021/temp" ]
    }

//  Thing topic Robbie "Robbie" @ "Living Room" {
//    Channels:
//      Type string : state "State" [ stateTopic="valetudo/rockrobo/state", transformationPattern="JSONPATH:$.state" ]
//  }
//
//  Thing topic ShellyOffice "Shelly" @ "Office" {
//    Channels:
//      Type switch : switch1 "Switch 1" [ stateTopic="shellies/office/input/0", on="1", off="0" ]
//  }
]
```

## openHAB Item(s)



# Topic 4 - Notifications
