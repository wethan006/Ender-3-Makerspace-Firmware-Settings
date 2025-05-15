# Ender 3 Firmware Changes Running Marlin 2.1.2.5
This is an update of all the changes I have made to my Ender 3's within the makerspace. Not all of these changes are probably useful for a solo printer. First some background.
I work with 20 Ender 3's that are being used by all kinds of students with various levels of experience with them, some have honestly no idea what they are doing.
To stream line things, some big changes have been made such as not writing the EEPROM to the SD card this is because all of my cards keep changing printers so when printer 1 tried to use printer 2's EEPROM it would fail.
Because of that a lot of changes are hard coded in, including the ability for the printer to level before every print. Normally you would add G29 after G28 in your slicer, for us this is done in the firmware.
Also the EEPROM is still enabled because marlin 2.0/current will save the z-offset to the hardware instead of the EEPROM, but the setting still has to be on until I can figure it out.
I'll try and orginaize this more but for now this is a copy paste of the text file I keep of changes. I used visual studio code to go through the files I won't be explaining that here.

You can download marlin here -> https://marlinfw.org/meta/download/

# General Changes
### Configuration.h
These settings were disabled to save flash sense the EEPROM won't be used

DISABLE:
```cpp
#define EEPROM_CHITCHAT

#define EEPROM_SETTINGS 

#define DISABLE_M503
```
Disabled pid sense I chose to use MPCTEMP note that this will require you run the auto tune for your printer

DISABLE:
```cpp
#define PIDTEMP
```
I had to change all the max tempatures because students where using higher temps then needed an causing problems

CHANGE:

All the temperature values here for heaters are changed to 230
```cpp
#define HEATER_0_MAXTEMP

#define BED_MAXTEMP CHANGED TO: 75
```
I also changed the machine name to just Ender 3 these can be anything
```cpp
#define CUSTOM_MACHINE_NAME CHANGED TO: "Ender 3"
```
I increased the fade height I was noticing this helped with some of the longer printer that take up more space on the bed
```cpp
#define DEFAULT_LEVELING_FADE_HEIGHT CHANGED TO: 20.0
```
ENABLE:
```cpp
#define MPCTEMP

#define EEPROM_INIT_NOW
```
An average from four printers was used for the temperatures to use starting with MPC_BLOCK_HEAT_CAPACITY in order

13.665f

0.2516f

0.1064f

0.1113f

### Configuration_adv.h

ENABLE:

Found that our SD card on startup was causing issues some times because of blackouts
```cpp
#define SD_IGNORE_AT_STARTUP
```
SD card set to read only and no autostart both for flash and because again the EEPROM is not being used
```cpp
#define SDCARD_READONLY

#define NO_SD_AUTOSTART
```

### menu_bed_leveling.cpp

I don't need to show the load settings menu

DISABLED:
```cpp
ACTION_ITEM(MSG_LOAD_EEPROM, ui.load_settings);
```
This will disable the toggle for bed leveling, not need for us

DISABLE THIS SECTION OF CODE:
```cpp
if (is_homed && is_valid) {
bool show_state = planner.leveling_active;
EDIT_ITEM(bool, MSG_BED_LEVELING, &show_state, _lcd_toggle_bed_leveling);
}
```

### language_en.h

Changes the ready message to say online, instead of ready. I just don't like that it days ready lol
```cpp
LSTR WELCOME_MSG = MACHINE_NAME _UxGT(" Online");
```
### Bootscreen and Status Screen

Look in "_Bootscreen.h" and "_Statusscreen.h" follow the instructions to change the icons I don't have an example but I changed my to show my companies logo just a cool detail, gave me brownie points with the boss.

# Enabling BLtouch

### Configuration.h

All of these settings are enabled or disable to allow for Bltouch to work some improve how well it works, you do not need to change the grid point to 5 if you don't want a 25 point grid, but because of speed changes I make later 25 is still pretty fast.

ENABLE:
```cpp
#define USE_PROBE_FOR_Z_HOMING

#define BLTOUCH

#define AUTO_BED_LEVELING_BILINEAR

#define RESTORE_LEVELING_AFTER_G28

#define BILINEAR_SUBDIVISIONS 3
```
DISABLE:
```cpp
#define Z_MIN_PROBE_USES_Z_MIN_ENDSTOP_PIN

#define MIN_SOFTWARE_ENDSTOP_Z
```
CHANGES:
```cpp
#define GRID_MAX_POINTS_X CHANGED TO: 5

#define NOZZLE_TO_PROBE_OFFSET CHANGED TO: { -45, -5, 0 }

# Faster Leveling and Homing
```
### Configuration.h

These are all changes that affect the probe while leveling to increase the speed, while keeping accuracy and I increased how fast the printer will home.
```cpp
#define Z_PROBE_FEEDRATE_FAST (10*60)

DEFAULT_MAX_FEEDRATE : {500, 500, 10, 25}

Z_CLEARANCE_BETWEEN_PROBES : 2

#define BLTOUCH_DELAY 1000

#define HOMING_FEEDRATE_MM_M CHANGED TO: { (100*60), (100*60), (10*60) }
```
### Configuration_adv.h

Turning on High speed mode enable this setting.
```cpp
#define BLTOUCH_HS_MODE true
```
### bltouch.cpp

Setting HS mode to true by default, since this menu will be disabled find "bool BLToch::high_speed_mode" add "= true;" to the end of it.
```cpp
bool BLTouch::high_speed_mode = true;
```

# Disabling menus but not their values

### menu_motion.cpp

This will turn off auto home in the motion menu but not in the leveling menu which I will be using

DISABLE:
```cpp
GCODES_ITEM(MSG_AUTO_HOME, FPSTR(G28_STR));

SUBMENU(MSG_MOVE_E, _menu_move_distance_e_maybe);
```
### menu_main.cpp

This will turn off configuration menu

DISABLE:
```cpp
SUBMENU(MSG_CONFIGURATION

GCODES_ITEM(MSG_CHANGE_MEDIA, F("M21" TERN_(MULTI_VOLUME, "S")));
```
### menu_bed_leveling.cpp

This will disable the fade height edit menu on the printer screen

DISABLE:
```cpp
EDIT_ITEM_FAST(float3, MSG_Z_FADE_HEIGHT, &editable.decimal, 0, 100, []{ set_z_fade_height(editable.decimal); });
```
### Configuration.h

These might be disabled from turning off PID I don't remember I did this part first before I switched to MPCTEMP

DISABLE:
```cpp
#define PID_EDIT_MENU

#define PID_AUTOTUNE_MENU
```
ENABLE:
```cpp
#define LCD_BED_LEVELING

#define SLIM_LCD_MENUS
```
### Configuration_adv.h

DISABLE:
```cpp
#define LCD_INFO_MENU

#define SHOW_ELAPSED_TIME 

#define BABYSTEPPING

#define SHOW_PROGRESS_PERCENT 
```
ENABLE: 
```cpp
#define SHOW_REMAINING_TIME

#define BROWSE_MEDIA_ON_INSERT

#define MEDIA_MENU_AT_TOP
```
# Creating auto leveling before a print is started

This is how I got around needing to imput G29 into the slicer, again probably not needed for almost all applications that aren't mine lol

### Configuration.h 

adding a new define variable, this will allow us to turn on or off the feature incase you need to without removing all the code, this feature modifies the original G28 code. This line can be put anywhere I chose to be under "#define BLTOUCH"
```cpp
#define LEVEL_BEFORE_PRINT
```
### bedlevel.h 

We need to add custom functions for setting a true or false and getting the value of that. Place below "void reset_bed_level();". The function scope is defined within bedlevel.cpp

```cpp
#if ENABLED(LEVEL_BEFORE_PRINT) //start of if
	void set_level_before_print_enabled(const bool enable=false); //sets value of bool default is false
	bool get_level_before_print(); //gets the value of bool	
#endif //end of if
```

### bedlevel.cpp

This is for creating the function scope and the Boolean variable. Place above "TemporaryBedLevelingState". The variable default state is set to false.

```cpp
	//Defined custom functions get and set
	#if ENABLED(LEVEL_BEFORE_PRINT) //start of if
	  static bool level_before = false; //this must have default value if the method is never called as a fail safe
	  void set_level_before_print_enabled(const bool enabled) { //set bool value
	    level_before = enabled;
	  }
	  bool get_level_before_print() { //returns bool value
	    return level_before;
	  }
	#endif //end of if
```

### G28.cpp

Next we need to edit our G28 code. We are adding a call to G29 for leveling with a if statement that checks for the Boolean value. Add this at the very end but still in scope.

```cpp
	#if ENABLED(LEVEL_BEFORE_PRINT)
	    if(get_level_before_print()) {
	      G29(); //calls G29
	    }
	
	    set_level_before_print_enabled(false); //resets the value to false just incase
	#endif
```

### menu_bed_leveling.cpp

We need to change our auto home button so it always appears allowing us to use it. Taking the auto home message out of the if statement will allow it to always appear so you can keep using it. It will look like this you can find it by searching for the top "#if NONE(PROBE_MANUALLY, MESH_BED_LEVELING)". 

```cpp
	 #if NONE(PROBE_MANUALLY, MESH_BED_LEVELING)
	    GCODES_ITEM(MSG_AUTO_HOME, FPSTR(G28_STR));
	  #endif
```

### menu_main.cpp

Changes the stop command to made our boolean value false

Add this somewhere at the top of the file

```cpp
	#if ENABLED(LEVEL_BEFORE_PRINT)
	  #include "src/feature/bedlevel/bedlevel.h"
	#endif
```

Then under "#if MACHINE_CAN_STOP" add

```cpp
	#if ENABLED(LEVEL_BEFORE_PRINT)
	      set_level_before_print_enabled(false);
	#endif
```


### menu_media.cpp

We now need to set the instance of our boolean value to true so that G28 will call G29
Inside the scope of "inline void sdcard_start_selected_file()" add this line, when this method is called which happens when you click print it will change our boolean value to true.

```cpp
	#if ENABLED(LEVEL_BEFORE_PRINT)
	   set_level_before_print_enabled(true);
	#endif
```

This will set it to true when you start a print. Now we add bedlevel.h to the top of the file. Without this the file can't use our method

```cpp
	#if ENABLED(LEVEL_BEFORE_PRINT)
	  #include "src/feature/bedlevel/bedlevel.h"
	#endif
```

# Adding G1 to display the bed after stopping a print 

### Configuration_adv.h
Under this line #define EVENT_GCODE_SD_ABORT "G28XY"  add

```cpp
	#define EVENT_GCODE_SD_ABORT_SHOW_Y "G1 Y235" //235 is the max y axis
```

CHANGE:
```cpp
#define EVENT_GCODE_SD_ABORT "G28XY" to "G28X" this will not home the Y axis
```
### MarlinCore.cpp
Under queue.inject(F(EVENT_GCODE_SD_ABORT)); add

```cpp
	queue.inject((EVENT_GCODE_SD_ABORT_SHOW_Y));
```

Make sure not to use F this means to force the Gcode which will glitch the G28 call, now technically you could not use the G28X and just use G1 but I found this to be glitchy for some reason and sometimes the command wouldn't even run.

# Bonus How to add a custom Gcode if you needed to

I did this the first time when I was trying to figure out how to make a boolean method work. First were G28 is stored inside of the calibrate folder create a new blank file called G228.cpp (G228 is the example Gcode name it could be anything you want)

### gcode file

You'll need some things in the file for it to have basic function. add

```cpp
	#include "../../inc/MarlinConfig.h"
	
	#include "../gcode.h"
	
	void GcodeSuite::G228() {
	}
```

### gcode.h

adding the function of our new gcode to be called later. Place under "static void G28();" for this example I call the Gcode G228

```cpp
	#if ENABLED(LEVEL_BEFORE_PRINT)
		static void G228();
	#endif
```

### gcode.cpp

Creating the ability to call it. Place under "case 28: G28(); break;"

```cpp
	#if ENABLED(LEVEL_BEFORE_PRINT)
		case 228: G228(); break;
	#endif
```

### platformio.ini

Now that the gcode has been made we have to let the compiler know to include the file. Place below "+<src/gcode/calibrate/G28.cpp>"

```cpp
	+<src/gcode/calibrate/G228.cpp>
```
