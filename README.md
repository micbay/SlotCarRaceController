# **<span style="color:red;font-size:72px"> -- IN DRAFT -- </span>**

# **Arduino Race Timer and Game Controller**
This is an Arduino based project that implements a functional race game controller that can be used for timed racing games. In the present implementation it consists of a main LCD display with a keypad, for user input and menu selection, as well as an additional 8 digit LED lap timer display for each racer. Also with audio feedback and custom victory song for each racer.

In the presented configuration, the lap sensing input is simulated using buttons, but can be adapted to be used with a myriad of simple, circuit completion, or other type sensing methods that can be implemented in the physical lap gate.  
![MainMenu](Images/MainMenu.png)  
![Wiring Diagram](Images/Demolayout_Adjusted.jpg)  
![Wiring Diagram](Images/ArduinoNanopPinOut_RealHardware.png)  
![Wiring Diagram](Images/Displays_DemoStartup.png)
The software uses non-blocking techniques with port register interrupts to  but is a working reference that demonstrates how to interact with several common micro-controller peripherals in a non-blocking manne.

The implementation shown here is immediately useable for 2 player racing games such as would be used in a 1/64 slot car set or drone race. However, the hardware included can handle up to 4 racers. In several places in the code we take advantage of knowing that there are only 2 racers. These aspects of the code would need to be rafactored into an object or array with iterating loops. Something similiar to how the lanes/racers are enabled in this 2-player embodiment.

### **Prerequisites**  
This project requires almost no experience to execute, however it will not cover basic usage of Arduino. It is expected the reader understands how to use the Arduino IDE, connect wires, and program boards. To get up to speed on the basics, there are many great resources from [Arduino](https://www.arduino.cc/en/Guide) and around the web covering these topics exhaustively.

<br>

# **Hardware Configuration**  
All of the components are readily available and can be connected with basic jumper leads or simple conductor and header pin soldering.
> ***Note on Housing and Mechanical Interface** - This project only documents the functional electrical and software configuration. It can be wired, and used as illustrated for demonstration, however, for practical usage, the construction of a physical housing and the mechanical lane sensing interface are left up to the implementer to adapt to their specific use.*  
> ***Note on Reference Sources** - All links are for reference only and are not to be taken as an endorsement of any particular component supplier. I attempt to reference official Arduino resources whenever possible, but this is also not an endorsment for or against using the Arduino store.*

## **Parts for Race Controller**  
- [Arduino Nano](https://www.arduino.cc/en/pmwiki.php?n=Main/ArduinoBoardNano) (or equivalent microntroller module)
- [4 x 4 membrane keypad](https://duckduckgo.com/?q=4+x+4+membrane+keypad)
- [LCD2004 4 row x 20 character display](https://duckduckgo.com/?q=LCD2004A+4+x+20+I2C+backpack), with [I2C backpack](https://www.mantech.co.za/datasheets/products/LCD2004-i2c.pdf)
- 2 Chainable, [8-digit, 7-segment LED bar with integrated MAX7219](https://duckduckgo.com/?q=8-digit%2C+7-segment+LED+display)
- Active Buzzer (DC compatible) or speaker, 5V
- 2-4 lap sensors/switches/buttons
- 1 momentary button for in game pause and restart
- Jumper leads to wire connections between peripherals & Arduino

## **PinOut Diagram for Wiring Arduino Nano**
(these are the pins used by this code, but can be re-arranged, if desired)  
<div style="color:orange;font-size:12px">  

|Attached        |Pin  |Nano|Pin  |Attached        |
|---------------:|----:|:--:|-----|----------------|
|                |D13  |\-- |D12  |Keypad 8 (C4)   |
|                |3V3  |\-- |D11  |Keypad 7 (C3)   |
|                |Free |\-- |D10  |Keypad 6 (C2)   |
|Lane1 lap sensor|A0   |\-- |D9   |Keypad 5 (C1)   |
|Lane2 lap sensor|A1   |\-- |D8   |Keypad 4 (R4)   |
|Pause Button    |A2   |\-- |D7   |Keypad 3 (R3)   |
|Buzzer          |A3   |\-- |D6   |Keypad 2 (R2)   |
|LCD SDA         |A4   |\-- |D5   |Keypad 1 (R1)   |
|LCD SCL         |A5   |\-- |D4   |LED CLK         |
|                |A6   |\-- |D3   |LED CS          |
|                |A7   |\-- |D2   |LED DIN         |
|LCD 5V          |5V   |\-- |GND  |                |
|                |Reset|\-- |Reset|                |
|                |GND  |\-- |Rx0  |                |
|                |Vin  |\-- |Tx1  |                |
</div>  

![Wiring Diagram](Images/WiringDiagram1600x800.png)

## **Power Supply (+5V)**  
All devices in this build are powered from a +5V source. The displays should draw power from the source supply and not through the Arduino which cannot support enough current to run everything without flickering.

> ***<span style="color:yellow"> Connect Arduino GND to external ground reference</span>** - Since this project must use an external power supply to get enough current for the dipslays; during development and programming, we will often have the USB plugged in as well. If we do not connect the Arduino GND to the power supply ground we run the risk of a reference mismatch that can cause intermittent errors, or the device to not work at all.  
> If we were to find ourselves with the configuration shown here. It may appear to be ok on first look, but with the multi-meter we can see that there is a 1.9V differential between the two ground references when it should be around 0.*
> 
> ![GroundLoop HIGH](Images/GroundLoop_VoltageHIGH_Edit.png)
> Connecting our grounds to bring them to the same potential, as below, will eliminate the problem above.  
> 
> ![Connected Grounds](Images/GroundLoop_VoltageLOW.png)
 
<br>

# **Software Configuration**  
In order to interact with our different peripherals, we will be making use of several existing [Arduino libraries](https://www.arduino.cc/reference/en/libraries/). Unless otherwise specified, these libraries can be downloaded the usual manner using the [Arduino Library Manager](https://docs.arduino.cc/software/ide-v1/tutorials/installing-libraries). Each library will be introduced with the hardware it's related to.  
Otherwise all the custom code is in the main Arduino sketch file. The only additional files referenced are some .h files used to store the string and array constansts used to define notes and song tones.

<br>

# **The Main Display (LCD2004 + I2C Backpack)**  
LCD character displays are readily available in 2 or 4 rows, of 16 or 20 characters. A 4 row x 20 character LCD dispaly provides just enough space for a general menu interface that can be used to update settings, start a race, and review best lap result times.  
> ***LCD Part Numbers:** these types of character LCDs usually follow a Part Number pattern of 'LCDccrr', where rr = number of rows, and cc = the number of characters wide it is. (ie. LCD2004 = 4rows of 20ch).*

These displays can be controlled directly using 13 Arduino pins. However because this uses so many pins, it is common to add a small 'backpack' board that will allow us to control these via I2C instead. This reduces the number of signal pins from 13 to just 2. This addition is so common that most LCDs of this type, sold for use with Arduino, have an I2C backpack included.  
> *Though a deeper understanding isn't necessary to use I2C in this project, one may find it helpful for troubleshooting, or if modifying the projcect hardware or software. These references can provide more details regarding I2C, and using the built-in Arduino 'Wire' library.*
> - [I2C Basics](https://rheingoldheavy.com/i2c-basics/) 
> - [The Arduino Wire Library](https://rheingoldheavy.com/arduino-wire-library/)

[![I2C PinsImage](Images/LCD2004%20I2C%20backpack.png)](https://www.mantech.co.za/datasheets/products/LCD2004-i2c.pdf)

## **LCD Libraries and Initialization in Code:**  
The libraries used to communicate with, and update, the LCD provides a pretty straighforward set of methods (i.e. API) that can be called to write strings, numbers, or turn on and off an input cursor on the display.  
  - [Wire](https://www.arduino.cc/en/Reference/Wire) - Built-in Arduino library used to setup and control I2C communication.
  - [hd44780](https://www.arduino.cc/reference/en/libraries/hd44780/) - Of the many available, we have chosen `hd44780` as our LCD display driver and API.  
    - `hd44780_I2Cexp.h` - Because we are using an LCD with an I2C backpack we need to also include the *hd44780_I2Cexp.h* io class which is installed with the *hd44780* library.

Declaration and Setup of LCD dispaly in `SlotCarRaceController.ino`
```cpp
// The 'Wire' library is for I2C, and is included in the Arduino installation.
// Specific implemntation is determined by the board selected in Arduino IDE.
#include <Wire.h>
// LCD driver libraries
#include <hd44780.h>						// main hd44780 header
#include <hd44780ioClass/hd44780_I2Cexp.h>	// i/o class for i2c backpack

//***** Variables for LCD 4x20 Display **********
// This display communicates using I2C via the SCL and SDA pins,
// which are dedicated by the hardware and cannot be changed by software.
// For the Arduino Nano, pin A4 is used for SDA, pin A5 is used for SCL.
// Declare 'lcd' object representing display:
// Use class 'hd44780_I2Cexp' for LCD using i2c i/o expander backpack (PCF8574 or MCP23008)
hd44780_I2Cexp lcd;
// Constants to set display size
const byte LCD_COLS = 20;
const byte LCD_ROWS = 4;

void setup(){
  --- other code ---
  // --- SETUP LCD DIPSLAY -----------------------------
  // Initialize LCD with begin() which will return zero on success.
  // Non-zero failure status codes are defined in <hd44780.h>
  int status = lcd.begin(LCD_COLS, LCD_ROWS);
  // If display initialization fails, trigger onboard error LED if exists.
  if(status) hd44780::fatalError(status);
  // Make sure display has no residual data and starts in a blank state
  lcd.clear();
  --- other code ---
}
--- remaining program ---
```

<br>

# **Racer Lap Timers (8-digit, 7-seg LED Bar)**
Each racer we will have a dedicated lap counter and timer display. We will need 3 digits for the lap count and 4 significant digits to display for the current lap time. This will allow up to 3 decimal places up to 10sec, 2 decimal places up to 1min, and 1 decimal place up to 10min.  
We can use a single 8 digit LED bar to accomodate this need, for each racer.  
Because the primary purpose of this display is to show numbers, a 7-segment LED is a perfect choice. As with the LCD, we could drive them directly from the Arduino, but the number of required pins is even worse. Each 7-segment digit has 1 LED for each segment and an 8th for a decimal LED, each requiring its own pin. This means a single 8 digit bar of 7-segment LEDs, for a single racer, requires 64 pins.  
[![Arduino 7 Segment LED](Images/7-segment-led-display.png)](https://www.electroschematics.com/arduino-segment-display-counter/)

## **The MAX7219 Serial LED Driver:**
### **Use Serial LED Driver to Minimize Pin Count**  
Luckily, our pin problem can be overcome by using a chip like the [MAX7219](https://www.14core.com/wp-content/uploads/2016/03/MAX7219-MAX7221.pdf), which can drive up to 64 LEDs while requiring only 3 signal pins from the Arduino. As such, it's common to find pre-assembled LED bars having 4, or 8, 7-segment digits, with an integrated MAX7219 such as shown here. Each racer will r  
![8-digit 7-Seg LED](Images/MAX7219,%208-digit%207-seg%20LED%20Bar.png)  
### **Chain The Lap Timer Displays**  
Another feature of the MAX7219, that makes these LED bars a good choice for this application, is the ability to cascade (i.e. daisy chain) a number of them together. By taking advantage of the MAX7219's no-op register we can update any digit of any of the racer's LED bars using the same 3 signal pins from the Arduino. The LED driver library will handle the implementation details regarding this, so it's not really necessary to understand more than we can connect them together and address any given digit individually.  
![MAX7219 LED Cascade](Images/LED_MAX7219_Cascade.png)

### **Noise Sensitivity**  
The MAX7219 can sensitive to noise on its power input. If the power lines are clean there will likely not be an issue, however, the MAXIM documentation on using the [MAX7219](https://www.14core.com/wp-content/uploads/2016/03/MAX7219-MAX7221.pdf), strongly recommends using a [bypass filter](https://www.electronicdesign.com/power-management/power-supply/article/21808839/3-ways-to-reduce-powersupply-noise), consisting of a 10&mu;F (polarized, electrolytic) and 100nF (i.e. 0.1&mu;F, #104) capacitors across the input voltage into the MAX7219 and ground.
 
|Bypass Diagram| Capacitor Diagram Symbol Review|
|:---:|:---:|
|![Bypass Caps](Images/BypassCaps.png) | [<img src=Images/CapacitorSymbols.png />](https://www.ifixit.com/Wiki/Troubleshhoting_logic_board_components) 

<br>

## LED LIibraries and Initialization in Code:  
- [LedControl](https://www.arduino.cc/reference/en/libraries/ledcontrol/) - library supports MAX7219 & MAX7221 LED displayseed for the LED bars.

Declaration and Setup of LED dispalys in `SlotCarRaceController.ino`
```cpp
// library for 7-seg LED Bars
#include <LedControl.h>

// ***** 7-Seg 8-digit LED Bars *****
const byte PIN_TO_LED_DIN = 2;
const byte PIN_TO_LED_CS = 3;
const byte PIN_TO_LED_CLK = 4;
// # of attached max7219 controlled LED bars
const byte LED_BAR_COUNT = 2;
// # of digits on each LED bar
const byte LED_DIGITS = 8;
// LedControl parameters (DataIn, CLK, CS/LOAD, Number of Max chips (ie 8-digit bars))
LedControl lc = LedControl(PIN_TO_LED_DIN, PIN_TO_LED_CLK, PIN_TO_LED_CS, LED_BAR_COUNT);


```

<br>

# **Playing Audio**  
## **Arduino [tone()](https://www.arduino.cc/reference/en/language/functions/advanced-io/tone/)**
Playing simple beeps and boops on the Arduino can be done with a single call to the built in Arduino `tone()` function. Here we use `tone()` in a wrapper function, `Beep()` that we can call when we want to play a feedback sound, such as when a keypad button is pressed.

```cpp
// A3 is a built in Arduino pin identifier
const byte buzzPin1 = A3;

void Beep() {
  // tone(pin with buzzer, freq in Hz, duration in ms)
  tone(buzzPin1, 4000, 200);
}
```

> *Notes regarding playing sounds using Arduino `tone()`.*
> - *The requested `tone()` plays in parallel once it is called, therefore it does NOT block the code loop while playing out the duration of a note.*
> - *`tone()` uses the same timer as pins 3 and 11. Therefore, one cannot `analogWrite()` or PWM on those pins while `tone()` is playing.*
> - *It is not possible to play the `tone()` function on two pins at the same time. Any in process tones must be stopped before starting a tone on a different pin.*  
> - *The minimum tone that can be generated is `31Hz`. A lower value can be submitted without error, but it won't play lower than `31Hz`.*
> - *The maximum frequency for UNO and most common boards is `65535Hz`.*  
> - *The audible range for most people is `20Hz-20kHz`.*

## **Songs & Melodies**
To play a melody, we need to play a series of tones corresponding to the appropriate musical notes. Presented here are two common methods of coding and playing non-blocking audio melodies on the Arduino.

## **Method 1: `Notes[]` & `Lengths[]` Arrays ( `pitches.h` )**  
This is probably the most commonly used approach, and is the most versatile way to use `tone()` to play a melody. In this approach we will represent the musical notes that make up a melody, using two arrays, one array to hold the note frequencies, `Notes[]`, and one to hold the note lengths, `Lengths[]`, which will be used to determine each note's tone duration.

> *Though most of the necessary concepts will be reviewed herein, some existing understanding of basic music structure and notation will be extremely helpful in grasping how playing melodies works.*  
> *Here are some resources to review or learn about musical notation and structure:*  
> - *[Musical Note Names: Organizing the Notes](https://www.allaboutmusictheory.com/piano-keyboard/music-note-names/)* - understanding 'C4', 'C5', etc.
> - *[Sheet Music Notation: The Complete Beginner’s Guide](https://yourcreativeaura.com/sheet-music-notation/) - good review on sheet notation from the ground up.*
> - *[How to Read Music Notes (Quick-learn cheat sheets)](https://cookband.files.wordpress.com/2012/02/how-to-read-music-notes-qlcss-pp1-9-dunn.pdf) - pdf cheat sheet of music notation.*
> - *[Open Music Theory](http://openmusictheory.com/) - an interactive, online, college level music theory text.*
> 
> *These are good articles for grasping key signatures, which is kind of a tricky topic.*
> - *[Key Signature and Music Staff](https://www.aboutmusictheory.com/key-signature.html)*
> - *[A Complete Guide to Music Key Signatures](https://www.merriammusic.com/school-of-music/piano-lessons/music-key-signatures/)*

### **Determining The Frequency Array ( `Notes[]` )**
To understand the relationship between our code and real music, we'll start by considering the keys of a piano. Each key plays a different note which is quantified as a particular frequency of sound waves. In this diagram we find the notes corresponding to each key, their frequencies in Hz, their octave number, and staff notation for the center notes.  

[![Note Freq](Images/NoteFrequencies.png)](https://education.lenardaudio.com/en/03_db.html)

With this information we can construct a [pitches.h](pitches.h) file that defines a list of notes and their corresponding frequencies in Hz. 
A portion of `pitches.h` is shown here, defining C in octave 4 (aka middle C) = 262Hz, C4# = 277 Hz, D4 = 294Hz, and D4# = 311Hz:

```cpp
#define NOTE_C4 262
#define NOTE_CS4 277
#define NOTE_D4 294
#define NOTE_DS4 311
```

> ***The #define directive:** - In the `pitches.h` files, we are using the [`#define` preprocessor directive](https://www.ibm.com/docs/en/zos/2.3.0?topic=directives-define-directive). This is a macro definition (e.g. `#define NOTE_C4 262`) that contains an identifier (e.g. `NOTE_C4`), and a replacement token-string, (e.g. `262`).  
> Just before the code is actually compiled, a preprocessor will replace all instances, in code, of the identifier, with the replacement token-string. In the case of `pitches.h` it will replace a given note id with the integer frequency in Hz. This is to be distinguished from using a constant variable.*

Using the notes defined in `pitches.h`, we can now build an array of the notes that make up a melody. For example, we can take the basic C-Major Scale:

 C • D • E • F • G • A • B:

![C Major Scale](Images/C_Major_Scale.png)

and record it in a `Notes[]` array, as such:
> Storing in `PROGMEM` is optional
```cpp
const int cMajorScaleNotes[] PROGMEM = {
  NOTE_C4, NOTE_D4, NOTE_E4, NOTE_F4, NOTE_G4, NOTE_A4, NOTE_B4
};
```
### **Determining The Lengths Array ( `Lengths[]` )**
Each note in the `Notes[]` arrary has a note length that must be accounted for. We can store this note length in a second array, `Lengths[]`, where `Note[i]` = a note's freq and `Lengths[i]` = the corresponding length. 

The length of a note, or rest, in music is measured in number of beats and recorded on sheet music as follows:

![note lengths](Images/notes%20and%20rests.jpg)

Ultimately we will need a millisecond integer value, to input as the duration of a note, to play a `tone()`. However, it is more musically natural, and more versatile to capture the intended note length in the `Lengths[]` array. This allows the same song data to be played at different tempos, using the same code.

Most often, we find note length in a `Lengths[]` array using the following notation:

- `1` = whole note
- `2` = half note
- `4` = quarter note
- `8` = eight note
- `16` = 1/16th note
- etc.

All of the notes in our C-Major Scale are quarter notes, so using the note length notation, we can finish our C-major Scale arrays as follows:

```cpp
const int cMajorScaleNotes[] PROGMEM = {
  NOTE_C4, NOTE_D4, NOTE_E4, NOTE_F4, NOTE_G4, NOTE_A4, NOTE_B4
};
const int cMajorScaleLengths[] PROGMEM = {
  4, 4, 4, 4, 4, 4, 4
};
```
In order to convert our note lengths into millisecond durations, we need to establish a **tempo**.

How long a beat lasts in real time is established by the tempo of the melody in **beats per minute (bpm)**. The tempo on a sheet of music is sometimes declared by assigning a bpm to a note. Often this is the quarter note since a quarter note is equal to 1 beat, but it doesn't have to be.  

This indicates the tempo is 70bpm:

![bmp quarternote](Images/bpmq.png)  
Usually however, instead of a numerical bpm, an Italian term desribing the tempo is used which can be interpreted into bpm and duration as such: ([Music Note Length Calculator](https://rechneronline.de/musik/note-length.php))
| Tempo       | Speed                     |bpm         | ms/beat      |
| ----------- | --------------------------|----------- | :-----------:|
| Larghissimo | very, very, slow          |20 or lower | \> 3000      |
| Grave       | slow and solemn           |20 to 40    | 3000 - 1500  |
| Lento       | slowly                    |40 to 45    | 1500 - 1333  |
| Largo       | broadly                   |40 to 60    | 1333 - 1000  |
| Larghetto   | rather broadly            |60 to 66    | 1000 - 909   |
| Adagio      | slow and stately          |66 to 76    | 909 - 789    |
| Andante     | at a walking pace         |76 to 108   | 789 - 556    |
| Moderato    | moderately                |108 to 120  | 556 - 500    |
| Allegro     | fast, quickly, and bright |120 to 168  | 500 - 357    |
| Vivace      | lively and fast           |138 to 168  | 435 - 357    |
| Presto      | extremely fast            |168 to 200  | 357 - 300    |
| Prestissimo | even faster than Presto   |200 and up  | < 300        |

To account for tempo we could use the same tempo for everything and hard code it into the `Play()` function, but it's easy enough to be flexible and let each song have its own tempo.

Finally, because in C++ it can be challenging to know how many elements are in an array if using pointers and passing them into functions it's worth turning it into a constant right away and use a `count` variable. This will be used by the play function to determine when the melody is over.

The result is, 2 arrays, and 2 integer constants that will fully define each melody.

```cpp
const int cMajorScaleNotes[] PROGMEM = {
  NOTE_C4, NOTE_D4, NOTE_E4, NOTE_F4, NOTE_G4, NOTE_A4, NOTE_B4
};
const int cMajorScaleLengths[] PROGMEM = {
  4, 4, 4, 4, 4, 4, 4
};
// tempo in beats per minute
const int cMajorScaleTempo = 60;
// getting note count for easy reference later
const int cMajorScaleCount = sizeof(cScaleNotes)/sizeof(int); 
```

## **Playing the Melody Arrays**
Now that we have a melody transcribed into an array of frequencies and durations, in order to play it we need to cycle through the arrays playing each note in time. Because we want to be able to do other things while the song is playing we will need to track passing time so we know when to play the next note.

Because we have many songs to play we'll create a set of global reference variables that we can use to point to different song data variables. We use pointers to the data arrays instead of a temporary copy, to save memory, and because we have songs of different sizes. Managing dynamic array sizing is an uncessary, memory fragmenting, challenge.

```cpp
// Globals for holding the current reference song data
int *playingNotes;
int *playingLengths;
int songLength;
// Default tempo to 0 which means, use note length value as raw milliseconds
byte activeTempoBPM = 0;
// flag to indicate to the main program loop whether a melody is in process
// so it should execute the 'PlayNote()' function with the current melody parameters.
bool melodyPlaying = false;
// Holds the time of last tone played so timing of next note in melody can be determined
unsigned long lastNoteMillis = 0;
int melodyIndex = 0;
// time in ms between beginning of last note and when next note should be played.
int noteDelay = 0;

int PlayNote(int *songNotes, int *songLengths, int curNoteIdx, byte tempoBPM = 0){
  int noteDuration;
  // If tempo is zero then use length directly as ms duration
  if(tempoBPM == 0){
    noteDuration = pgm_read_word(&songLengths[curNoteIdx]);
  } else {
    // Otherwise calculate duration from bpm:
    // (60,000ms/min)/Xbpm * 4beats/note * 1/notelength
    noteDuration = (60000 / tempoBPM) * 4 * ( 1.0 / pgm_read_word(&songLengths[curNoteIdx]) );
  }
  // Mark time this note began
  lastNoteMillis = millis();
  tone(buzzPin1, pgm_read_word(&songNotes[curNoteIdx]), noteDuration);
  melodyIndex++;

  if(curNoteIdx == songLength - 1){
    // When done turn play flag off and reset the play tracking variables.
    melodyPlaying = false;
    noTone(buzzPin1);
    melodyIndex = 0;
    noteDelay = 0;
    songLength = 0;
  }
  // Digital notes have no transition break so can sound unatural
  // Making the time between notes slightly longer than the note will
  // create a small break to make the melody sound more natural.
  // the note's duration + 10% seems to work well:
  return noteDuration * 1.1;
}

void loop() {
  if(melodyPlaying){
    if(millis() - lastNoteMillis >= noteDelay){
      // songs like Knight rider duration is in ms are 'raw = true'
      // songs like Take On Me use inverse notes 4 = 1/4note, 8=1/8th, etc.
      // these are considered 'raw = false'. 
      noteDelay = PlayNote(playingNotes, playingLengths, melodyIndex, false);
    }
  } else {
    noTone(buzzPin1);
  }
  --- other code ---
  // To play a song we set the flag to true and re-assign song pointer to desired tune.
  melodyPlaying = true;
  playingNotes = takeOnMeNotes;
  playingLengths = takeOnMeLengths;
  songLength = takeOnMeCount;

  --- other code ---
}
```

> ## **Transcribing *'Take On Me'* Into Playable Arrays**
> To illustrate the process, we will transcribe the intro to Take On Me by Aha! Here are the first 12 meausres (bar 1 repeated twice, and 2nd bar) of the sheet music.

> ![Take On Me Sheet](Images/TakeOnMeIntro_SheetMusic.png)

> Looking at the beginning of the staff



> Transcribed 'Take On Me' melody snippet from `melodies_prog.h`
```cpp
const int takeOnMeNotes[] PROGMEM = {
  NOTE_FS5, NOTE_FS5, NOTE_D5, NOTE_B4, 0, NOTE_B4, 0, NOTE_E5,
  0, NOTE_E5, 0, NOTE_E5, NOTE_GS5, NOTE_GS5, NOTE_A5, NOTE_B5, 
  NOTE_A5, NOTE_A5, NOTE_A5, NOTE_E5, 0, NOTE_D5, 0, NOTE_FS5, 
  0, NOTE_FS5, 0, NOTE_FS5, NOTE_E5, NOTE_E5, NOTE_FS5, NOTE_E5,

  NOTE_FS5, NOTE_FS5, NOTE_D5, NOTE_B4, 0, NOTE_B4, 0, NOTE_E5,
  0, NOTE_E5, 0, NOTE_E5, NOTE_GS5, NOTE_GS5, NOTE_A5, NOTE_B5, 
  NOTE_A5, NOTE_A5, NOTE_A5, NOTE_E5, 0, NOTE_D5, 0, NOTE_FS5, 
  0, NOTE_FS5, 0, NOTE_FS5, NOTE_E5, NOTE_E5, NOTE_FS5, NOTE_E5,

  NOTE_FS5, NOTE_FS5, NOTE_D5, NOTE_B4, 0, NOTE_B4, 0, NOTE_E5,
  0, NOTE_E5, 0, NOTE_E5, NOTE_GS5, NOTE_GS5, NOTE_A5, NOTE_B5, 
  NOTE_A5, NOTE_A5, NOTE_A5, NOTE_E5, 0, NOTE_D5, 0, NOTE_FS5, 
  0, NOTE_FS5, 0, NOTE_FS5, 0
};
// The note duration, 8 = 8th note, 4 = quarter note, etc.
const int takeOnMeLengths[] PROGMEM = {
  8, 8, 8, 8, 8, 8, 8, 8,
  8, 8, 8, 8, 8, 8, 8, 8,
  8, 8, 8, 8, 8, 8, 8, 8,
  8, 8, 8, 8, 8, 8, 8, 8,

  8, 8, 8, 8, 8, 8, 8, 8,
  8, 8, 8, 8, 8, 8, 8, 8,
  8, 8, 8, 8, 8, 8, 8, 8,
  8, 8, 8, 8, 8, 8, 8, 8,

  8, 8, 8, 8, 8, 8, 8, 8,
  8, 8, 8, 8, 8, 8, 8, 8,
  8, 8, 8, 8, 8, 8, 8, 8,
  8, 8, 8, 8, 2
};
const int takeOnMeCount = sizeof(takeOnMeNotes)/sizeof(int);
```
<br>

---  

## Method 2: Ringtone RTTTL Format  
At one time when ringtones were

## Libraries and Aduio Data Files  
TODO
```cpp
// Enable if using RTTL type song/melody data for playing sounds
//-------------------
// library for playing RTTTL song types
#include <PlayRtttl.h>
// file of RTTTL song definition strings.
// Because these strings are stored in PROGMEM we must also include 'avr/pgmspace.h' to access them.
#include "RTTTL_songs.h"
//-------------------

// Enable if using Note-Lengths Array method of playing Arduino sounds
//-------------------
// // Defines the note constants that make up a melodie's Notes[] array.
// #include <pitches.h>
// // File of songs/melodies defined using Note & Lengths array.
// // Because these arrays are stored in PROGMEM we must also include 'avr/pgmspace.h' to access them.
// #include "melodies_prog.h"
//-------------------

// Library to support storing/accessing constant variables in PROGMEM
#include <avr/pgmspace.h>
```

