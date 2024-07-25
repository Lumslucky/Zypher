# ZYPHER AUTONOMOUS DRONE

## Overview

The Main goal of this project is to create a drone with the following requirements:

- Fully autonomous
- Waypoint programmable
- Small as possible

## Part Management

This Project consists of four main parts:

- Zypher Connect    (ZyCo)
- Zypher-Link       (ZyLi)
- Zypher Flight Unit (ZyFU)

### Zypher Connect

This separate software is deployed on user-machines with an app for the user to be able to either manually or automatically fly the vehicle.
It should enable the user to plan paths, add missions and waypoints, then deploy them correctly to the `Zypher-Link`.

### Zypher-Link

The Zypher is used to enable communication between user-device and the drone.
The used hardware is an ESP32-S3 with WIFI long range enabled, since the ESP32 Long Range is an espressif proprietary protocol, the end devices cannot establish communication and is therefore necessary to use a 'Zypher-Link' as a middle-man for data transmission.
The software is used to create a link between Zypher-Connect and the actual Flight Unit.
The main job of this module is to connect to a PC via USB and transmit data, WITHOUT any sort of modification or adjustment.

### Zypher Flight Unit

This unit is the main part of the project. The "Drone" itself! It's Job is to fly the drone according to the data received, which consists of flying, balancing, power-management and self-monitoring. The system should `ALWAYS` have a PERFECT and CLEAR understanding of itself and the environment surrounding (not including object detection).

The following data should always be present and taken into account during operation:

- GPS Location
- Altitude
- Geographical Orientation (Heading in Degree)
- Rotational Speed X / Y / Z - Aches
- Acceleration X / Y / Z - Aches
- Time
- Connection To Zypher-Link

## Procedures

### Initialization

The following lines describe the high level functioning of the devices and the communication between them.

`ALL` data coming from the Zypher-Connect is transferred to the Zypher-Link, which is then automatically transferred to the drone within 500 milliseconds, encrypted with AES encryption in JSON format, using the ESP32-S3 long range WIFI protocol.
Every data transmission is of type `TCP` for checking any communication error.

---

The Zypher-Connect is used for the configuration and navigation of the drone. The data is sent to the Zypher-Link via USB-Serial Communication in plain text, JSON formatting. This data includes all the configurations of the drone such as pre-programmed path, manual control parameters, altitude, SAFE-Location, etc...
Once all the has been configured, the drone can be put into take-off mode. Some initial tests have to be done before continuing the take-off procedure.
These tests include:

- GPS Location for cross-checking
- Heading for cross-checking
- Take-off altitude

---

- Battery State
- Battery Percentage
- Connection To Zypher-Link

The first checks MUST be done by the operating person to assure proper functioning.

Once the checks are done and everything returns an OK message, the procedure can continue. The software waits for a READ_FOR_TAKEOFF signal coming from the copter, which allows the software to initialize a takeoff.

The takeoff procedure then gets initiated.

### Takeoff

At this point the drone becomes completely autonomous, with no further communication allowed, other than an `Emergency Abort` signal, where the drone MUST immediately cancel the flight and cut the motors within 1 second, even if this results in the drone dropping to ground. This procedure follows till the drone levitates at 0.5m above the ground and stabilizes itself. Afterwards the Flight Mode initiates.

### Flight Mode

There are two types of flight modes.

- Manual Flight (Used mostly in emergency situations, but not meant for long flights)
- Autopilot Flight (Used the majority of the time)

The Manual Flight mode of course has a `HIGHER` piloting priority.
In both cases, once the [Flight Mode](#flight-mode) has been initiated, every packet sent has to include the current senders GPS location. (If existent)

#### Manual Flight

The manual flight mode's functioning principle relies on a joystick type of flight handling (can either be done via a digital joystick, like for apps OR can also be done using a keyboard). This mode gives control over the movement, rotation and altitude flight parameters.

#### Autopilot

The autopilot can control the drone in two ways:

- Pathway (This is a precise path which has been configured pre-flight in the Zypher-Connect app)
- Single Waypoint (A location is given which the drone has to reach)

Both can be reconfigured mid-flight and canceled mid-flight, as well as switched to manual control at any given time.

### Safe Spot

There is the option to pre-define a safe spot where the drone __`MUST`__ return to if any major issue occurs and the connection times out OR if there is no special instruction given to the ZyFU. This has to be a carefully pre-configured spot where the drone always has clear LOS of and there are no obstacles around it. If there is no safe spot explicitly defined, the take-off position is used. The Safe Spot cannot be reconfigured mid-flight or after the READY_FOR_TAKEOFF signal has been received.

### Home Spot

The home spot is another specifically defined location that is used when the user "sends the drone home", similar to the safe spot, except that this can be redefined any time, even mid home-run. This should only be used for easier back navigation purposes and __NOT__ for flying.

This location doesn't include altitude data, which means that the drone will stay at the same height for the entire home-run duration, unless manually intercepted. (This means that the drone will continue it's home-run, but the altitude can be varied at any time).

### Landing

The landing procedure can be triggered by two different occurrences.

- Manual Land Now
- Automatic Landing Waypoint reached

In both cases the drone will ONLY land after an extra landing request has been approved by the user. The ZyFU will first get into the precise latitudinal and longitudinal position, at about 1m above the ground (which is given by the map in the Zypher-Connect app | interventions are possible).

### After-Landing Procedure

The motors stop and the user has the ability to either start a new flight or shutdown the drone completely. After every flight, there is an automatic test report created, which allows for a better maintenance of the air-vehicle.

### Emergency Procedures

There are multiple emergency conditions and procedures, that have to be followed for a safe and normal operation. The goal is to have the drone be able to "live" fully on it's own, as far as event handling goes.

__The following pre-programmed emergency procedures are available in the latest firmware version.__ (Priority)

- Engine Stop (50)
- Sudden Altitude loss (30)
- GPS position loss (20)
- Connection loss (25)

And of course every possible combination has to be separately handled. This is done by a priority order, such that an engine fall-off is more of a problem, than the loss of GPS signal.

#### Engine Stop

With only one engine failing, the quadcopter is still able to maneuver without a problem, the only difference is that the quadcopter has to be in an angle to compensate for the lost motor. An immediate `Land Now` Procedure has to be done, before any more motors come to a stop as well. With the lost of two motors, the quadcopter is unable to fly anymore and the only thing left to do is to try to get to a human landing spot.

#### Sudden Altitude loss

In this procedure, the motors try to speed up as well, even taking control from the user to reduce the risk of slamming into the ground. This mode is present until the drone is idle in a stationary position for at least 5 seconds (If there is movement because of wind, that counts as stationary as well until a specific threshold). If this doesn't happen within 5 seconds off fast falling. It figures an undetected engine stop has happened and does the procedures for [Engine Stop](#engine-stop).

#### GPS position loss

If the GPS signal is lost, the drone has to immediately stop, no matter the operation as this can lead to flying into objects. It is important to note that the `Connection` between the Link and the FCU has to be available. If the connection is lost as well, the [Connection Lost](#connection-lost) procedure overrides this issue. Since the location is sent within every packet, the drone has a clear understanding of where the client is located at. So if an issue occurs, the user has to choose a position where the drone has to return to. This can either be the current location or the pre-defined safe-location. The drone will then try to calculate it's own position with the sensor data on the FCU and try to fly to the given position.

#### Connection Lost

If the connection between the FCU and the Link is interrupted, the Connection Lost procedure is ran. The main task of this mode is to try to regain connection to the Link. The user then `MUST` as well move in the direction of the drone, as the drone cannot be left unattained for more than 10 seconds. The FCU tries to fly the flown path precisely to get closer to the user. If the signal is not found after 10 seconds, the [Blocked](#blocked-mode) mode is initiated.

#### Blocked Mode

This mode is also called `Panic mode`. The connection has to be established as soon as possible.
The drone descends to 2 meters from the ground, turning on everything (Lights, Sounds, etc... if available) to able to detect the drone even in daylight. At this point both the FCU and the Zypher-Connect App are aware of the situation. The FCU stays stationary and advertises it's precise location every second on the long range protocol. This is done for 40 seconds, afterwards the drone Lands safely and shutdowns completely. The only way to get the drone out of this blocked mode, is by making a system restart. (Of course everything has to be reconfigured before being able to take off once again).

The position advertised, should help in determining the position of the drone if descends to a barely over-lookable place.

__This is a fully open-source project. If you have any questions contact me: example@example.com__  
__Feel free to contribute to the project!__

## About me

This project of mine has a really deep meaning to me and that is not because of what it can do or how much time I've been spending on this project, but rather to create something that I can be proud of and to see what one person alone can do (with smaller contributions and reviewers of course). But for a longer time I've been on and off projects and have never come as far as to actually complete something bigger other than the my [`MACROPAD`](#about-me) (but that was also only for a school project).

I was 18 years old when I started this project at the beginning of the summer vacation and mostly worked on it while traveling between my `Home` and the company where I was doing my `Internship` at.
