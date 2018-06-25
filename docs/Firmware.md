# Firmware

## Contents

- [A word on the GPLv3](#a-word-on-the-gplv3)
- [Obtaining the firmware](#obtaining-the-firmware)
- [Firmware overview](#firmware-overview)
	- [Configuration.h](#configurationh)
	- [pins_RAMPS.h](#pinsrampsh)
	- [Marlin_main.cpp](#marlinmaincpp)

## A word on the GPLv3

The DSP Informatik AG 3D printer operates on a fork of the [open source Marlin Firmware](http://marlinfw.org/), which falls under the terms of the **GPLv3** license. As a result, it is an **obligation** to provide a copy of the firmware source code with the device running it. Our fork of the Marlin firmware can be viewed and obtained at our public GitHub organization [here](https://github.com/DSP-Informatik-AG/Marlin).

> **Any changes done to the firmware must be published! Refusing in doing so is an of the GPLv3 and will ultimately damage the legacy of our printer!**
  When the printer was first built, its maintainers have violated the GPLv3 by not publishing the source code for the firmware. At that time, it was considered of little importance to publish the firmware code, which was quickly proven wrong when the source code was lost with the machine it was stored on. Due to this violation, the printer could no longer be calibrated without reconfiguring the firmware from scratch, which resulted in a total abandonment of the device. While the original firmware remains lost till this day, it was only after two years later that our team has decided to reconfigure a marlin fork from scratch and learn from the mistakes of the previous maintainers.

## Obtaining the firmware

To obtain a local copy of the printer firmware, clone our Marlin fork from GitHub:

```
git clone https://github.com/DSP-Informatik-AG/Marlin
```

## Firmware overview

The Marlin source code is by far too big and complex to be summarized in our documentation. We will only focus on the changes that we've made and the parts required to make simple modifications.

For a good understanding of the Marlin firmware we highly suggest to take a glance at the following pages:

- [RepRap - Calibrating your 3D printer](https://www.reprap.org/wiki/Calibration)
- [Official Marlin site](http://marlinfw.org/)
- [Official Marlin Documentation](http://marlinfw.org/docs/configuration/configuration.html)

For the best understanding of the Marlin firmware we highly suggest to view its source code, which is guided by very descriptive comments.

- [Official Marlin Repository](https://github.com/MarlinFirmware/Marlin)
- [DSP Informatik AG Marlin Fork](https://github.com/DSP-Informatik-AG/Marlin)

All source code for the Marlin firmware can be found under the [Marlin](https://github.com/DSP-Informatik-AG/Marlin/tree/1.1.x/Marlin) directory.

#### Configuration.h

The [Configuration.h](https://github.com/DSP-Informatik-AG/Marlin/blob/1.1.x/Marlin/Configuration.h) header provides a list of parameters that describe the 3D printer to the firmware, so that It knows which peripherals to expect, what their limits are, and control them. Most configurations are correct by default, as the majority of printers share functionality and components. Flags and parameters such as the printer board, bed size, end stops, motor steps per mm, temperature sensors and probes are defined in the configuration header. Most modifications done to the printer Firmware will only require changes in the configuration header.

Most parameters are self explanatory. For example, the following parameters define the printers bed size:

```
// The size of the print bed
#define X_BED_SIZE 195
#define Y_BED_SIZE 195
```

A history of changes applied to the configuration header can be found [here](https://github.com/DSP-Informatik-AG/Marlin/commits/1.1.x/Marlin/Configuration.h)

#### pins_RAMPS.h

The [pins_RAMPS.h](https://github.com/DSP-Informatik-AG/Marlin/blob/1.1.x/Marlin/pins_RAMPS.h) header specifies the Arduino pins used to communicate and control printer hardware via the [RAMPs 1.4 board](https://reprap.org/wiki/RAMPS_1.4). Re-assigning pins is typically not required if the printer has been built according to the official ramps manual. However, as we're dealing with only a limited choice of hardware, clone devices and extraordinarily lazy people, the temperature sensors have been falsely wired up.

The following change defines the pins according to our hardware setup:

```diff
diff --git a/Marlin/pins_RAMPS.h b/Marlin/pins_RAMPS.h
index 9b10751e5..005cd6d09 100644
--- a/Marlin/pins_RAMPS.h
+++ b/Marlin/pins_RAMPS.h
@@ -113,7 +113,6 @@
 #define E1_ENABLE_PIN      30
 #define E1_CS_PIN          44

-
 #if ENABLED(HAVE_TMC2208)
   /**
    * TMC2208 stepper drivers
@@ -168,8 +167,8 @@
 // Temperature Sensors
 //
 #define TEMP_0_PIN         13   // Analog Input
-#define TEMP_1_PIN         15   // Analog Input
-#define TEMP_BED_PIN       14   // Analog Input
+#define TEMP_1_PIN         14   // Analog Input
+#define TEMP_BED_PIN       15   // Analog Input

 // SPI for Max6675 or Max31855 Thermocouple
 #if DISABLED(SDSUPPORT)
```

#### Marlin_main.cpp

The [Marlin_main.cpp](https://github.com/DSP-Informatik-AG/Marlin/blob/1.1.x/Marlin/Marlin_main.cpp) defines the fimrware entry point and its main program loop. At the time of writing the file consists of more than 14 000 lines of code. For obvious reasons, only modifications done to the code are summarized.

Because our Z end point has been placed slightly too low, the nozzle would collide with the bed during prints. To make up for this hardware flaw, we have simply hacked the homing code to allow us to define an offset from the original homing position in the configuration header.

When the Z Axis is being homed by the `G28` command, it performs a fast movement until it hits the Z endpoint. Then it performs, what is specified in the firmware as a "bump", where the nozzle is quickly lifted by a few mm, and then slowly moved down by the same distance (see Fig. 1).

The following hack increases the distance by witch the Z axis moves upwards, without having an affect on the slow "down" movement (See Fig. 2):

```diff
diff --git a/Marlin/Marlin_main.cpp b/Marlin/Marlin_main.cpp
index 4f4271a09..6a29c63e6 100644
--- a/Marlin/Marlin_main.cpp
+++ b/Marlin/Marlin_main.cpp
@@ -2984,7 +2984,7 @@ static void homeaxis(const AxisEnum axis) {
     #if ENABLED(DEBUG_LEVELING_FEATURE)
       if (DEBUGGING(LEVELING)) SERIAL_ECHOLNPGM("Home 2 Slow:");
     #endif
-    do_homing_move(axis, 2 * bump, get_homing_bump_feedrate(axis));
+    do_homing_move(axis, 2 * bump + (axis == Z_AXIS ? Z_HOMING_OFFSET : 0), get_homing_bump_feedrate(axis));
   }

   /**
```

To define the nozzle offset from its actual position, modify the following parameter in [Configuration.h](https://github.com/DSP-Informatik-AG/Marlin/blob/1.1.x/Marlin/Configuration.h):
```
// [CUSTOM MODIFICATION!]
// Z distance (mm) from initial homing point: +Up | -Down
+#define Z_HOMING_OFFSET 2.4
```

Where `2.4` is currently the offset in mm.
