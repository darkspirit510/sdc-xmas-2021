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

- Without persistence, openHAB just keeps values in RAM
- On startup, all values are undefined
    - If set, openHAB will receive all values via persistence
    - openHAB will receive/update values from provider (channel), if possible
- openHAB offers several persistence bindings for different use cases
    - RRD4J: Round robin, stores a series of values. It has a fixed size and will replace the oldest value if the limit
      is reached.
    - MapDB: Just like Java map, it's a simple, builtin key-value store. Useful for restoring values on boot up.
    - Multiple database integrations (MongoDB, DynamoDB, MySQL, MariaDB, InfluxDB, ...)
- It is possible to use multiple persistence addons with different configuration

### jdbc.cfg

```
# [...]
sqltype.STRING        =   VARCHAR(65500)
# [...]
tableUseRealItemNames=true
# [...]
tableIdDigitCount=0
# [...]
```

### mapdb.persist

```
Strategies {
        default = everyUpdate
}

Items {
        * : strategy = everyChange, restoreOnStartup
}
```

### jdbc.persist

```
Strategies {
    everyFiveMinutes : "0 */5 * * * ?"
    everyHour : "0 0 * * * ?"
    default = everyChange
}

Items {
    StoreFiveMinutes* : strategy = everyFiveMinutes
    StoreEveryHour* : strategy = everyHour

    StoreChange* : strategy = everyChange
    StoreUpdate* : strategy = everyUpdate
}
```

### tankerkoenig.items

```
Group StoreEveryHour

Number StarTankstelleFallersleben_E10       "E10 [%.3f €]"      <oil>  (StoreEveryHour)  {channel="tankerkoenig:station:Tankerkoenig:StarTankstelleFallersleben:e10"}
Number StarTankstelleFallersleben_Super     "Super [%.3f €]"    <oil>  (StoreEveryHour)  {channel="tankerkoenig:station:Tankerkoenig:StarTankstelleFallersleben:e5"}
Number StarTankstelleFallersleben_Diesel    "Diesel [%.3f €]"   <oil>  (StoreEveryHour)  {channel="tankerkoenig:station:Tankerkoenig:StarTankstelleFallersleben:diesel"}
```

# Topic 2 - HTTP

HTTP binding allows to add endpoints as Things (I'm not using this) or simply send HTTP requests within rules.

### GET-Request

```
rule "Toggle docking station"

when
    Channel "deconz:switch:homeserver:OfficeSwitchDockingStation:buttonevent" triggered
then
  if(receivedEvent == "1001" || receivedEvent == "1002") {
    sendHttpGetRequest("http://shelly-docking-station/relay/0?turn=toggle")
  }
end
```

### POST/PUT request (with body)

```
rule "Turn everything off"

when
    Channel "hue:0820:01234567890:90:dimmer_switch_event" triggered
then
    if(receivedEvent.startsWith("400")) {
        sendHttpPutRequest("http://hue-bridge/api/{API-TOKEN}/groups/1/action", "text/plain", "{\"on\":false}")
        sendHttpPutRequest("http://hue-bridge/api/{API-TOKEN}/groups/2/action", "text/plain", "{\"on\":false}")
    }
end
```

Source: [openHAB docs - HTTP Binding](https://www.openhab.org/addons/bindings/http/)

# Topic 3 - MQTT

## Basics

"MQTT is an OASIS standard messaging protocol for the Internet of Things (IoT)." (mqtt.org)

- Lots of open source and commercial products support MQTT to send/receive data to/from a given MQTT server (broker)
- Has a small implementation footprint (can run on most microcontrollers)
- Uses almost zero CPU, even for hundreds of messages
- Clients can publish data to specific topic (/home/basement/kitchen/lights/1/brightness = 100)
- Clients can subscribe data from specific or wildcard topic (/home/basement/kitchen/lights/1/brightness,
  /home/basement/kitchen/lights/+/brightness)
    - \+ = single level wildcard (/a/+/c matches /a/b/c, but not /a/b1/b2/c)
    - \# = multi level wildcard (/a/# matches /a/b/c and /a/b1/b2/c)
    - \# is nice for debugging, but "dangerous" on root level
- openHAB comes with both MQTT client and broker, but we'll just use the client

## openHAB Thing(s)

```
Bridge mqtt:broker: MqttBroker @ "Installation room" [
  host="broker.hivemq.com",
  clientId="openHAB demo 1234"
] {
Thing topic Raspberry "Raspberry" @ "Wohnzimmer" {
  Channels:
    Type number: temperature "Temperature" [ stateTopic="sdc-xmas-2021/temp" ]
  }

  Thing topic Robbie "Robbie" @ "Living Room" {
    Channels:
      Type string : state "State" [ stateTopic="valetudo/rockrobo/state", transformationPattern="JSONPATH:$.state" ]
  }
  
  Thing topic ShellyOffice "Shelly" @ "Office" {
    Channels:
      Type switch : switch1 "Switch 1" [ stateTopic="shellies/office/input/0", on="1", off="0" ]
  }
}
```

Note: Transformations are available after installation.

## openHAB Item(s)

```
Number Raspberry_Temperature "Temperature" { channel="mqtt:topic:MqttBroker:Raspberry:temperature" }

String Robbie_Command { channel="mqtt:topic:MqttBroker:Robbie:state" }

Switch Arbeitszimmer_Schreibtisch_Schalter "Docking Station state" { channel="mqtt:topic:MqttBroker:ShellyOffice:switch1" }
```

Source: [openHAB docs - MQTT Binding](https://www.openhab.org/addons/bindings/mqtt.generic/)

# Topic 4 - Notifications

Can be accomplished via several ways. I prefer telegram because it has a great bot API and a stable, reliable plugin. If
you want to use this way, [create your bot first](https://core.telegram.org/bots#creating-a-new-bot). This section could
also be done via Slack (or any other HTTP) notification. openHAB also supports crazy stuff like sending E-Mail or even
low-level things like RS232. But why would anyone want to do this?

## telegram.cfg

```
#
# Read http://www.instructables.com/id/Telegram-Bots-for-beginners/
# to see how to set up bots and find your chat ids
#

bots=botSascha,botGroup

# bot1.chatId=22334455
# bot1.token=xxxxxx

# bot2.chatId=654321
# bot2.token=yyyyyyyyyyy

botSascha.chatId=12345678
botSascha.token=12345678:abcdABCDabcdABCDabcdABCDabcdABCD

botGroup.chatId=-12345678
botGroup.token=12345678:abcdABCDabcdABCDabcdABCDabcdABCD
```

## dishwasher.rules

```
rule "Dishwasher done"

when
  Item Kitchen_Shelly_Dishwasher_Wattage changed to 0
then
  Thread::sleep(90000)

  if(Kitchen_Shelly_Dishwasher_Wattage.maximumSince(now.minusSeconds(89), "jdbc").state == 0) {
    getActions("telegram", "telegram:telegramBot:TelegramBot").sendTelegram(-12341234L, "Dishwasher has finished!")
  }
end
```

## robbie.rules

```
rule "Robbie finished cleaning"

when
    Item Robbie_Command changed from "returning" to "docked"
then
    getActions("telegram", "telegram:telegramBot:TelegramBot").sendTelegramPhoto(-12341234L, "http://192.168.1.156:8803/api/map/image", "Robbie ist fertig!")
end
```
