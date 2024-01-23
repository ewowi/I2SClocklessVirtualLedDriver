[![build status](https://github.com/hpwit/I2SClocklessVirtualLedDriver/actions/workflows/testcode.yml/badge.svg)](https://github.com/hpwit/I2SClocklessVirtualLedDriver/actions/workflows/testcode.yml) &nbsp;&nbsp;[![Badge Version](https://img.shields.io/github/v/release/hpwit/I2SClocklessVirtualLedDriver?label=latest%20release)](https://github.com/hpwit/I2SClocklessVirtualLedDriver/releases/latest) &nbsp;[![Badge Version](https://img.shields.io/badge/release_note-blue)](Releasenote.md)

# I2SClocklessVirtualLedDriver for esp32
## Introduction
Hello led afficionados !! Here is the new version of the Virtual pins library. In reality this version of the library had been sitting more or less finalized on my laptop for a while. I need to take the time and energy to write the correct examples and of course update the documentation. 
I have been writing led drivers for the past couple of years now while I was building [my 123x48 panel](https://hackaday.io/project/158268-5904-leds-panel). 
It inspired me to create the I2S driver implemented in FastLED and then the Virtual pins library.
I am also planning to merge all my different led libraries (4 of them in total)


### What kind of led do you want to drive
This library is a new take on driving ws2812 leds with I2S on an esp32. It allows to drive up to 120 strips !!! leds in parallel of  
* RGB:
    * WS2812,
    * WS2813,
    * WS2815 
* RGBW 
    * SK6812. 

If you are using RGB led type then this library is fully compatible with FastLED library objects

### Why have I rewritten the library ?
I have rewritten the library out of the FastLED framework to allow easier testing but also create a pixel pusher independant of the library you want to use. But the main reason is the way I wanted to drive the leds. I haev tried to put more functionalities in the driver than the usual 'leds pusher'. Indeed this driver integrates:
* led mapping
* color palette
* ease the use of the second core
* framebuffer
* non blocking capabilities
* 'live led' calculation
* scrolling, rotating, scaling
* duplication
* emulate 'line interrupts' (retro programers will understand)
* Options to avoid artifacts if you're have interrupt intensive code


I am trying to be kinda lenghtly on this readme. I hope to explain the why of some functions and for the user to use the one most suitable for its use case.
Some of the 'maybe most' of the functionailities will never been used be it was fun and intersting to implement.

## Let's start
### How can 120 strips can de driven in parallel with an esp32 !!!
The esp32 has 24 pins that can act as outputs pins. This is far away from 120 pins ...
The main idea is to drive 8 strips out of one single pin of the ESP32. and you can do this for 15 esp32 pins which gives you 8x15=120 strips !!!

To do this feat you will need to get some ICs
* 74HC245 : this is a bus used as a level shifter (you will need only one of them for LATCH and CLOCK)
* 74HC595 : this is an 8 bit shift register (you will need one 74HC595 for each Virtual pin)

You'll find the schematics within the 'extra' folder of this library.

I personally use it to drive my panels 5904=123x48 (16 strips of 369 leds) and a 12288=128x96 (48 16x16 panels)  but still have space for an I2C Nintenda controller and SD card reader.


### Pixel object
In FastLED ``CRGB`` and ``CHSV`` objects exist, they represent data in RGB or CHSV color space. These are made of 3 bytes (24 bits)

In this library there is a similar object ``Pixel`` which is equivalent to ``CRGB``. But it can be changed at compile time.
```C
#define COLOR_RGBW

#include "I2SClocklessVirtualLedDriver.h"
#define NUM_LEDS 200

Pixel leds[NUM_LEDS];
//here the Pixel is not RGB but RGBW hence this time it will be a 36 bits
//it's still ocmpatible with the CRGB object 
leds[0]=CRGB(233,23,44); 
```
in that case the following mapping will be done 
W = min(R, G, B);
R = R - W; 
G = G - W;
B = B - W;

In the examples you will find some that integrate the FastLED objects.

### MACROS that have nothing to do with led driving

* ``HOW_LONG(name, func)`` : This macro will the time taken by a part of the code
ex:
```C
HOW_LONG("Time taken", {
//insert the code i.e
driver.showPixels();

});
```

the result  in the serial output:
``The function *** Time taken *** took 100 ms or 10 fps``

*  ``RUN_SKETCH_FOR(name, duration, func)``:  Execute part of the code for the duration in millisecondes
ex:
```C
//it will run for 1000ms or 1s
int offset=0;
RUN_SKETCH_FOR("scroll", 1000 {
    leds[offset]=CRGB::Red;
    driver.showPixels();
    offset++;
});

```

*  ``RUN_SKETCH_N_TIMES(name, ntimes, func)``:  Execute part of the code N times
ex:
```C
//it will run 100 times
int offset=0;
RUN_SKETCH_FOR("scroll", 100 {
    leds[offset]=CRGB::Red;
    driver.showPixels();
    offset++;
});

```

### Array of strips
In most leds driver librairies you declare each strip attached to one pin, one line at a time.

example 1: For 4 strips in FastLED
```C
CRGB leds1[number_of_leds1];
CRGB leds2[number_of_leds2];
CRGB leds3[number_of_leds3];
CRGB leds4[number_of_leds4];

FasLED.addLeds<PIN1,ORDER>(leds1,number_of_leds1);
FasLED.addLeds<PIN2,ORDER>(leds2,number_of_leds2);
FasLED.addLeds<PIN3,ORDER>(leds3,number_of_leds3);
FasLED.addLeds<PIN4,ORDER>(leds4,number_of_leds4);
```

example 2: If you are using a large array of same length strips, you would do this:
```C
CRGB leds[4*NUM_LED_PER_STRIPS];
FasLED.addLeds<PIN1,ORDER>(leds,0,NUM_LED_PER_STRIPS);
FasLED.addLeds<PIN2,ORDER>(leds,NUM_LED_PER_STRIPS,NUM_LED_PER_STRIPS);
FasLED.addLeds<PIN3,ORDER>(leds,2*NUM_LED_PER_STRIPS,NUM_LED_PER_STRIPS);
FasLED.addLeds<PIN4,ORDER>(leds,3*NUM_LED_PER_STRIPS,NUM_LED_PER_STRIPS);
```

example 3: For information if you want to get (for development purpose ease) the leds1,leds2,...
```C
CRGB *leds1=leds;
CRGB *leds2=leds+NUM_LED_PER_STRIPS;
CRGB *leds3=leds+2*NUM_LED_PER_STRIPS;
CRGB *leds4=leds+3*NUM_LED_PER_STRIPS;
```

If all the strips of the first example are of the same size, then the 2 examples are the doing exactly the same. Hence when using strips of different lengths we cannot put them in a  big array ? **FALSE**. You cant create a large array when using `NUM_LED_PER_STRIP` being the largest of `number_of_leds`. Of course you array woul be larger than you actual numbre of leds but we can do with the lost of space.

### OK, but what is the link between an array of strips and this driver ?
Here is how we would declare the 4 strips in of our example:
```C
CRGB leds[4*NUM_LED_PER_STRIPS];
int pins[4]={PIN1,PIN2,PIN3,PIN4};
driver.initled((uint8_t*)leds,pins,4,NUM_LED_PER_STRIPS,ORDER_GRB);
```
We are declaring that my `leds` array represent 4 strips of `NUM_LED_PER_STRIPS` leds ,each strip being linked to the pins defined in the pins array `pins`. This is way easier to declare a lot of strips. As discussed before if your strips are not of the same lentgh just define `NUM_LED_PER_STRIPS` with the largest `number_of_leds`.


### First let's declare a new driver

```C

#define NBIS2SERIALPINS 2  //the number of virtual pins

//set the strip color order
//by default RGB
/* the other values
#define COLOR_RGB
#define COLOR_RBG
#define COLOR_GBR
#define COLOR_BGR
#define COLOR_BRG
#define COLOR_RGBW
*/

#define NUM_LEDS_PER_STRIP 256

#define NUM_LEDS (NUM_LEDS_PER_STRIP*NBIS2SERIALPINS*8)

#define CLOCK_PIN 16

#define LATCH_PIN 4

#include "I2SClocklessVirtualLedDriver.h"

I2SClocklessVirtualLedDriver driver;
```

### How to define my leds array ?
#### You are using RGB type leds
RGB type leds store the information of the led over 3 bytes. `Red,Green,Blue`.  Hence the size in bytes of a led array of `NUM_LEDS` is `3xNUM_LEDS`
```C
Pixels leds[NUM_LEDS];

//if you are using FastLED library this definition will be equivalent to the previous as the CRGB object is 3 bytes
CRGB leds[NUM_LEDS];

//you can use either of those
```
#### You are using RGBW type leds
This time to store the information of the led you will need 4 bytes `Red,Green,Blue,White` Hence the size in bytes of a led array of `NUM_LEDS` is `4xNUM_LEDS`
```C
Pixels leds[UM_LEDS];
```

### Driver functions
#### `void initled(uint8_t *leds, int *Pinsq, int CLOCK_PIN, int LATCH_PIN)`:

 This function initialize the strips.
* `*leds`: a pointer to the leds array
* `*Pins`: a pointer to the pins array
* `CLOCK_PIN`: pin for the clock output needs to be >=16
* `LATCH_PIN`: pin for the latch ouput

 
 example 4: declaring 16 strips of 256 leds in GRB 
 ```C
 
#define NBIS2SERIALPINS 2  //the number of virtual pins
#define COLOR_GRB 1 //set the strip color
#define NUM_LEDS_PER_STRIP 256
#define NUM_LEDS (NUM_LEDS_PER_STRIP*NBIS2SERIALPINS*8)
#define CLOCK_PIN 16
#define LATCH_PIN 4
#define NUM_STRIPS (NBIS2SERIALPINS*8)
 
 #include "I2SClocklessVirtualLedDriver.h"

 I2SClocklessVirtualLedDriver driver;
 
 Pixels leds[NUM_STRIPS*NUM_LEDS_PER_STRIP]; //equivalent of CRGB leds[NUM_LEDS_PER_STRIPS*NUM_STRIPS]
 int pins[NBIS2SERIALPINS] ={0,2};
 driver.initled((uint8_t*)leds,pins,CLOCK_PIN,LATCH_PIN);
 ```
 
 example 5: declaring 30 strips of 256 leds in RGBW
 ```C
 #define NBIS2SERIALPINS 4  //the number of virtual pins
#define COLOR_GRBW 1 //set the strip color to GRBW
#define NUM_LEDS_PER_STRIP 256
#define NUM_LEDS (NUM_LEDS_PER_STRIP*NBIS2SERIALPINS*8)
#define CLOCK_PIN 16
#define LATCH_PIN 4
#define NUM_STRIPS 30
 
 #include "I2SClocklessVirtualLedDriver.h"

 I2SClocklessVirtualLedDriver driver;
 
Pixel leds[NUM_STRIPS*NUM_LEDS_PER_STRIP]; 
 int pins[NBIS2SERIALPINS] ={0,2,12,13};
 driver.initled((uint8_t*)leds,pins,CLOCK_PIN,LATCH_PIN);
 ```

#### Other initialization functions:
* `void initled(Pixel leds, int *Pinsq, int CLOCK_PIN, int LATCH_PIN)`
*  `void initled(CRGB leds, int *Pinsq, int CLOCK_PIN, int LATCH_PIN)`
*  `void initled(frameBuffer *leds, int *Pinsq, int CLOCK_PIN, int LATCH_PIN)` this will be explained later on how to use a frameBuffer
*  `void initled(int *Pinsq, int CLOCK_PIN, int LATCH_PIN)`: you can initialized the driver without a led buffer.
this is also used in the case of direct pixel calculation
 ```C
 Pixel led1[NUM_LEDS];
 Pixel leds2[NUM_LEDS];
driver.initled(pins,CLOCK_PIN,LATCH_PIN);

...
driver.showPixels(led1);
...
driver.showPixels(led2);
 ```


#### `setBrightness(int brightness)`:
 
 This function sets the default brightness for 0->255



 
#### `setPixel(uint32_t pos, uint8_t red, uint8_t green, uint8_t blue)`:
 Set the color of a pixel 
 NB1: if you are using a RGBW led, this function will do and RGB->RGBW transformation with the following algotithm thanks to  @Jonathanese 
```C
W = min(R, G, B);
R = R - W; 
G = G - W;
B = B - W;
```
 NB: if you are familiar with FastLED it would be `leds[pos]=CRGB(red,green,blue)` as you will see in the examples
 
 
 
 #### `setPixel(uint32_t pos, uint8_t red, uint8_t green, uint8_t blue,uint8_t white)`:
 Set the color of a pixel for RGBW strips
 
 #### `showPixels()`:
 
 This function displays the pixels.

 #### `showPixels(uint8_t *leds)`:
This function allow set 'on the go' the pointer to the leds. This will help if you are using two buffers for your animation. 
It can also be used to ease dithering see example `Dithering` (I need to work on a hardware implementation btw)
```C
#define NBIS2SERIALPINS 4  //the number of virtual pins
#define COLOR_GRB 1 //set the strip color
#define NUM_LEDS_PER_STRIP 256
#define NUM_LEDS (NUM_LEDS_PER_STRIP*NBIS2SERIALPINS*8)
#define CLOCK_PIN 16
#define LATCH_PIN 4
#define NUM_STRIPS 30
#include "I2SClocklessVirtualLedDriver.h"

 I2SClocklessVirtualLedDriver driver;
 
 Pixel leds[NUM_STRIPS*NUM_LEDS_PER_STRIP];  
 Pixel leds2[NUM_STRIPS*NUM_LEDS_PER_STRIP]; 
 int pins[NBIS2SERIALPINS] ={0,2,12,13};
 driver.initled((uint8_t*)leds,pins,CLOCK_PIN,LATCH_PIN);

//displyaing the leds in leds1
driver.showPixels();

//displaying the leds in leds2
driver.showPixels(leds2);
```

## Let's talk about framerate
It's kind of a goal to display as fast as it's possible to have a good framerate. The  calculation of the framerate is not only base on the time to display the leds but also the time 'create' the led array.
i.e
```C
//display total  tiome/fps fpstotal
HOW_LONG("Total framerate", {
    //display time/fps to calculate the array fpscalc
    HOW_LONG("calcualtion led", {
    for(int i=0;i<NUM_LEDS;i++)
    {
        //do something
    }
    });
    //will display time/fps display the array fpsdisp
    HOW_LONG("display", {
        driver.showPixels();
    });
});

// here is the formula to calculte the total framerate 1/fpstotal=1/fpscalc+1/fpsdisp
```

## How do we improve the framerate

### On the same core 
Even if the showPixels() takes time, actually the CPU is not busy during the time. Indeed even if to send a pixel (RGB) it takes 30us (microseconds) but the calculation of the buffer takes for 5us to 23us depending on the number of virtual pins to calculate.
As a consequence we are loosing CPU time. Hopefully the esp32 rune several tasks at the sametime thanks to RTOS.

the driver allows to take advantage of this quite easily. `void showPixels(displayMode dispmode)` 

```C

......
 driver.showPixels(NO_WAIT);
....

```

In the example `waitnowaitmode.ino` we display 256 leds (max framerate 130fps). If you run the code (even without any leds attached) it will display `75fps` for the showPixels() and `112fps` for the `showPixels(NO_WAIT)`.
Why is it not 130 fps the max ? because the calculation of the leds array is done wiht the same ressource as the one to calculate the leds buffers to be sent. Both 'compiting' for the same CPU ressource. But we get a 50% speed increase.

### Enable the use of the second core
The esp32 has two cores and the driver simplify the usage of the second core.
The main core is core 1 on which runs the application bu default. The second core (core 0) is less used. We can tell the driver to run driver.showPixels() on the second core by `enableShowPixelsOnCore(int core)`

```C

......
//move the display on core 0
 driver.enableShowPixelsOnCore(0);
 //from now on all the showPixels() will be done on core 0
....


//move the display back on core 1
 driver.enableShowPixelsOnCore(1);
```
In the example `secondcore.ino`  without the second core we are still at `75fps` and `129fps` (close to the max) when using the two cores.
In the latest case both the calculation and the display occurs at the sametime.

:arrow_forward: NB: by construct using `driver.enableShowPixelsOnCore(1)` all the following `showPixels()` will be like `showPixels(NO_WAIT)`  

### Impact of these two previous functions
Let's have a look at this example.(here I use `NO_WAIT` but I could have used the second core)

```C
for(int i=0;i<NUM_LEDS;i++)
    {
        leds[i]=CRGB::Red;
    }

driver.showPixels(NO_WAIT); 

for(int i=0;i<NUM_LEDS;i++)
    {
         leds[i]=CRGB::Blue;
    }
```

After the `showPixel(NO_WAIT)` the code continues its execution and the second loop starts. But we have an issue because the program modifies the leds array that is currently being displayed. In that case some blue leds will appear :scream:.
If you're displaying a fast animation this would not be to much of an issue but in any case we need to find a solution if you see any overlap due to this :
* calculate the next frame starting with the pixels that would be displayed last. good luck with your algorithm.
* use two frame buffers

## Framebuffer
Here it is :smiley: As this driver simplify the use of the second core, it simplifies the use of framebuffers.
```C
//use this instead of Pixel leds[NUM_LEDS]
frameBuffer leds = frameBuffer(NUM_LEDS);
...
driver.initled(&leds, Pins, _CLOCK_PIN, _LATCH_PIN);
//NB: it's initled(&leds,...  this time and not initled(led,...
```
The rest of your code will not change at all
```C
    for (int i = 0; i < NBIS2SERIALPINS * 16; i++)
    {
        for (int j = 0; j < 8 * 16; j++)
        {
            //here we write on buffer x
            leds[i * (8 * 16) + (j) % (8 * 16)] = colors[offset % 3];
        }
    }
    //we display buffer x, we change the writing buffer to x+1
    driver.showPixels();
    //now we write in buffer x+1
    leds[54]=CRGB(25,48,79);
    //we display buffer x+1 , we switch back the writing buffer to x
    driver.showPixels();
```
:arrow_forward:if you initialized yuour leds with `framebuffer` Automatically upon calling `showPixels`:
* The call will be switch to `NO_WAIT`
* the driver will change the buffer on which you be writing

hence in the following example at each iteration of the loop we write in another buffer that the one we are currently displaying.
```C
void loop()
{
    for(int i=0;i<NUM_LEDS;i++)
    {
        //do something with the leds
    }
    driver.showPixels();
}
```
In the example `framebuffer.ino` I advice you to change `#define TEST_USE_FRAMEBUFFER 1` to `#define TEST_USE_FRAMEBUFFER 0` to see the effect of the overlap

## Use of color palettes
Even if the RBG leds allows you to display more than 16 millions colors, it can happen that you do not need that many colors. you would refer to a palette then.
for instance if you have less than 256 colors, instead of having `Pixel leds[NUM_LED]` which take `3xNUM_LED` bytes you can define `uint8_t leds[NUM_LED]` whihc will take 3 times less memory.

```C
#define _USE_PALETTE //necessary to tell the driver to use palette
#include "I2SClocklessVirtualLedDriver.h"
...
Pixel palette[256]; //or less if you do not use all 256 colors.
uint8_t leds[NUM_LEDS];

...
void functionToFillThePalette(param ..)
{
    //do so
}

void setup()
{
    driver.initled(leds, Pins, CLOCK_PIN, LATCH_PIN);
    driver.setPalette((uint8_t *)palette);
}
...

driver.showPixels();
```

NB: the mapping between the leds value and the palette color is done at runtime => no need of extra memory. There is no creation od a temporary led array.

### moving the palette instead of the pixels
Like in our old game console, soem animation were made by switching palette. This is waht will happen if you change the palette. like in the example `colorpalete.ino`.
In this case the animation is occuring because of the swift of the palette colors. I need to change 256 color valuies instead of recalculating 12000 leds. In this exmaple I am modifying the palette bu of course you could do this:
```C

Pixel palette1[256]; 
Pixel palette2[256]; 
...


driver.setPalette((uint8_t *)palette1);
driver.showPixels();

driver.setPalette((uint8_t *)palette2);
driver.showPixels();

```

### More than 256 colors palette ?
You can create a 2bytes palette for a max of 65536 colors. in that case your leds array would need to be 2 bytes per leds

```C
#define _USE_PALETTE //necessary to tell the driver to use palette
#define PALETTE_SIZE 2 //for a 2 bytes palette
#include "I2SClocklessVirtualLedDriver.h"

Pixel palette[11123];
uint16_t leds[NUM_LEDS]; //still 1 third less memory than RGB led array
void setup()
{
    driver.initled(leds, Pins, CLOCK_PIN, LATCH_PIN);
    driver.setPalette((uint8_t *)palette);  //still the same call to match the palette
}
```



## Leds mappping
When creating leds structures sometimes it can be hard to match how the leds informaiton is stored and what it physically represent. For instance you have a panel with a snake pattern or a set of 16x16 panels or something else. You can always write a map(x,y) function but the default is that you cannot manipulate the memory as you wish. THe mapping functions do not work for pictures or for artnet you need to reproduce the physical representation of the strip in your artnet software.
The idea of the driver is to provide a way to define the mapping function so that in memory the leds are laid like your expect in (X,Y) cooridinate. the acutal mapping is done using calculation withoout touching the led array. Hence for instance you can define your artnet universes as a simple rectangle and it will display correctly.

### How to define the mapping
for this driver you need to map the led number in the X,Y coordinates to the leds number of the strips.

![mapping](/extra/pictures/mapping.png)
here is the correspondant mapping function.
```C
uint16_t mapfunction(int pos)
{
    int x=pos%8;
    int y=pos/8;
    if(y%2 == 0)
    {
        return y*8+x;
    }
    else
    {
        return (y+1)*8-x-1;
    }
}
```

### You Need to indicate to the driver that you will use mapping
```C
#define I2S_MAPPING_MODE I2S_MAPPING_MODE_OPTION_MAPPING_IN_MEMORY
#include "I2SClocklessVirtualLedDriver.h"
#define LED_HEIGHT 45
#define LED_WIDTH 14
...

uint16_t mapfunction(int pos)
{
    //your maapping function
}

....
driver.initled(leds, Pins, CLOCK_PIN, LATCH_PIN);
driver.setMapLed(&mapfunction);

...
for (int i=0;i<LED_WIDTH;i++)
{
    for(int j=0;j<LED_HEIGHT;j++)
    {
        leds[j*LED_WIDTH+i]=....   //here you're putting your leds in the usual X,Y coordinates
    }
}

driver.showPixels();

```

Because the memory layout is exactly the one of the X,Y coordinate it makes manipulation easier and faster than doing `leds[map(i,j)]=...`

please find a full code example in `mapping.ino`

### About I2S_MAPPING_MODE_OPTION_MAPPING_IN_MEMORY
There are two 'technical' options for the driver to deal with the 'hardware' mapping.

* In memory `#define I2S_MAPPING_MODE I2S_MAPPING_MODE_OPTION_MAPPING_IN_MEMORY`: once the `setMapLed` is executed, an array for the correspondance is created
    * PRO: less CPU intensive
    * CON: you need more memory
* In software `#define I2S_MAPPING_MODE I2S_MAPPING_MODE_OPTION_MAPPING_SOFTWARE`: the mapping function is call during the interrupt function that creates the memory buffer.
    * PRO: no need for more memory
    * CON: more CPU intensive

Depending on your specific need you can choose the option that is more suitable.

:arrow_forward: NB: `setMapLed(NULL)` will cancel the mapping. As you will see in `mapping.ino`

## 'HARDWARE SCROLLING'
Old term for a nice trick. The idea is to do a remapping of the leds within the driver directly so that the leds are displayed in another order. Pixels are pushed one at a time, and the normal way to do it is by going led 0,1,2,3 ....,N
Let's say that I want to 'scroll' by 5 pixels all the leds. Normally you would move leds 4->N-1 into 0,N-5 and then copy led 0=>led N-4 act. and then do the fastled.show().
The way I do it is to push within the driver led 4,5,6,7, ...., N-1,0,1,2,3 by calculating each time which pixels needs to be displayed using a simple algorithm about something along this `lednumber=> (lednumber+scroll)%N` (then a bit more complicated to take into account snake arrangement or not ,...)

#### `OffsetDisplay` object:
```C
struct OffsetDisplay
{
    int offsetx;
    int offsety;
    int panel_height;
    int panel_width;
    int image_height;
    int image_width;
    float scalling;
    int xc;
    int yc;
    float rotation;
    bool enableLoopx;
    bool enableLoopy;
    ....
};
```
At the initiation of the leds a default Offdisplay is created with certain values. You can get this default object with `getDefaultOffset();`.

I

#### Defining a panel
To be able to 'hardware scroll' in all directions you need to define how you panel is setup.
for instance if you have a panel 100 leds wide 20 leds height `panel_height=20` and `panel_witdh=100`.
If you are using mutilple strips you have two parameters 
NB: these parameters need to be put before `#include "I2SClocklessLedDriver.h"` :


#### `showPixels(OffDisplay offset)`:
This function can help you scroll your leds without doing a mem copy.

#### `showPixels(uint8_t * leds,OffDisplay offset)`:
Same function as before, where you can set the led buffer you want to display.

#### Is it reallly needed ?
Maybe not but fun (humm maybe not that fun lol) to make but great results.



 



## What is next ?










 
 
 
 
 

