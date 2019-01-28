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

Please follow the instructions from https://www.stringify.com/app/uploads/2017/06/Node-Developer-Module-Technical-Doc-rev-03.pdf and make any additional adjustment if necessary.

Changes I made for this example : 

- package.json

```
    +"@clusterws/cws": "^0.11.0", // https://www.npmjs.com/package/@clusterws/cws. "uws" replacement
    +"bonjour": "^3.5.0", // https://www.npmjs.com/package/bonjour. Publish/discover services on the local network 
    +"mqtt": "^2.18.8",
    -"uws": "^0.14.5", // deprecated
    -// remove below packages since my example runs on a Mac
    -"stringify-developer-gpio": "https://cdn.stringify.com/developer/stringify-developer-gpio.tgz", 
    -"stringify-developer-speaker": "https://cdn.stringify.com/developer/stringify-developer-speaker.tgz", 
```

- lib/ws.js

```
+ const { WebSocket } = require('@clusterws/cws'),
- const WebSocket = require('uws'),
```

- lib/event-api.js

**NOTE** `user` from `stringifyEvents.getUser()` is always _**null**_ although `accessToken` is ccorrectly generated and stored in `$HOME/.stringify`. Not clear whether `https://api.stringify.com/v2/users/me` is a valid URL. 

```         
            +// Disable user validation 
            +//if (user) {
                logger.debug(`Please visit ${httpHost} in your web browser to configure.`);
                wsConnection = new ws.connect(at);
                cb(null, wsConnection);
            +//} else {
            +//    settings.delSetting('accessToken');
            +//    logger.debug(`User credentials failed. Please visit ${httpHost} in your web browser to login.`);
            +//    cb(`Invalid credentials`);
            +//}
```

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

## Node-RED Iot Simulator + [Mosca MQTT Broker](https://github.com/zuhito/node-red-contrib-mqtt-broker) + [bonjour](https://www.npmjs.com/package/bonjour) 

<p align="center">
<img src="https://github.com/phyunsj/iot-device-simulator-2-stringify/blob/master/images/node-red-mqtt-broker.png" width="700px"/>
</p>

- Example Flow :
```
[{"id":"7f8ba21e.8148cc","type":"debug","z":"16dd99f8.e7c476","name":"Display msg.payload","active":false,"tosidebar":true,"console":false,"tostatus":false,"complete":"payload","x":301.49214935302734,"y":385.13576316833496,"wires":[]},{"id":"159e8e96.0a31d1","type":"inject","z":"16dd99f8.e7c476","name":"Sensor Trigger","topic":"","payload":"","payloadType":"date","repeat":"","crontab":"","once":false,"onceDelay":0.1,"x":115.49175262451172,"y":285.33571910858154,"wires":[["d9192b66.3f4c98"]]},{"id":"d9192b66.3f4c98","type":"iot-simulator","z":"16dd99f8.e7c476","name":"","options":[{"label":"Temperature","value":"30","range":"10"},{"label":"Air_Quality","value":"40","range":"10"},{"label":"UV_Index","value":"9","range":"3"}],"preOptions":[{"label":"Temperature","value":"30","range":"10"},{"label":"Air_Quality","value":"40","range":"10"},{"label":"UV_Index","value":"6","range":"5"}],"timestamp":true,"allinone":false,"optionEdited":false,"x":295.49195098876953,"y":284.2024335861206,"wires":[["7f8ba21e.8148cc","4ca9d91.95f1828"]]},{"id":"4ca9d91.95f1828","type":"function","z":"16dd99f8.e7c476","name":"Payload Split","func":"var msg1 = null\nif ( msg.payload[\"Temperature\"] !== undefined) {\n   msg1 = {};\n   msg1.payload = msg.payload[\"Temperature\"];\n}\n\nvar msg2 = null\nif ( msg.payload[\"Air_Quality\"] !== undefined){\n   msg2 = {};\n   msg2.payload = msg.payload[\"Air_Quality\"];\n}\n\nvar msg3 = null\nif ( msg.payload[\"UV_Index\"] !== undefined){\n   msg3 = {};\n   msg3.payload = msg.payload[\"UV_Index\"];\n}\nreturn [msg1, msg2, msg3];","outputs":3,"noerr":0,"x":500.4919853210449,"y":286.40238189697266,"wires":[["ba1120cf.cbe08"],["52a667d9.a86518"],["ba6254a5.f06d98","cbd576ea.5a0238"]]},{"id":"ba1120cf.cbe08","type":"ui_gauge","z":"16dd99f8.e7c476","name":"Temperature","group":"17ddf76d.548979","order":0,"width":"4","height":"4","gtype":"gage","title":"Temperature","label":"°C","format":"{{value}} °C","min":0,"max":"100","colors":["#00b500","#e6e600","#ca3838"],"seg1":"","seg2":"","x":705.4920539855957,"y":204.20241832733154,"wires":[]},{"id":"52a667d9.a86518","type":"ui_chart","z":"16dd99f8.e7c476","name":"Air Quality","group":"17ddf76d.548979","order":1,"width":"4","height":"4","label":"Air Quality","chartType":"line","legend":"false","xformat":"HH:mm:ss","interpolate":"linear","nodata":"","dot":true,"ymin":"","ymax":"","removeOlder":1,"removeOlderPoints":"","removeOlderUnit":"3600","cutout":0,"useOneColor":false,"colors":["#1f77b4","#aec7e8","#ff7f0e","#2ca02c","#98df8a","#d62728","#ff9896","#9467bd","#c5b0d5"],"useOldStyle":false,"x":707.4920539855957,"y":254.2024335861206,"wires":[[],[]]},{"id":"ba6254a5.f06d98","type":"ui_gauge","z":"16dd99f8.e7c476","name":"","group":"17ddf76d.548979","order":2,"width":"4","height":"4","gtype":"gage","title":"UV Index","label":"units","format":"{{value}}","min":"1","max":"12","colors":["#00b500","#e6e600","#ca3838"],"seg1":"6","seg2":"9","x":701.7152214050293,"y":299.76327419281006,"wires":[]},{"id":"cbd576ea.5a0238","type":"mqtt out","z":"16dd99f8.e7c476","name":"","topic":"ny-10001/uv-sensor","qos":"","retain":"","broker":"807fba11.28b758","x":739.5234298706055,"y":387.6133337020874,"wires":[]},{"id":"61932cae.a50be4","type":"mqtt in","z":"16dd99f8.e7c476","name":"ny-10001/sensor1","topic":"ny-10001/+","qos":"0","broker":"807fba11.28b758","x":545.51171875,"y":489.3594493865967,"wires":[["95a21c0b.8dc36"]]},{"id":"95a21c0b.8dc36","type":"debug","z":"16dd99f8.e7c476","name":"","active":true,"tosidebar":true,"console":false,"tostatus":false,"complete":"payload","x":722.5234642028809,"y":488.6602306365967,"wires":[]},{"id":"75d5b10a.d1dc5","type":"function","z":"16dd99f8.e7c476","name":"MQTT Broker Annoucement","func":"\n// https://www.npmjs.com/package/bonjour\nvar bonjour = require('bonjour')()\nvar service = flow.get(\"service\") || null;\n\nbonjour.unpublishAll(); \n\nif (service) {\n    service.stop();\n}\n\n// advertise MQTT server\nvar service = bonjour.publish({ name: 'MQTT broker', \n                  type: 'mqtt', \n                  txt : { clientid : 'ny-10001-sensor1'},\n                  port: 1883 });\n\nflow.set('service', service);\n\nreturn null;","outputs":1,"noerr":0,"x":319.5234375,"y":71.1523494720459,"wires":[[]]},{"id":"640b59.5b2c84a8","type":"inject","z":"16dd99f8.e7c476","name":"","topic":"Started!","payload":"","payloadType":"str","repeat":"","crontab":"","once":true,"onceDelay":"0.1","x":108.51171875,"y":70.44529724121094,"wires":[["75d5b10a.d1dc5"]]},{"id":"63ad7813.da72b8","type":"function","z":"16dd99f8.e7c476","name":"MQTT service finder","func":"//https://www.npmjs.com/package/bonjour\nvar bonjour = require('bonjour')()\n \n// browse for all http services\nbonjour.find({ type: 'mqtt' }, function (service) {\n  console.log('Found an MQTT broker:', service)\n  \n})\n\nreturn null;","outputs":1,"noerr":0,"x":796.51953125,"y":71.15235710144043,"wires":[[]]},{"id":"8126d77c.34b598","type":"inject","z":"16dd99f8.e7c476","name":"","topic":"Test MQTT finder","payload":"","payloadType":"str","repeat":"","crontab":"","once":false,"onceDelay":0.1,"x":579.5234375,"y":72.43361473083496,"wires":[["63ad7813.da72b8"]]},{"id":"594183a9.e30ecc","type":"comment","z":"16dd99f8.e7c476","name":"Debug : find MQTT service","info":"","x":589.26171875,"y":30.582021713256836,"wires":[]},{"id":"fc79e967.e4c548","type":"comment","z":"16dd99f8.e7c476","name":"Start MQTT Service/Publish","info":"","x":143.5234375,"y":31.58203125,"wires":[]},{"id":"25a5b901.b54946","type":"comment","z":"16dd99f8.e7c476","name":"IoT Device - Temperature, Air Quality, UV Index - mqtt publisher","info":"","x":241.5234375,"y":239.58202838897705,"wires":[]},{"id":"7799c715.45b3e8","type":"comment","z":"16dd99f8.e7c476","name":"Mosca MQTT Broker","info":"","x":109.515625,"y":119.58208274841309,"wires":[]},{"id":"63d55c41.b46a74","type":"comment","z":"16dd99f8.e7c476","name":"MQTT subscriber \"ny-10001/+\"","info":"","x":582.51953125,"y":450.5703639984131,"wires":[]},{"id":"36ce3c3e.5ec814","type":"comment","z":"16dd99f8.e7c476","name":"MQTT publisher","info":"","x":721.26953125,"y":345.8242177963257,"wires":[]},{"id":"d2d09d80.23273","type":"debug","z":"16dd99f8.e7c476","name":"","active":false,"tosidebar":true,"console":false,"tostatus":false,"complete":"false","x":303.51952362060547,"y":159.64457511901855,"wires":[]},{"id":"88fde18b.95e77","type":"mosca in","z":"16dd99f8.e7c476","mqtt_port":1883,"mqtt_ws_port":8080,"name":"","username":"","password":"","dburl":"","x":110.515625,"y":158.76955318450928,"wires":[["d2d09d80.23273"]]},{"id":"17ddf76d.548979","type":"ui_group","z":"","name":"SensorData","tab":"50111b43.8a2fe4","order":2,"disp":true,"width":"12","collapse":false},{"id":"807fba11.28b758","type":"mqtt-broker","z":"","name":"ny-10001-sensor1","broker":"localhost","port":"1883","clientid":"ny-10001-sensor1","usetls":false,"compatmode":true,"keepalive":"60","cleansession":true,"birthTopic":"","birthQos":"0","birthPayload":"","closeTopic":"","closeQos":"0","closePayload":"","willTopic":"","willQos":"0","willPayload":""},{"id":"50111b43.8a2fe4","type":"ui_tab","z":"","name":"Dashboard","icon":"dashboard","order":1}]
```
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
