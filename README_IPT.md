<img width="1024" height="1024" alt="IPT" src="https://github.com/mpkottawa/ignitron-preset-tools/blob/main/Ignitron%20preset%20tools%20logo.png" />

# Ignitron Preset Tools

*Shoutout to stangreg for this great project https://github.com/stangreg/Ignitron *

This application will allow you to do 3 things:

1. A tool to back up loaded presets on the pedal over usb, converted to the format ignitron uses for presets, .json file.  

2. A tool to pull and save presets from the spark app with just a click,formatted to use with ignitron. The pedal is connected over usb to your computer, and at the same time connected to the app over bluetooth.

3. A tool selects and sorts presets, then updates your 2 preset files(presetlist.txt and uuidindex.txt) the ignitron uses to build its preset tree in the /data folder. it also creates a preset banks list in pdf form beside the other 2 files
---

this program was coded 100% with chatgpt. I wanted a way to backup presets with ignitron, and struggled with pulling data back from the esp32 and converting it.  i simply asked chatgpt if it could
convert extracted raw preset data into usable .json files .  Then I just kept asking for more.

INSTALLATION
---
Extract IGNITRON PRESET TOOLS to the root folder of your Ignitron project (/ignitron/ignitron preset tools/)  run Ignitron Preset Tools.exe.  It also includes 3 standalone .exe's(preset_picker.exe, preset_puller.exe, and app_scraper.exe) which are all integrated within Ignitron Preset tools.exe
---
FIRMWARE SETUP
---
To enable the Ignitron pedal to pull current pedal presets(by responding to the calls: LISTPRESETS and LISTBANKS), as well as stream presets from the app, two firmware files must be modified:

- `ignitron.ino` (main Ignitron folder) → **3 edits**  
- `SparkPresetControl.cpp` (in `/src` folder) → **1 edit**

You can either manually edit these files as described below, or replace them entirely with provided versions and update pedal-specific bits (pins, LEDs, display, etc.).

---

## A. Preset Pulling Configuration(3 edits)  
(edit the file: `/ignitron/ignitron.ino`)

### 1. Add this include at the top with the other libraries (around line 8):
```cpp
#include <LittleFS.h>
```

---

### 2. Add this line right below `void loop()` (around line 100):
```cpp
handleSerialCommands();   // so it will react to LISTPRESETS
```

---

### 3. Add the following block at the **end of the file** (should be around line 142):
```cpp
// === BEGIN: LISTPRESETS serial support =======================================

// Case-insensitive “.json” check
static bool hasJsonExt(const char *name) {
  if (!name) return false;
  size_t len = strlen(name);
  if (len < 5) return false;
  const char *ext = name + (len - 5);
  return ext[0] == '.' &&
         (ext[1] == 'j' || ext[1] == 'J') &&
         (ext[2] == 's' || ext[2] == 'S') &&
         (ext[3] == 'o' || ext[3] == 'O') &&
         (ext[4] == 'n' || ext[4] == 'N');
}

// Dump entire JSON file to a single line (removes CR/LF/TAB)
static void printJsonFileSingleLine(File &f) {
  Serial.print("JSON STRING: ");
  while (f.available()) {
    char c = (char)f.read();
    if (c == '\r' || c == '\n' || c == '\t') continue;
    Serial.write(c);
  }
  Serial.println();
}

// List every *.json at the LittleFS root
static void listAllPresets() {
  Serial.println("LISTPRESETS_START");

  File root = LittleFS.open("/");
  if (!root) {
    Serial.println("⚠️ Could not open LittleFS root");
    Serial.println("LISTPRESETS_DONE");
    return;
  }

  while (true) {
    File f = root.openNextFile();
    if (!f) break;

    if (!f.isDirectory()) {
      const char *name = f.name();
      if (name && hasJsonExt(name)) {
        Serial.print("Reading preset filename: ");
        Serial.println(name);
        printJsonFileSingleLine(f);
      }
    }
    f.close();
  }

  Serial.println("LISTPRESETS_DONE");
}

// Robust line-buffered serial command reader
static void handleSerialCommands() {
  static String buf;

  while (Serial.available()) {
    char c = (char)Serial.read();
    if (c == '\r') continue;

    if (c == '\n') {
      String cmd = buf;
      buf = "";
      cmd.trim();
      if (cmd.length() == 0) return;

      String u = cmd;
      u.toUpperCase();

      if (u == "LISTPRESETS") {
        listAllPresets();
      }
      if (u == "LISTBANKS") {
        File f = LittleFS.open("/PresetList.txt");
        if (f) {
          Serial.println("LISTBANKS_START");
          while (f.available()) {
            char c = f.read();
            if (c == '\r') continue;
            Serial.write(c);
          }
          Serial.println("LISTBANKS_DONE");
          f.close();
        } else {
          Serial.println("⚠️ PresetList.txt not found");
          Serial.println("LISTBANKS_DONE");
        }
      }
    } else {
      buf += c;
      if (buf.length() > 256) {
        buf.remove(0, buf.length() - 256);
      }
    }
  }
}

// === END: LISTPRESETS serial support =========================================
```

---

## B. Spark App Streaming Configuration(1 edit)  
(edit `/ignitron/src/SparkPresetControl.cpp`)

Add the following lines **after** `DEBUG_PRINTLN(appReceivedPreset_.json.c_str());` (should be around line 400):

```cpp
// 🔧 Added for App Scraper
Serial.println("received from app:");
Serial.println(appReceivedPreset_.json.c_str());
```

### Final snippet should look like:
```cpp
void SparkPresetControl::updateFromSparkResponseAmpPreset(char *presetJson) {
    presetEditMode_ = PRESET_EDIT_STORE;
    appReceivedPreset_ = presetBuilder.getPresetFromJson(presetJson);

    DEBUG_PRINTLN("received from app:");
    DEBUG_PRINTLN(appReceivedPreset_.json.c_str());

    // 🔧 Added for App Scraper
    Serial.println("received from app:");
    Serial.println(appReceivedPreset_.json.c_str());

    presetNumToEdit_ = 0;
}
```


---

## C. optional- AMP Mode Toggle Switch Configuration(2 edits)

I found it a pain to always hold switch 1 to enter AMP mode.  use this mod to automatically handle it.  To add a toggle switch for AMP mode (spst 2 way rocker switch),\:

connect one side of the switch to gpio pin 35 of the esp32. On that same pin, put a 10k ohm resistor in series with a wire to 3.3volts on the board.  the other side of the switch top ground.
---

  to enable allowing a rocker switch to activate AMP mode any time it boots up, modify ignitron firmware by modifying one file in 2 spots, ignitron.ino in the root folder (/ignitron/ignitron.ino).  

1.
add this after the first string of includes in ignitron.ino:
---

```
#ifndef AMP_MODE_SWITCH_PIN
#define AMP_MODE_SWITCH_PIN 34    // <- change to the GPIO you wired; SPST to GND
#endif
```
----------------------------------------------------------------

2.
in /ignitron/ignitron.ino, at around line 59, the next line after: 
       
```
operationMode = spark_bh.checkBootOperationMode(); 
```

add the following:

```
// --- Amp Mode toggle on GPIO35 ---
pinMode(AMP_MODE_SWITCH_PIN, INPUT);  // external pull-up to 3.3V
int _ampToggleState = digitalRead(AMP_MODE_SWITCH_PIN);  // HIGH=open, LOW=closed
if (_ampToggleState == LOW) {
    operationMode = SPARK_MODE_AMP;   // force Amp Mode
    Serial.println("Amp toggle ON → forcing AMP mode");
} else {
    Serial.println("Amp toggle OFF → normal boot");
}
```

---------------------------------------------------------------


---

## Building firmware:

After editing, clean, build, flash to esp32, build filesysyem, upload filesystem, and reset.  
Enjoy!

---‐---------------------------------------------------------------
==========================================
---
---

# Ignitron Preset Tools Operation


**this tool will only pull presets in amp mode. you'll have to hold preset 1 on any boot.  I have installed and programmed an amp mode toggle switch, enabling it to boot 
to amp mode automatically by leaving the switch on.  see installation section for setup.


## Overview
- Ensure your pedal connects to your device running the Spark app (BLE or SRL). I have only tested this over SRL. 

- Preset pulling only works when the pedal is connected via USB in **AMP mode** (hold switch 1 while booting). 

-before plugging in the usb, put the pedal in AMP mode(power it on while holding switch 1) 

- Keep switch 1 pressed while the tool starts its connection. The pedal will reboot, and you can release switch 1 after the screen cycles.  

---


### Main Menu
When you run `ignitron_preset_tools.exe`, you’ll see 4 options:

```
1. Preset Picker
2. Preset Puller
3. App Scraper
4. Exit
```

### 1. Preset Picker
**DESCRIPTION:**  
Load your `/data` folder of presets, organize them in the GUI, and export the 2 files needed to build your preset banks.
<img width="1914" height="1025" alt="Screenshot 2025-09-19 185020" src="https://github.com/user-attachments/assets/c4d7df5b-296b-4690-9aad-0d835d0a1afb" />

**OPERATION:**
it starts with asking for how many banks, select 1-30.

<img width="218" height="142" alt="Screenshot 2025-09-19 184915" src="https://github.com/user-attachments/assets/3f0d051a-cc6f-4793-a56b-81f4f25aa419" />

then asks for your data folder, point it to /ignitron/data/.

<img width="779" height="498" alt="Screenshot 2025-09-19 184957" src="https://github.com/user-attachments/assets/a641fc86-161d-4f8b-95d0-d70b108447b2" />

after it opens up, you can see all presets from that folder on the left. you can then drag each preset to where you want it, if you double click it will 
add that preset to the next available slot. you can add or delete banks.  a button to fill all empty slots with random presets, will fill with unused presets from 
the left , then fill the rest with all presets in random locations to fill the grid.

when you're happy with your setup, hit the export button to update the /data/presetlist.txt, and the /data/PresetListUUIDs.txt, ready to upload to the pedal.  
Recompile and upload to the pedal and all your banks are where you put them.

it makes a presetlist.txt file like:
<img width="361" height="722" alt="Screenshot 2025-09-19 211613" src="https://github.com/user-attachments/assets/eeb15bff-ed62-4fc0-a372-c4fce5920a67" />

and the PresetListUUIDs.txt file like:
<img width="690" height="684" alt="Screenshot 2025-09-19 211746" src="https://github.com/user-attachments/assets/d68d71a6-bfbc-4f44-9d12-d7c4d5dd2ffa" />

which are formatted correctly, and directly ready to upload to your pedal upon save.  this tool also exports a puff of your current preset layout for easy reference:



---

### 2. Preset Puller
**DESCRIPTION:**

use this tool to retrieve, convert, and save presets from your ignitron pedal to your computer.  It pulls your presets saved to preset banks on the pedal, or pull all presets 
in the entire system.

**OPERATION:**

preset puller starts and asks pull mode:

   1. Only presets in pedal's presetlist.txt

   2. All presets on the pedal


<img width="586" height="562" alt="Screenshot 2025-09-19 204658" src="https://github.com/user-attachments/assets/3024e33a-39b6-46ba-bebe-8c331e9ccfb9" />


 it then asks for your connection port, enter the corresponding number for that port

 
<img width="578" height="187" alt="Screenshot 2025-09-19 205323" src="https://github.com/user-attachments/assets/00ef57e0-18df-4fb8-9602-a4cd8bf34267" />


 at this point, hold down switch 1 on the pedal and hit enter, keep holding 1 until you see the screen reboot and be in AMP mode(shows BLE or SRL.
 it will display the results after.  


<img width="682" height="291" alt="Screenshot 2025-09-19 210222" src="https://github.com/user-attachments/assets/655a60e7-1a0f-451e-8e2d-8049cbae2f0b" />


it will then open the save folder which should be time stamped.


<img width="959" height="860" alt="Screenshot 2025-09-19 210209" src="https://github.com/user-attachments/assets/68e1246a-3708-424e-830d-f377f69218ba" />



all these can be moved to your data folder and used with preset picker to save them to the pedal.(although they were pulled from the pedal.




---

## 3. App Scraper
**Description:**  
this tool is my favourite.  you connect to the pedal over usb, connect the app to the pedal over bluetooth, it monitors for preset chitchat, converts 
and saves selected presets to your computer. click a preset in the app, it saves directly to your computer.

**OPERATION:**

!this has only been tested with SRL not BLE!

important- before you select option 3 and hit enter for app scraper, your pedal needs to be already connected and in AMP mode.
once again, this tool you have to be holding preset 1 as you hit enter so it will save presets properly.                        

start with the pedal booted in AMP mode and connect the usb cable.  
Press and hold switch 1, enter 3 in the main menu and then hit enter. it will connect over usb, restart and boot to AMP mode. you will see the screen reset, 
release switch 1 after it reboots. it is now streaming the usb connection, waiting to convert and save preset messages


<img width="1112" height="630" alt="Screenshot 2025-09-19 222040" src="https://github.com/user-attachments/assets/29be511e-2ec0-4d07-9bcc-ca0235bc36ca" />


now open the spark app on your device, and connect to ignitron pedal.

once connected, when you select any preset in the app, it reads the request, converts it and saves it as a proper .json file on your computer.  it will
save it to a timestamped preset folder inside /ignitron preset tools with all the saved presets of that session. in this instance i clicked on the preset 
bush machinehead in my saved presets in the app.  It will also save them as you are browsing the Tone cloud in the app!  heres what it reads and outputs 
when you select a preset in the app: 


<img width="1119" height="319" alt="Screenshot 2025-09-19 222249" src="https://github.com/user-attachments/assets/68a605e7-7b73-46e1-94e2-ff6ed4336e62" />
<img width="783" height="447" alt="Screenshot 2025-09-19 223034" src="https://github.com/user-attachments/assets/49df7074-5310-4780-bfda-4b8e1eb18db4" />
<img width="407" height="858" alt="Screenshot 2025-09-19 223217" src="https://github.com/user-attachments/assets/7ace1cac-c9b9-44fc-9554-700d7a142a5c" />



<img width="1024" height="1024" alt="IPT" src="https://github.com/mpkottawa/ignitron-preset-tools/blob/main/Ignitron%20preset%20tools%20logo.png" />



