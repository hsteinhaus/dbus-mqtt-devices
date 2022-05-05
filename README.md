# dbus-mqtt-devices

*** IF YOU ARE INSTALLING ON CCGX PLEASE READ SECTION ON CCGX INSTALLATION ***

This Venus GX Driver works in concert with the [Victron dbus-mqtt gateway](https://github.com/victronenergy/dbus-mqtt). It 
allows Wi-Fi enabled devices (such as ESP32, some Arduino microcontrollers or Raspberry Pis) to self 
register to the dbus over MQTT. This avoids the need for additional dedicated 
custom drivers to be developed and deployed.

The following Victron dbus services are supported:
- temperature (com.victronenergy.temperature._device_)
- tank (com.victronenergy.tank._device_)
- pvinverter (com.victronenergy.pvinverter._device_)
- grid (com.victronenergy.grid._device_)

## Contents
1. [Install and Setup](#Install-and-Setup)
2. [The Registration Protocol](#Registration-Protocol)
3. [Design Notes](#Design-Notes)
4. [Troubleshooting](#Troubleshooting)
5. [To Do](#To-Do)
6. [Developers](#Developers)


## Install and Setup 

### Install and setup (not CCGX)

*** If you are installing on a CCGX device, please follow the CCGX specific instructions ***

To get the driver up and running, download the latest release from github and then run the setup script.

1. ssh into venus device (as root)

2. Download the latest zip from github and extract contents

```
$ mkdir -p /data/drivers
$ cd /data/drivers
$ wget -O dbus-mqtt-devices.zip https://github.com/freakent/dbus-mqtt-devices/archive/refs/tags/v0.5.1.zip
$ unzip dbus-mqtt-devices.zip
```

3. Run the set up script
```
$ ./dbus-mqtt-devices-0.5.1/bin/setup.sh
```

4. Check the contents of /data/rc.local to ensure dbus-mqtt-device automatically starts on reboot
```
$ cat /data/rc.local
ln -s /data/drivers/dbus-mqtt-devices-0.5.1/bin/service /service/dbus-mqtt-devices
```

5. Reboot (recommended)
```
$ reboot
```

### Install and Setup (only for CCGX)

*** Installation on CCGX devices has been known to cause the device to reboot, requiring manual re-installation of firmware to recover. This issue should be resolved now, but please ensure you are able to reset the CCGX device in case of issues. ***

To get the driver up and running, download the latest release from github and then run the setup script.

1. ssh into venus device (as root)

2. Download the latest zip from github and extract contents

```
$ mkdir -p /data/drivers
$ cd /data/drivers
$ wget -O dbus-mqtt-devices.zip https://github.com/freakent/dbus-mqtt-devices/archive/refs/tags/v0.5.1.zip
$ unzip dbus-mqtt-devices.zip
```

3. Run the set up script
```
$ ./dbus-mqtt-devices-0.5.1/bin/setup-ccgx.sh
```

4. Check the contents of /data/rc.local to ensure dbus-mqtt-device automatically starts on reboot
```
$ cat /data/rc.local
ln -s /data/drivers/dbus-mqtt-devices-0.5.1/bin/service /service/dbus-mqtt-devices
```

5. Reboot (recommended)
```
$ reboot
```


## The Registration Protocol
This driver uses a pair of MQTT topics under the "device/*" namespace to establish the 
registration, using the following protocol.  `<client id>` is a unique, short name you can use to identify the device (you MUST avoid using special characters ,.-/: in the client id). It is recommended (by not essential) that you use the same client ID during MQTT initialisation and connection.

1)  When a device initialises and EVERY time it connects to MQTT, it MUST do 2 things :

    1) subscribes to a topic `"device/<client id>/DBus"`.

	2) publishes a status message on the MQTT topic `"device/<client id>/Status"`. 
		
        The Status payload is a json object containing :
    	
        `{ "clientId": <client id>, "connected": <either 1 or 0>, "version": "<text string>", "services": [<a dictionary of services that this device wants to use>] }`
   	
        example 1:
		
        `{ "clientId": "fe001", "connected": 1, "version": "v1.0 ALPHA", "services": {"t1": "temperature"} }`
		In example 1, the device is registering that it is equipped with one temperature sensor which we are calling "t1". The label t1 is just an arbitrary identifier that distinguish one service from another within a device. The version field can contain any string you like and is displayed within the GX console and on VRM.

        example 2:
		
        `{ "clientId": "fe002", "connected": 1, "version": "v2.3", "services": {"t1": "temperature", "t2": "temperature", "tk1": "tank" } }`
		In example 2, the device is registering that it is equipped with two temperature sensors and a tank level sensor. The labels t1, t2, tk2 are the unique arbitrary identifiers that distinguish one service from another within a device. 

2)	The driver will then use this information to :
    - obtain a numeric device instance (for VRM) for each device service (using the [ClassAndVrmInstance](https://github.com/victronenergy/localsettings#using-addsetting-to-allocate-a-vrm-device-instance) dbus service), 
    - set up [local settings](https://github.com/victronenergy/localsettings) for persistent storage of some attributes
    - register the device on the dbus, 
    - set up the appropriate dbus paths for the service type (i.e. temperature sensor can provide Temperature, Pressure and Humidity)
    

3)	The driver publishes a DBus message under the same MQTT Topic
	namespace. This is the topic the device subscribed to in step 1.1. The 
	DBus message contains the numeric device instances (one for each 
	service) that the device should use when publishing messages for dbus-mqtt
	to process. It also contains the portal id needed to construct a dbus-mqtt topic (see 4). 
    
    For example:

		Topic: "device/<client id>/DBus"
		Payload: {"portalId": "<vrm portal id>", deviceInstance":"t1": 5, "t2":12}

	_Please note_: the original `device/<client id>/DeviceInstance` topic has been deprecated in favour of `device/<client id>/DBus`. Publishing to the DeviceInstance topic will be removed in a future release. By combining the `<portal id>` and `<device instance>` in the same message payload, client code will be simpler and it leaves scope for future expansion.


4)	The device uses the device instance to periodically publish messages to the 
	appropriate dbus-mqtt topics for the service they are providing. 
	
    For example:
	
    	Topic: "W/<portal id>/temperature/<device instance>/Temperature"
		Payload: { "value": 24.91 }


5) 	When a device disconnects it should notify the driver by publishing a 
	status message with a connected value of 0. With MQTT the preferred
	method of achieving this is through publishing an MQTT "last will" message.  
    
    For example:

		{ "clientId": "fe001", "version": "v1.0", "connected": 0, "services": {"t1": "temperature", "t2": "temperature"}}
	
    
    _please note_: on disconnect the contents of the "services" are actually irrelevant as all 
	the device services are cleared by this action.


## Design Notes

- 	The device can have multiple sensors of the same type (e.g. two 
	temperature sensors), each publishing to different dbus-mqtt topics as 
	different device services and unique Device Instance values.
- 	Each device service will appear separately on the Venus GX device, and 
	each can have a customised name that will show on the GX display and in 
	VRM.
- 	Currently this driver supports four services but the 
	protocol and the driver have been designed to be easily extended for 
	other services supported by dbus-mqtt (see [services.yml](https://github.com/freakent/dbus-mqtt-devices/blob/main/services.yml)).
-  client devices should always self register (by sending sending a Status message with connected = 1) on connecting to MQTT. Re-registering an already registered device has no adverse affect. 
-   A working Arduino Sketch (for Arduino Nano 33 IOT) that publishes temperature readings from an 
    Adafruit AHT20 temperature and humidity module using this driver and 
    mqtt-dbus is available at https://github.com/freakent/mqtt_wifi_sis
	

## Troubleshooting
1) First thing to check is that the dbus-mqtt-devices service is running, from the ssh command line use
```
$ svstat /service/dbus-mqtt-devices
```
More info on deamontools that VenusOs uses here: https://cr.yp.to/daemontools.html

2) If the service is not running then ensure that your rc.local script has execute permissions.
```
$ ls -l /data/rc.local
...
$ chmod +x /data/rc.local
```
3) If the service is running, then next thing to check is the log with the command:
```
$ more /var/log/dbus-mqtt-devices/current
```
It should contain something like this:
```
@400000006238ead134c233e4 INFO:device_manager:Received device status message {'clientId': 'fe001', 'connected': 1, 'version': 'v1.0', 'services': {'t1': 'temperature'}}
@400000006238ead134c25324 INFO:device:**** Registering device: fe001, services: {'t1': 'temperature'} ****
@400000006238ead134c25edc INFO:device:Registering Service temperature for client fe001
@400000006238ead134c26a94 INFO:device_service_config:About to open config file
@400000006238ead136d95fcc INFO:device_service:Unregistered mqtt_fe001_t1 from dbus
@400000006238ead136df10d4 INFO:device_service:Unregistered mqtt_fe001_t1 from dbus
@400000006238ead136ea9ddc INFO:device_service:Unregistered mqtt_fe001_t1 from dbus
@400000006238ead13755bbbc INFO:device_service:Registering service temperature for client fe001 at path com.victronenergy.temperature.mqtt_fe001_t1
@400000006238ead13903b20c INFO:settingsdevice:Setting /Settings/Devices/mqtt_fe001_t1/ClassAndVrmInstance does not exist yet or must be adjusted
@400000006238ead13a94dd44 INFO:vedbus:registered ourselves on D-Bus as com.victronenergy.temperature.mqtt_fe001_t1
@400000006238ead13ac572c4 INFO:device_service:Registered Service com.victronenergy.temperature.mqtt_fe001_t1 under DeviceInstance 1
@400000006238ead13ad8d79c INFO:device_manager:publish {'portalId': '<portal id>', 'deviceInstance': 't1': '1'} to device/fe001/DBus, status is 0
```

If you can have ssh open in another window, then
```
$ tail -f /var/log/dbus-mqtt-devices/current 
```
is a useful way to monitor the driver.

4) Finally, if you have re-installed more than once, make sure there is only one line in your rc.local for dbus-mqtt-devices.
```
$ more /data/rc.local 
```

5) I highly recommend using *MQTT-Explorer* (http://mqtt-explorer.com/) to monitor the queues while debugging and if you are doing anything with MQTT.


6) If you are still having a problem feel free to open an issue on the Github project here: https://github.com/freakent/dbus-mqtt-devices/issues
I get email alerts from Github which I don't seem to get from the Victron community forum.


## To Do
1) Use of command line args
2) Add support for more dbus-mqtt services
3) Automatically comment out old lines in /data/rc.local when new version installed


## Developers
if you are wanting to run the pytests on macos you need to install a few dependencies:

#### using homebrew
```
$ brew install dbus pygobject3 gtk+3
$ pip3 install pytest
$ pytest --ignore=ext
```
