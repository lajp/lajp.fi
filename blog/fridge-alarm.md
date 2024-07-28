---
title: Making a fridge door alarm system
description: How I made my fridge a nudge smarter by teaching it to scream whenever left open
tags:
- arduino
- electronics
published_date: 2023-09-10 11:51:53.752946197 +0000
layout: default.liquid
is_draft: false
---
# Side-project Sunday #1: Making my fridge a bit smarter

## Introduction

So I've recently moved into my own place, a small apartment close to the university campus.

My apartment is great and all but there seems to be one huge problem.
My fridge which I more often than not
<span class="definition" title="It's empty so the door doesn't have enough weight to slam shut on its own">fail to close</span>
is stupid enough to not have an alarm system to notify its users when it has been left open.

The only logical solution to <span class="definition" title="an engineering student">me</span>
was to come up with a system that sets up an alarm whenever the fridge has not been closed properly
(as opposed to just teaching myself how to close the fridge).

I thought that while I was at it, I might as well implement a system that logs the fridge-door usage for some potential future analysis.
Therefore I decided to increment a counter in the Arduino EEPROM every time the fridge is opened.

So this Sunday I woke up <span class="definition" title="around noon">early</span> and started building the contraption.

## The project

I've brought with me <span class="definition" title="some Arduino-microcontrollers and other basic electrical components">some hobbyist electronics</span>
from my childhood home and decided to make some use of them.

In case you would like to build it yourself, here is a (more or less) complete list of items I used:

* An Arduino-UNO board (Basically any Arduino or non-Arduino microcontroller board would work,
    but this is what I had and I couldn't get my Arduino-NANO working for some reason)
* A "piezo" buzzer
* A tactile switch
* A random piece of acrylic
* Some jumper wires
* Double-sided tape

Here's an attempt on a circuit diagram of the contraption (I'm not very good at drawing these)
A circuit diagram depicting my wirings

![A circuit schematic featuring an Arduino Uno, a switch and a piezoelectric buzzer](/static/fridge-alarm-circuit.webp)

## The code

I wrote, in total, two different Arduino-Sketches for this project. One for the actual alarm system and another one for managing and reading the statistic written in the EEPROM of the Arduino

They're quite simple scripts (as one would imagine for a project of this scale). The most notable thing is to remember to use the button pin in the correct mode: `pinMode(BUTTON_PIN, INPUT_PULLUP)`.

I added <span class="definition" title="completely optional">a delay</span> before the code starts playing the alarm because
I don't know if my neighbors would be very keen on listening to the beeping every time I open my fridge.
One could also expand the code to feature a rhythm or perhaps even a melody that would play when the fridge is left open.
I couldn't really manage this in my time frame and besides that I wanted the functionality over any aesthetics or other pleasantries anyways.

I've published the Sketches [on GitHub](https://github.com/lajp/fridge-alarm) and they're licensed under MIT so feel free to use them

## Conclusion and demo

Below is a demo-video of the project:

<video width="50%" height="auto" controls="">
    <source src="/static/fridge_alarm.mp4" type="video/mp4">
</video>

In conclusion I deem the project "a great success" out of 10

## Credits

I'd like to issue a special "thank you" to <span class="definition" title="the best electrical engineering student in Aalto">my friend</span>
for helping me with this project and providing some of the tools.
