# A Highly-Configurable and Ultra-Low Power ESP8266 Temperature Sensor for ThingSpeak
This project came about as a method of deploying multiple temperature sensors around my home in order to gather long-term data and determine which rooms have the best and worst thermal efficiencies.  It uses the [Dallas DS18B20 temperature sensor](https://datasheets.maximintegrated.com/en/ds/DS18B20.pdf) and reports its readings on a configurable interval to [ThingSpeak](https://thingspeak.com/) via a simple HTTP REST API implementation.
![banner](https://user-images.githubusercontent.com/20256822/142178795-50b34ecf-c55d-4fa2-8cea-da4c87b189b8.jpg)

Key features and concepts demonstrated in this project:

 - **Highly-configurable in-field** through captive portal interface
 - **Efficient REST API implementation** using ESP8266HTTPClient
 - **Super-short runtime** for optimal battery life
 - **Intelligent three-mode reporting interval** based upon rates of change or error
 - **Low-voltage condition alerting** using a single ThingSpeak field
 - **Optional debug data display** for in-field troubleshooting

### In-Field Configuration Options
The sensor is highly configurable in-field, using [WiFiManager](https://github.com/tzapu/WiFiManager) to present a captive portal-based configuration interface.  The captive portal is loaded when no condfiguration file is present or the "Mode" button is held while cycling power to the device.  The following options may be configured:

 - **Sensor ID:** A unique identifier to enable identification for error reporting
 - **API Host:** Hostname of the REST API endpoint (e.g. api.thingspeak.com)
 - **API Key:** Key for REST API access
 - **Temperature Field:** Index of temperature field
 - **Voltage Field:** Index of voltage field *(optional)*
 - **RSSI Field:** Index of RSSI field, useful for measuring signal strength *(optional)*
 - **Runlength Field:** Index of field for length of last run in milliseconds, useful for calculating expected battery life *(optional)*
 - **Error Field:** Index of field for reporting low voltage status, multiple sensors can report to this field *(optional)*
 - **Post Period:** Default reporting interval, in seconds
 - **Short Period:** High rate of change or error reporting interval, in seconds
 - **Hibernate Period:** Hibernate reporting interval, invoked after configured number of failures, in seconds
 - **Temperature Change Threshold:** Rate of change between two readings above which short reporting interval will be invoked *(optional)*
 - **Failure Limit Before Hibernation:** Number of consecutive errors after which hibernate interval will be invoked
 - **WiFi Timeout:** Timeout for Wi-Fi connection, in milliseconds
 - **HTTP Timeout:** Timeout for HTTP connection, in milliseconds
 - **IP Address / Subnet Mask / Gateway:** Static IP configuration options, configure for shorter runtimes *(optional)*
 - **Temperature Precision:** 1 - 4 corresponds to precision of 0.5°C, 0.25°C, 0.125°C, or 0.0625°C, higher values increase runtime and reduce battery life, 2 is your best compromise *(optional)*
 - **RTC Correction Factor:** Factor by which to correct sleep times (e.g 1.07).  Some ESP8266 modules run fast in deep sleep and may wake up several seconds earlier than expected *(optional)*
 - **Disable LED:** Disable LED during REST API request; leave enabled to give a visual representation of the HTTP request length.
 - **Additional Configuration Flags:** First 7 bits changes default number of cycles (10) between RF calibrations on wake, 8th bit enables debug mode *(optional)*

### Super-Short Runtime
The code represents the shortest possible runtime I could manage, to ensure the longest possible lifespan on battery.  In testing, all non-HTTP actions (including connecting to Wi-Fi) take around 200ms to complete - the remaining runtime is determined by the round trip time to your REST API endpoint.   This can be well under 100ms if you're in the US, which would yield years of runtime on 3 AAA cells with a suitable reporting interval.

### Three-Mode Reporting Interval
Three reporting intervals are configurable.  These intervals determine the time the device remains in deep sleep (minus runtime, so as to maintain a semi-regular reporting interval).  Here's when these intervals are active:

 - **Default:** Normal interval used unless any of the below applies.
 - **Short:** If the rate of change between the current and previous temperature reading is greater than the value configured for *'Temperature Change Threshold'*, or if the previous Wi-Fi/HTTP connection or REST API response returned an error.
 - **Hibernate:** If the number of successive error results exceeds the value configured for *'Failure Limit Before Hibernation'*.  This is intended to preseve battery life during persistent error conditions.

### Low Battery Condition Alerting
To avoid a sensor going offline due to a flat battery, each sensor can report a low voltage condition to a single ThingSpeak field.  As each ThingSpeak channel allows for only 8 sensors, reporting on each sensor's voltage limits the number of sensors we can deploy.

To work around this, each sensor is configured with a unique ID between 1 and 255, which it will report to the configured *"Error"* field on a low voltage condition.  This allows up to 7 sensors to report their status to a single field.  Each sensor will report a "0" to this field if low voltage condition reporting is enabled and the sensor voltage is sufficient.

### Optional Debug Data Display
By setting the 8th bit of the "Flags" field (e.g. 128), a debug mode is enabled.  Debug mode will display data from a file that is written on an error condition containing the following values:

 - Wi-Fi connection result
 -  HTTP status
 - API response
 -  Temperature
 - Configuration file size

This data is displayed at the bottom of the captive portal configuration screen.  Enabling debug mode will force the device to always boot to the captive portal; unsetting the 8th bit will revert to normal behaviour. 

The debug data file is not overwritten on a non-error result, so entering debug mode will always yield the last error state data.

Significant data is also written to Serial for debugging purposes.

### Circuit Implementation
A basic implementation of a circuit for this project is shown below.

![schematic](https://user-images.githubusercontent.com/20256822/142173821-8d602e9f-c3b9-4687-a9c3-1317feb7c48b.png)

A [Holtek HT7333 LDO 3.3v regulator](https://datasheet.lcsc.com/szlcsc/Holtek-Semicon-HT7333-A_C21583.pdf) is used for power; this is chosen for its ultra-low quiescent current (~7uA) and low-dropout (<0.1V) properties.  I have used tantalum capacitors; I found MLCC capacitors to yield unstable results.

A gerber file for a PCB is included.  The board accepts an ESP8266 using the widely-available breakout board.  This board will also accept a BME280/BMP280 sensor as shown in the above schematic; the 4k7 pullup resistor would be omitted in this configuration.  Support for this sensor has not been implemented in code; only the Dallas DS18B20 will function.  It is not possible to run both sensors simultaneously.

![tempmon-board-w-border](https://user-images.githubusercontent.com/20256822/142173054-af27affc-3281-4cdb-9967-e1d4a90a2816.jpg)
