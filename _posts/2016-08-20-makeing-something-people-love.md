---
layout: post
title: "Programming the generous tree"
description: "Building an art installation at Midburn"
category: Art
tags: ["Art","C++","Arduino"]
image: /images/gentree2.jpg
---
See that pretty tree on the left? Max and I made it. This is about how we did it. 

# Part 1 - The Experience


## How we started
[ws2812b]: /assets/img/posts/generoustree/ws2812b.jpg
Midburn, Israels Burningman, was approaching. I'd been yammering to all my friends about the virtues of programmable led strips for quite some time. Perhaps that's why my friend Roey called me, telling me of someone building some giant tree with thousands of leds that was looking for someone to program them. 

That's how I met one of the most talented Artists and executors I've met, Max Navot Meyers. Max indeed was building a 4 meter tall tree out of 1.5 liter plastic bottles and he planned to fill all of those bottles with led strips. Perhaps I could help with the programming?

Some people get excited by fast cars, some people get an adrenaline rush from roller coasters. Me, my eyes light up when I'm presented with thousands of WS2812b leds and an arduino.  And that is how Max and I started working together on the tree. 
![Excitement][ws2812b]
*The WS2812B led*


## We want smooth visuals

I've been programming long enough to have worked with non technical clients and one of the truisms I hold dear is that they have only a vague idea what they want and always assume they can do it as well as you, you are just saving them time. 

Having little patience for endless cycles Max and I agreed that he'd flesh out a written spec of what he wants, we'd review it together and do a single feedback iteration once I was ready. No ping pongs. I think Max deserves a lot of credit for taking me up on that, he had no idea he I am and who he was entrusting his baby to and he ended up almost sticking to the plan and giving me a lot of creative freedom. 

One of the core design concepts we agreed on was that we want things to be slow and smooth. I think Max was aiming to create a peaceful, almost sanctuary like piece, which meant that the animations on the tree would have to be slow and fluid.

## Being a service provider
[oren]: /assets/img/posts/generoustree/orenLighting.jpg
[conf]: /assets/img/posts/generoustree/confidence.gif
I'll get into the technical journey I faced in a moment but I'd like to spend a moment on my realization that as the technical/algorithmic side of the project I was very much a service provider. Throughout our work on the tree, I met Max three times and never saw the actual tree. Max on the other hand was bulding the actual object as well as hustling to gather and maintain  teem of volunteers that helped him with the tremendous amounts of labor involved in actually constructing this thing. 
![Oren testing the lights][oren]
*Oren testing the wiring for the tree*

I think that a major part of my job in this project, aside from actually designing and writing the code was to give max and the rest of the teem a true sense that my end of things were well handled and reliable. Being the technical part of something doesn't mean just executing you technical aspect in the best way possible, it means instilling the team with confidence and security that things are going on track. 
![conf][conf]
Its easy to be an arrogant programmer who brushes off the customer because he doest understand the nuances of whatever technical obstacle you just solved, but that leads to mistrust. One of my favorite parts about this project was that it was a team effort where each team member had unique skills. In that context keeping everyone informed and confidant that things were on track go a long way to keeping the working relations smooth. 

I think that that is a big part of being a good service provider, and something fundamental for being in a leading technical role with non technical people. 

## The best part
[firstTree]: /assets/img/posts/generoustree/firstTimeTree.jpg
[people]: /assets/img/posts/generoustree/peoplesitting.jpg

Midburn finally arrived. I had been in the desert a week, setting up our camp Shoobi Doobi before the organizers turned on the electricity and we could finish the lighting on the tree. 

This was a funny moment because we had been working on the tree for two months, but this was the first time I had seen it fully constructed. We tested the electronics before but we only had a rough guess of what things would work and how.
![The First time I saw it][firstTree]
*First time I saw the tree*


The code I had written was to be run off of an Arduino, a little $10 micro computer. Our first task was to get the code onto the Arduino, as I had never met the one that would be with us in the desert. And so, thats how on the first evening of Midburn I found myself with Max and my girlfriend Maria, on my knees in the sand transferring code from my laptop to the chip. It was a very surreal moment for me. 

And then the tree turned on. And it worked. And people came. I can't describe how exciting it is to see people wander into the desert and approach something you made and laugh and squeal with delight. People hugged the tree and told each other how amazing it was and we stood there and took pride in a lot of hard work.
![People sitting and enjoying the show][people]
*People enjoying the show*

# Part 2 - The technicals
This part is the geeky goodness, all about the things I did that got the lighting to work. 

I wrote the code in C++ as opposed to arduinos native language. I chose c++ for three reasons
1. I'm good at it. I know the language well and it's easy for me to work with.

2. Writing in C++ I have an easier time breaking my code into small files that are well defined, which makes it much easier to create and navigate a complex project. 

3. I used polymorphism extensively to keep the code manageable. The prime use case is in defining the different animations. An animation/pattern was defined in a base class *PatternBase* and each pattern implemented it. Then I could easily change patterns, color schemes and timing mechanisms separately. Aside from reducing complexity of the code base, this also allowed for a combinatorial growth in the number of different observed visualizations the tree showed.

Here is what the PatternBase.h file looks like. Each pattern had to implement updateLeds, which would write in the new values of the leds in the array.
{% highlight c++ %}

#include "stdint.h"
struct CRGB; //Forward declaration
class CRGBPalette16; //Forward declaration
class PalleteServer;
class PatternBase {
    protected:
        uint8_t brightness;
        CRGBPalette16* currentPallete;
        CRGB* leds;
    float brightNessBPM;
    uint8_t FPSIncrementer=0;

    uint16_t numLeds;
    uint8_t brightNessInc;
    void sinBrightNessInc();
    void FPSIncrementSin();
    void internalDelay();
public:
    uint8_t delayRate =0; //Zero means someone else should manage it.
    void updateFPS(uint8_t F) {FPS=F;delayRate=1000/FPS;}
    uint8_t FPS =25;
        bool hasPallete(){return currentPallete;}
        virtual CRGBPalette16* delPallete() { if (currentPallete) delete( currentPallete);}
        virtual void updateLeds() {};
        virtual void changePattern(CRGBPalette16* p){currentPallete =p;}
        virtual ~PatternBase(){ delPallete();}
    PatternBase(CRGB* l= 0,int16_t numLeds=-1);



};
{% endhighlight %}


## Tools

### Clion

Anyone who has used a JetBrains ide has enjoyed it. If you're writing C++ in something other than vim, you should defintly gibe Clions a try. Its fantastic.  Enough said.

### Arduino plugin

There is an Arduino plugin for CLion that makes writing to the Arduino board a breeze. You can find it [here](https://plugins.jetbrains.com/plugin/7889?pr=) and never think about configuring gcc or target architectures.

It's based in the [arduino-cmake](https://github.com/queezythegreat/arduino-cmake) project from [queezythegreat](http://www.null-dev.com/), which is a simple arduino solution for those of you that still love make.

### fastLED

The real enabler for this project and countless others is the amazing [fastLED](http://fastled.io/) project, a very extensive arduino library for led programming. It handles a plethora of different led drivers, as well as a lot of convenient functionality for working with different color spaces, timing and color pallets. 

Reading the fastLED code is also a lesson in hardcore programming, you'll find heavy use of techniques you don't see every day, especially compile time polymorphism. 

## Ideas
The basic flow of the program went something like this:

1. A single array was allocated. We had 24 led strips each with 30 controllers controlling 3 leds each. The array was then of size 24*30.

2. We assigned a particular section of the array to be outputted to each pin in the arduino, that is each led strip got information from the same 30 bytes in the array and only them. What changed was the information in the bytes.

### Setting up multiple strips with fastLED example
*globals.h*
{% highlight c++ %}
//
// Created by tal on 4/30/16.
//

#ifndef MBLIGHTS_GLOBALS_H
#define MBLIGHTS_GLOBALS_H

#include "FastLED.h"
#define TREE
#ifdef TREE
#define NUM_STRIPS 24
#define LEDS_PER_STRIP 50
#define NUM_LEDS LEDS_PER_STRIP*NUM_STRIPS
#define FIRE_DELAY 25
#else
#define BEAR
#define NUM_STRIPS 1
#define LEDS_PER_STRIP 150
#define NUM_LEDS LEDS_PER_STRIP*NUM_STRIPS
#define FIRE_DELAY 65
#endif
#endif //MBLIGHTS_GLOBALS_H
{% endhighlight %}

*mblights.ino*
{% highlight c++ %} 
#include "globals.h" // Includes fastled
CRGB leds[NUM_LEDS];
void setup() {
    delay(200);
    randomSeed(analogRead(A0));
    FastLED.addLeds<WS2812, 29,BRG>(leds,20*50, LEDS_PER_STRIP);
    FastLED.addLeds<WS2812, 30,BRG>(leds,1*50, LEDS_PER_STRIP);
    FastLED.addLeds<WS2812, 31,BRG>(leds,1*50, LEDS_PER_STRIP);
    FastLED.addLeds<WS2812, 32,BRG>(leds,2*50, LEDS_PER_STRIP);
    FastLED.addLeds<WS2812, 33,BRG>(leds,3*50, LEDS_PER_STRIP);
    FastLED.addLeds<WS2812, 34,BRG>(leds,4*50, LEDS_PER_STRIP);
    FastLED.addLeds<WS2812, 35,BRG>(leds,5*50, LEDS_PER_STRIP);
    FastLED.addLeds<WS2812, 36,BRG>(leds,6*50, LEDS_PER_STRIP);
    FastLED.addLeds<WS2812, 37,BRG>(leds,7*50, LEDS_PER_STRIP);
    FastLED.addLeds<WS2812, 38,BRG>(leds,8*50, LEDS_PER_STRIP);
    FastLED.addLeds<WS2812, 39,BRG>(leds,9*50, LEDS_PER_STRIP);
    FastLED.addLeds<WS2812, 40,BRG>(leds,10*50, LEDS_PER_STRIP);
    FastLED.addLeds<WS2812, 41,BRG>(leds,11*50, LEDS_PER_STRIP);
    FastLED.addLeds<WS2812, 42,BRG>(leds,12*50, LEDS_PER_STRIP);
    FastLED.addLeds<WS2812, 43,BRG>(leds,13*50, LEDS_PER_STRIP);
    FastLED.addLeds<WS2812, 44,BRG>(leds,14*50, LEDS_PER_STRIP);
    FastLED.addLeds<WS2812, 45,BRG>(leds,15*50, LEDS_PER_STRIP);
    FastLED.addLeds<WS2812, 46,BRG>(leds,16*50, LEDS_PER_STRIP);
    FastLED.addLeds<WS2812, 47,BRG>(leds,17*50, LEDS_PER_STRIP);
    FastLED.addLeds<WS2812, 48,BRG>(leds,18*50, LEDS_PER_STRIP);
    FastLED.addLeds<WS2812, 49,BRG>(leds,19*50, LEDS_PER_STRIP);
    FastLED.addLeds<WS2812, 50,BRG>(leds,20*50, LEDS_PER_STRIP);
    FastLED.addLeds<WS2812, 51,BRG>(leds,20*50, LEDS_PER_STRIP);
    FastLED.addLeds<WS2812, 52,BRG>(leds,20*50, LEDS_PER_STRIP);
    FastLED.addLeds<WS2812, 53,BRG>(leds,20*50, LEDS_PER_STRIP);

{% endhighlight %}




### Patterns and Pallets
Max and I wanted to get the tree to show many different behaviors, but we did not want to write them. The trick we used was to use many different color pallets but manage the active pallet separately from the logic that dictated the animation. That way each animation would cycle through 20 odd different coloring schemes, giving the appearance of many different visual behaviors in the tree. 


0. **PatternChanger** This was the ultimate coordinator in the code, that decided when to change a pattern and when to change a color scheme. The fundamental arduino loop essentially just called the pattern changer which decided what to do.
1. **Pattern** As I mentioned each Pattern derived from the base class **PatternBase** which defined the interface for a pattern. I dumped whatever logic I wanted into these and the PatternChanger just called the **updateLeds** function. The Patterns real job was to run some logic to figure out what the next state was in the led array. One of the cool things we did was have the pattern operate on the entire array but send different parts of it to different strips which made the movement in the true seem very "connected" and flowing. I show an example below.
2. **[CRGBPalette16](http://fastled.io/docs/3.1/class_c_r_g_b_palette16.html)** This is a fastled object, you give it 16 colors and it gives you a 256 color gradient by interpolating between what you give it. 
3. **PalleteServer** This was the singleton(ish) object in the code. An active pattern held a reference to it, and it would choose a color scheme of its own accord and give it back to the Pattern. The Pattern could ask for a new color scheme at any time, based either on external or internal logic
4. **PatternServer** Like the pallet server, except this served as a randomized factory for providing different animations. 


### Cellular Automata
I coded (and stole from the Internet) a bunch of different patterns but one that I'm most pleased with is based on the [Rule 30](https://en.wikipedia.org/wiki/Rule_30) cellular automata.

Rule 30 is a simple rule, where the value in each item in a binary array is inferred from the value of its neighbors, either live or dead. I used Rule 30 to decide if a cell should change color, and used fastLEDs color arithmetic to make subtle changes in the color of those cells who were triggered. 

The visual effect was a very obvious but very subtle changing in colors, where it was obvious that some pattern was at play but it couldn't really be found. I give credit to rule 30's chaotic nature and think its a great example of the appeal of mathematics in art. 

Here's the code

{% highlight c++ %}
Rule31Pattern::Rule31Pattern(CRGB *l, int16_t nl) :PatternBase(l,nl) {
    cells[0] =new bool[numLeds];
    cells[1] =new bool[numLeds];
//    cellColors = new uint8_t[numLeds];
    for (int i =0; i<numLeds; i++){
    //    cellColors[i] =100;
       cells[currentCell][i] = random(2);
    }
    currentCell=0;
}
Rule31Pattern::~Rule31Pattern() {
    delete cells[0];
    delete cells[1];
    delete cellColors;
}
bool Rule31Pattern::rules (bool a, bool b, bool c) {
    if (a == 1 && b == 1 && c == 1) return ruleset[0];
    else if (a == 1 && b == 1 && c == 0) return ruleset[1];
    else if (a == 1 && b == 0 && c == 1) return ruleset[2];
    else if (a == 1 && b == 0 && c == 0) return ruleset[3];
    else if (a == 0 && b == 1 && c == 1) return ruleset[4];
    else if (a == 0 && b == 1 && c == 0) return ruleset[5];
    else if (a == 0 && b == 0 && c == 1) return ruleset[6];
    else if (a == 0 && b == 0 && c == 0) return ruleset[7];
}
void Rule31Pattern::updateCells() {
    uint8_t nextCell = (currentCell+1) %2;
    for (uint8_t i =0; i<numLeds;i++){
        cells[nextCell][i] =rules(cells[currentCell][i-1],cells[currentCell][i],cells[currentCell][(i+1)%numLeds]);
        cellColors[i] += cells[nextCell][i] ? random(5) :-1*random(5);
    }
    currentCell =nextCell;
}

void Rule31Pattern::updateLeds() {
    uint8_t temp = beatsin8(1,10,20);
    CRGB color = ColorFromPalette(*currentPallete, 10, 255, LINEARBLEND);
    for (uint8_t i =0; i<numLeds;i++){
        {
            leds[i]=ColorFromPalette(*currentPallete, cellColors[i], 255, LINEARBLEND);
        }

    }
    updateCells();
}
{% endhighlight %}

Not code

## My Favourite fastLED Features
As I mentioned, fastLED is an amazing library that has tons of awesome features to make beautiful light art. I want to dive into two under advertised and super useful ones.
### Color Palettes

### Timing
