# node-modbus-rtu
Pure NodeJS implementation of ModbusRTU protocol using [node-serialport](https://github.com/voodootikigod/node-serialport) and promises

## Implementation notes
This library implement ONLY **ModbusRTU Master** and only few most important features:
 * **03** Read Holding Registers
 * **06** Write Single Register
 * **16** Write Multiple Registers

Coil functions (readCoils, writeCoils) are not yet implemented. Please feel free to fork and add these features.

Also Modbus response doesn't error check. (If the slave device returns an exception packet, it will not be understood as a failure.), Howeverm CRC is checked.

## Installation
The simplest way to install node-modbus-rtu is via npm.

Add to `packages.json` 2 dependencies:

```js
  "dependencies": {
    //...
    "modbus-rtu" : "*",
    "serialport" : "*",
    //...
   }
```

Then run `npm i`.

**NOTE!** [node-serialport](https://github.com/voodootikigod/node-serialport) doesn't support  node 0.11.11+. Check [documentation](https://github.com/voodootikigod/node-serialport) to more info


## Benefits
The main goal of this library is native JS implementation of the MODBUS-RTU protocol.
There is another nodejs modbusrtu implementation, however it uses native libmodbus library which work only on *nix systems. Furthermore, it is unsupported by current versions of node (binaries don't compile).

There are a few implementations of ModbusTCP protocol, however these are not compatible with RTU protocol.

This library uses node-serialport which has an active community, node buffers and promises.

You don't need deal with timeouts or think about sequental writing to serialport, this is handled by the library.

## Examples

### The basic example
```js
var SerialPort = require('serialport').SerialPort;
var modbus = require('modbus-rtu');

//create serail port with params. Refer to node-serialport for documentation
var serialPort = new SerialPort("/dev/ttyUSB0", {
   baudrate: 2400
});

//create ModbusMaster instance and feed them serial port
new modbus.Master(serialPort, function (master) {
      //Read from slave with address 1 four holding registers starting from 0.
      master.readHoldingRegisters(1, 0, 4).then(function(data){
        //promise will be fulfilled with parsed data
        console.log(data); //output will be [10, 100, 110, 50] (numbers just for example)
      }, function(err){
        //or will be rejected with error
        //for example timeout error or crc.
      })

      //Write to first slave, second register, value 150.
      //-slave-, -register-, -value-
      master.writeSingleRegister(1, 2, 150).then(success, error);
})
```

### Poll data from slaves in loop.

When polling in loop, it is necessary to wait until all responses are collected, otherwise pause between loop will be not work. (The number of returned bytes may vary).

All promises are collected into an array:  Q.all().  Then, a new promise is created which will be fulfilled when all requests are finished.

```js
var SerialPort = require('serialport').SerialPort;
var modbus = require('modbus-rtu');
var Q = require('q');

//create serail port with params. Refer to node-serialport for documentation
var serialPort = new SerialPort("/dev/ttyUSB0", {
   baudrate: 2400
});

new modbus.Master(serialPort, function (master) {
   var promises = [];

   (function loop (){
      //Read from slave 1
      promises.push(master.readHoldingRegisters(1, 0, 4).then(function(data){
        console.log('slave 1', data);
      }))

      //Read from slave 2
      promises.push(master.readHoldingRegisters(2, 0, 4).then(function(data){
         console.log('slave 2', data);
      }))

      //Read from slave 3
       promises.push(master.readHoldingRegisters(3, 0, 4).then(function(data){
          console.log('slave 3', data);
       }))

      Q.all(promises).catch(function(err){
        console.log(err); //catch all errors
      }).finally(function(){
        //when all promises fullfiled or rejected, restart loop with timeout
        setTimeout(loop, 300);
      })
   })()
})
```
This approach is very similar to arduino or PLC programming workflow, but it extremely uncomfortable in JS.

Imagine that you need do something after response is received. In this approach you need to do it in a promise callback of particular request.
If you need to wait more than one request or do a cascade (wait one, then second, one by one) Your code became into callback doom very quickly.

So I suggest another approach: apply OOP and IoC pattern to code and write some classes for our slaves:

### Use in real project: creating objects for slaves
For example we have a modbus thermostat, and we want to do something when data from thermostat is changed. 

Create class for this thermostat (suggest extracting this to a new file...):

```js
var _ = require('lodash'); //npm i lodash
var MicroEvent  = require('microevent'); //npm i microevent

module.exports = Thermostat;

var ENABLED_REGISTER = 0,
    FAN_SPEED_REGISTER = 1,
    MODE_REGISTER = 2,
    ROOM_TEMP_REGISTER = 3,
    TEMP_SETPOINT_REGISTER = 4,


function Thermostat(modbusMaster, modbusAddr) {
    this.modbusMaster = modbusMaster;
    this.modbusAddr = modbusAddr;

    this.enabled = false;
    this.fanSpeed = null;
    this.mode = null;
    this.roomTemp = null;
    this.tempSetpoint = null;

    this._rawData = [];
    this._oldData = [];

    this.watch();
}

MicroEvent.mixin(Thermostat);

_.extend(Thermostat.prototype, {
    update: function () {
        var th = this;
        return this.modbusMaster.readHoldingRegisters(this.modbusAddr, 0, 6)
        .then(function (data) {
            th._rawData = data;

            th.enabled = data[ENABLED_REGISTER] != 90;
            th.fanSpeed = data[FAN_SPEED_REGISTER];
            th.mode = data[MODE_REGISTER];
            th.roomTemp = data[ROOM_TEMP_REGISTER] / 2;
            th.tempSetpoint = data[TEMP_SETPOINT_REGISTER] / 2;

        })
    },

    toString: function(){
       return 'Status: '+ (this.enabled ? 'on' : 'off') +
        '; Room temp: ' + this.roomTemp + 'C; Set temp: ' + this.tempSetpoint +'C;'
    },

    watch: function () {
        var self = this;

        self.update().finally(function () {
            if (!_.isEqual(self._oldData, self._rawData)) {
                self.trigger('change', self);
                self._oldData = self._rawData.slice(0); //clone data array
            }

            setTimeout(function () {
                self.watch();
            }, 300)
        }).catch(function(err){
            console.log(err);
        }).done();
    }
})

```

This class blackboxes all modbus communication and provides a simple, clean api.

```js
new modbus.Master(serialPort, function (modbus) {
    var t = new Thermostat(modbus, slave);
    t.bind('change', function(){
        console.log('Thermostat '+ i +'. '+ t.toString());
        onThermostatUpdate();
    });
})
```

Now our thermostat will trigger callback only if data is changed.

One can write similar classes for all slave devices, add events which make sense for particular situations and write useful code more comfortably.


### How it works inside

Communicating via serial port is sequential. It is not possible to write a few requests, then read a few responses.

It is necessary to write something to serial, then wait for the answer, then write the next request, etc.

End of packet must be determined by timeout, which varies depending on baudrate.

The problem is, if scripted functions are called in synchronus style (one by one without callbacks),
they will write to port immediately. This causes a failure, because the slave tries to respond immediately without waiting for response of previous request, so as a result we receive trash...


To deal with this problem, the library provides simple queuing. 


When a modbus function is called, it does not write to the port immediately, but instead the function is added to a queue.
This returns a promise.  
The quese processes the request, and adds a timeout to the promise.
If the result of the queue is not recieved in `constants.RESPONSE_TIMEOUT;` the promise will be rejected with a timeout error.

SerialPort will accept data of variable length.  It may recieve 2 bytes per tick, or more than 4.  Modbus packets are or variable length.

The task is then to collect all of the responded bytes and correctly determine the end of the packet.

This library uses the debounce function for collecting buffer.

Each time serialPort recieves data, it triggers a function which stores response buffer in an array and calls the debounce function.

The debounce function ignores calls which happen too often while buffers are being stored.  (PLEASE VERIFY STATEMENT ACCURACY.)
Debounced function ignores calls which happen to often and while it we store buffers.

When pause between calls reaches a timeout state, the function inside debounce is called and all stored buffers will be concatenated.

This indicates end-of-packet, and the promise is fulfilled.

Each of the timeouts (for debounce, and promise) can be tuned for a particular project.

If the debounce timeout is too small, it will incorrectly collect buffers and you'll see CRC response errors.
If the debounce timeout is too large, a lot of time will be wasted in the communication cycle.

If there are many connected slaves, this timout becomes very important because it may take more than a minute to complete a single poll request for all devices on a segment!

#### How to set timeouts
All constants stored in `modbus-rtu/constants.js`. You can require this file and override default values:

```js
var constants = require('modbus-rtu/constants');
constants.END_PACKET_TIMEOUT = 15;
constants.RESPONSE_TIMEOUT = 500;
```

### API Documentation

#### new modbus.Master(serialPort, onReady)

Constructor of modbus class.

* **serialPort** - instance of serialPort object
* **onReady** - onReady callback. Modbus Master object will be passed as first parameter

Example:
```js
var serialPort = new SerialPort("/dev/ttyUSB0", {
   baudrate: 2400
});

new modbus.Master(serialPort, function (master) {
    //call master function here
})
```

#### master.readHoldingRegisters(slave, start, length) -> promise
Modbus function read holding registers

* **slave** - slave address (1..247)
* **start** - start register for reading
* **length** - how many registers to read

**Returns promise** which will be fulfilled with array of data

Example:
```js
new modbus.Master(serialPort, function (master) {
    master.readHoldingRegisters(1, 0, 4).then(function(data){
        //promise will be fulfilled with parsed data
        console.log(data); //output will be [10, 100, 110, 50] (numbers just for example)
    }, function(err){
        //or will be rejected with error
        //for example timeout error or crc.
    })
})
```

#### master.writeSingleRegister(slave, register, value) -> promise
Modbus function write single register

* **slave** - slave address (1..247)
* **register** - register number for write
* **value** - int value

**Returns promise**

Example:
```js
new modbus.Master(serialPort, function (master) {
  master.writeSingleRegister(1, 2, 150);
}
```

#### master.writeMultipleRegisters(slave, start, array) -> promise
Modbus function write multiple registers.

Set the starting register and data array. Register from `start` to `array.length` will be filled with array data

* **slave** - slave address (1..247)
* **start** - starting register number for write
* **array** - array of values

**Returns promise**

Example:
```js
new modbus.Master(serialPort, function (master) {
  master.writeMultipleRegisters(1, 2, [150, 100, 20]);
}
```
