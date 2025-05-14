# ender 3 firmware changes
This is an update of all the changes I have made to my Ender 3's within the makerspace. Not all of these changes are probably useful for a solo printer. First some back ground.
I work with 20 Ender 3's that are being used by all kinds of students with various levels of experience with them, some have honestly no idea what they are doing.
To stream line things, some big changes have been made such as not writing the EEPROM to the SD card this is because all of my cards keep changing printers so when printer 1 tried to use printer 2's EEPROM it would fail.
Because of that a lot of changes are hard coded in, including the ability for the printer to level before every print. Normally you would had G29 after G28 in your slicer, for us this is done in the firmware.
Also the EEPROM is still enabled because marlin 2.0/current will save the z-offset to the hardware instead of the EEPROM, but the setting still has to be on.
I'll try and orginaize this more but for now this is a copy paste of the text file I keep of changes.

----> General Changes <----
^
-> Configuration.h <-

These were disabled to save flash

DISABLE:
#define EEPROM_CHITCHAT
#define EEPROM_SETTINGS 
#define DISABLE_M503
#define PIDTEMP

The EEPROM can not be used sense the SD card is read only

CHANGE:
#define HEATER_0_MAXTEMP
All the temperature values here for heaters are changed to 230
#define BED_MAXTEMP :Change to: 75
#define CUSTOM_MACHINE_NAME : Changed to say "Ender 3"
#define DEFAULT_LEVELING_FADE_HEIGHT 10: Change to 20.0

ENABLE:
#define MPCTEMP
#define EEPROM_INIT_NOW

An average was taken for the temperatures to use starting with MPC_BLOCK_HEAT_CAPACITY in order

13.665f
0.2516f
0.1064f
0.1113f

-> Configuration_adv.h <-

ENABLE:
#define SD_IGNORE_AT_STARTUP
#define SDCARD_READONLY
#define NO_SD_AUTOSTART


-> menu_bed_leveling.cpp <-

Don't need to show the load settings menu

DISABLED:
ACTION_ITEM(MSG_LOAD_EEPROM, ui.load_settings);

Comment out
// Homed and leveling is valid? Then leveling can be toggled.
  //if (is_homed && is_valid) {
    //bool show_state = planner.leveling_active;
    //EDIT_ITEM(bool, MSG_BED_LEVELING, &show_state, _lcd_toggle_bed_leveling);
  //}

This will not show the toggle for bed leveling, not need for us

-> language_en.h <-

Changes the ready message to say online, instead of ready.

LSTR WELCOME_MSG = MACHINE_NAME _UxGT(" Online");

-> Bootscreen and Status Screen <-

Look in "_Bootscreen.h" and "_Statusscreen.h" follow the instructions to change the icons

----> Enabling BLtouch <----
^
-> Configuration.h <-

All of these settings are enabled or disable to allow for Bltouch

ENABLE:
#define USE_PROBE_FOR_Z_HOMING
#define BLTOUCH
#define AUTO_BED_LEVELING_BILINEAR
#define RESTORE_LEVELING_AFTER_G28
#define BILINEAR_SUBDIVISIONS 3

DISABLE:
#define Z_MIN_PROBE_USES_Z_MIN_ENDSTOP_PIN
#define MIN_SOFTWARE_ENDSTOP_Z

CHANGES:
#define GRID_MAX_POINTS_X 3 :Change to: 5
#define NOZZLE_TO_PROBE_OFFSET { -45, -5, 2 } :Change to: { -45, -5, 0 }

----> Faster Leveling and Homing <----
^
-> Configuration.h <-

These are all changes that affect the probe while leveling to increase the speed, in various ways.

#define Z_PROBE_FEEDRATE_FAST (4*60) :Change to: (10*60)
DEFAULT_MAX_FEEDRATE : {500, 500, 5, 25} :Change to: {500, 500, 10, 25}
Z_CLEARANCE_BETWEEN_PROBES : 5 :Change to: 2
#define BLTOUCH_DELAY 500 :Change to: 1000
#define HOMING_FEEDRATE_MM_M { (100*60), (100*60), (10*60) } : Changed from the default values

-> Configuration_adv.h <-

Turning on High speed mode enable this setting.

#define BLTOUCH_HS_MODE true

-> bltouch.cpp <-

Setting HS mode to true by default, since this menu will be disabled find "bool BLToch::high_speed_mode" add "= true;" to the end of it.

bool BLTouch::high_speed_mode = true;


----> Disabling menus but not their values <----
^
-> menu_motion.cpp <-

This will turn off auto home in the motion menu

DISABLE:
GCODES_ITEM(MSG_AUTO_HOME, FPSTR(G28_STR));

-> menu_main.cpp <-

This will turn off configuration menu

DISABLE:
SUBMENU(MSG_CONFIGURATION
GCODES_ITEM(MSG_CHANGE_MEDIA, F("M21" TERN_(MULTI_VOLUME, "S")));

-> menu_bed_leveling.cpp <-

This will disable the fade height edit menu

DISABLE:
EDIT_ITEM_FAST(float3, MSG_Z_FADE_HEIGHT, &editable.decimal, 0, 100, []{ set_z_fade_height(editable.decimal); });

-> Configuration.h <-

DISABLE:
#define PID_EDIT_MENU
#define PID_AUTOTUME_MENU

ENABLE:
#define LCD_BED_LEVELING
#define SLIM_LCD_MENUS

-> Configuration_adv.h <-

DISABLE:
#define LCD_INFO_MENU
#define SHOW_ELAPSED_TIME 

ENABLE: 
#define SHOW_REMAINING_TIME
#define SHOW_PROGRESS_PERCENT
#define BROWSE_MEDIA_ON_INSERT
#define MEDIA_MENU_AT_TOP
#define BABYSTEPPING

----> Creating auto leveling before a print is started <----
^
-> Configuration.h <-

adding a new define variable, this will allow us to turn on or off the feature, this feature modifies the original G28 code. This line can be put anywhere I chose to be under "#define BLTOUCH"

#define LEVEL_BEFORE_PRINT

-> How to Add a gcode if you need too: gcode.h <-

adding the function of our new gcode to be called later. Place under "static void G28();"

#if ENABLED(LEVEL_BEFORE_PRINT)
	static void G228();
#endif

-> How to add a gcode if you need too: gcode.cpp <-

calling the gcode function so that the command can be used. Place under "case 28: G28(); break;"

#if ENABLED(LEVEL_BEFORE_PRINT)
	case 228: G228(); break;
#endif

-> How to add a gcode if you need too: platformio.ini <-

Now that the gcode has been made we have to let the compiler know to include the file. Place below "+<src/gcode/calibrate/G28.cpp>"

+<src/gcode/calibrate/G228.cpp>

-> bedlevel.h <-

We need to add custom functions for setting a true or false and getting the value of that. Place below "void reset_bed_level();". The function scope is defined within bedlevel.cpp

#if ENABLED(LEVEL_BEFORE_PRINT) //start of if
  void set_level_before_print_enabled(const bool enable=false); //sets value of bool default is false
  bool get_level_before_print(); //gets the value of bool
#endif //end of if

-> bedlevel.cpp <-

This is for creating the function scope and the Boolean variable. Place above "TemporaryBedLevelingState". The variable default state is set to true.

//Defined custom functions get and set
#if ENABLED(LEVEL_BEFORE_PRINT) //start of if
  static bool level_before = false; //this must have default value is the method is never called
  void set_level_before_print_enabled(const bool enabled) { //set bool value
    level_before = enabled;
  }
  bool get_level_before_print() { //returns bool value
    return level_before;
  }
#endif //end of if

-> How to add a gcode if you need too: G228.cpp <-

We need to make the Gcode now, either create a new file with this name "G228.cpp" or copy and paste the G28 file inside of the calibrate folder. Erase everything if you copied it, G228 needs to contains this.

/**
 * Calls G28, and disables leveling before printing
 */

#include "../../inc/MarlinConfig.h"

#include "../gcode.h"

#if HAS_LEVELING
  #include "../../feature/bedlevel/bedlevel.h"
#endif

void GcodeSuite::G228() {
  set_level_before_print_enabled(false);
  G28();
}

-> G28.cpp <-

Next we need to edit our G28 code. We are adding a call to G29 for leveling with a if statement that checks for the Boolean value. Add this at the very end but still in scope.

#if ENABLED(LEVEL_BEFORE_PRINT)
    if(get_level_before_print()) {
      G29(); //calls G29
    }

    set_level_before_print_enabled(false); //resets the value to false just incase
#endif

-> menu_bed_leveling.cpp <-

We need to make it so that the auto home button will set the level before to false, that way it doesn't level after we home it ourselves. Taking the auto home message out of the if statement will allow it to always appear so you can keep using it.

 #if NONE(PROBE_MANUALLY, MESH_BED_LEVELING)
    GCODES_ITEM(MSG_AUTO_HOME, FPSTR(G28_STR));
  #endif

-> menu_main.cpp <-

Change what how the stop command homes after selected
Add this somewhere at the top

#if ENABLED(LEVEL_BEFORE_PRINT)
  #include "src/feature/bedlevel/bedlevel.h"
#endif

Then under "#if MACHINE_CAN_STOP" add

#if ENABLED(LEVEL_BEFORE_PRINT)
      set_level_before_print_enabled(false);
#endif


-> menu_media.cpp <-

We now need to set the instance of level before print to true, that way G28 will call G29
Under "inline void sdcard_start_selected_file() {"
place

#if ENABLED(LEVEL_BEFORE_PRINT)
   set_level_before_print_enabled(true);
#endif

This will set it to true when you start a print. Now we add bedlevel.h to the top

#if ENABLED(LEVEL_BEFORE_PRINT)
  #include "src/feature/bedlevel/bedlevel.h"
#endif

---> Adding G1 to display the bed after stopping a print <---
^

In Configuration_adv.h add this under  #define EVENT_GCODE_SD_ABORT "G28XY" 
#define EVENT_GCODE_SD_ABORT_SHOW_Y "G1 Y235" //235 is the max y axis

Change  #define EVENT_GCODE_SD_ABORT "G28XY" to "G28X" this will not home the Y axis

In MarlinCore.cpp add this under queue.inject(F(EVENT_GCODE_SD_ABORT));
queue.inject((EVENT_GCODE_SD_ABORT_SHOW_Y));

Make sure not to use F this means to force the Gcode which will glitch the G28f

---> Disabled move extruder menu option <---

-> menu_motion.cpp <-
commented out SUBMENU(MSG_MOVE_E, _menu_move_distance_e_maybe);
This will turn off the move extruder menu item

