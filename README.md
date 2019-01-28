# IoT Device Simulator + Stringify

## In Action

<p align="center">
<img src="https://github.com/phyunsj/iot-device-simulator-2-stringify/blob/master/images/iot-simulator-stringify-1.gif" width="250px"/>
<img src="https://github.com/phyunsj/iot-device-simulator-2-stringify/blob/master/images/iot-simulator-stringify-2.gif" width="250px"/>
</p>

## Stringify Developer Module

With the Developer Module, you can connect just about anything that runs Node.js to Stringify. You can send and receive events and use your custom Thing in Flows just like any other Thing. The Developer module connects to Stringify and lets you write custom code in JavaScript to do just about anything you can imagine. _...from stringify_

See Also 
- [Step Up Your Automation](https://www.stringify.com/makers/)
- [Stringify News : Announcing Advanced Tools for Makers on May 26,2017](https://www.stringify.com/step-up-your-automation/)
- [Strinify Developer Module Rev 0.3 (June,2017)](https://www.stringify.com/app/uploads/2017/06/Node-Developer-Module-Technical-Doc-rev-03.pdf)

## `Developer Template` as a Trigger as well as a MQTT subscriber

- Trigger a notification `self.emit('trigger',...)` if If UV index is above 8.

```
     // client is a MQTT subscriber (topic "ny-10001/uv-sensor")
     client.on('message',function(topic, message, packet){
            if ( parseInt(message) > 8) { // 8<' Very High/Dangerous
              self.emit('trigger', {
                trigger : 'Warning'
              });
            }
        });
```

<p align="center">
<img src="https://github.com/phyunsj/iot-device-simulator-2-stringify/blob/master/images/node-red-mqtt-stringify-text.gif" width="700px"/>
</p>

- package.json

```
    +"@clusterws/cws": "^0.11.0", // https://www.npmjs.com/package/@clusterws/cws. "uws" replacement
    +"bonjour": "^3.5.0", // https://www.npmjs.com/package/bonjour. Publish/discover services on the local network 
    +"mqtt": "^2.18.8",
    -"uws": "^0.14.5", // deprecated
    // remove below packages since my example runs on a Mac
    -"stringify-developer-gpio": "https://cdn.stringify.com/developer/stringify-developer-gpio.tgz", 
    -"stringify-developer-speaker": "https://cdn.stringify.com/developer/stringify-developer-speaker.tgz", 
```

- node_modules/stringify-developer-template/index.js

```
const mqtt = require('mqtt');
const bonjour = require('bonjour')();
const settings = require('../../lib/settings'); 

const StringifyEventsModule = function (logger) {
    var self = this;
    var client; // mqtt subscriber
    if (!(this instanceof StringifyEventsModule)) return new StringifyEventsModule(logger);
    this.init = () => {
        logger.debug('stringify-developer-template module initialized');

        // browse for all mqtt brokers 
        bonjour.find({ type: 'mqtt' }, function (service) {
        settings.saveSettings( "MQTTAddress", service.addresses[0] );
        settings.saveSettings( "MQTTPort", service.port );
        settings.saveSettings( "MQTTClient", service.txt['clientid'] );

        var mqttOptions = {
            clientId:service.txt['clientid']+'-uv' // MUST be unique
        };
    
        client  = mqtt.connect('mqtt://'+service.addresses[0]+':'+ service.port,  mqttOptions);
        //handle incoming messages
        client.on('message',function(topic, message, packet){
            if ( parseInt(message) > 8) { // 8<' Very High/Dangerous
              self.emit('trigger', {
                trigger : 'Warning'
              });
            }
        });

        client.on('error', function (err) {
            logger.debug('Error from '+'mqtt://'+service.addresses[0]+':'+ service.port);
            logger.debug(err);
        });

        client.on('connect', function () {
            logger.debug('Connected to '+'mqtt://'+service.addresses[0]+':'+ service.port);
            client.subscribe('ny-10001/uv-sensor', function (err) {
            if(err) logger.debug('MQTT Subscriber Error:',err);
            });
        });

     });

    };
    ...
    
```
- lib/ws.js

```
+ const { WebSocket } = require('@clusterws/cws'),
- const WebSocket = require('uws'),
```

- lib/event-api.js

`user` from `stringifyEvents.getUser()` is always **null** even though `accessToken` is ccorrectly generated and stored in `$HOME/.stringify`. Not clear whether `https://api.stringify.com/v2/users/me` is a valid URL. 

Disable validation routine and create a websocket connection with `accessToken`. 



## Node-RED Iot Simulator + [Mosca MQTT Broker](https://github.com/zuhito/node-red-contrib-mqtt-broker) + [bonjour](https://www.npmjs.com/package/bonjour) 

<p align="center">
<img src="https://github.com/phyunsj/iot-device-simulator-2-stringify/blob/master/images/node-red-mqtt-broker.png" width="700px"/>
</p>

- MQTT Broker Announcement

```
// https://www.npmjs.com/package/bonjour
// See https://github.com/phyunsj/node-red-custom-dashboard-system-page to enable "require"
var bonjour = require('bonjour')() 
var service = flow.get("service") || null;
bonjour.unpublishAll();
if (service) service.stop(); // allow re-deploy
// advertise MQTT server
var service = bonjour.publish({ name: 'MQTT broker', 
                  type: 'mqtt', 
                  txt : { clientid : 'ny-10001-sensor1'},
                  port: 1883 });
flow.set('service', service);
return null;
```

#### Related Posts :

- [stringify forum : SDM troubleshooting](https://forums.stringify.com/t/developer-module-sdm/3403)
- [SmartThings:Custom device handlers to fool Stringify](https://community.smartthings.com/t/custom-device-handlers-to-fool-stringify/112226)
