---
layout: post
title: Somfy blinds with Raspberry Pi
categories: [raspberrypi,nodejs,electronics]
tags: []
description: Sample placeholder post.
---
I´ll begin with a short backstory.

I got motorized blinds installed during the summer but was a bit frustrated that i was not able to control them when I was not home. The point of installing them in the first place was to avoid having the house scorching hot many hours after the sun had gone down. So to solve this problem I had to do something.

Since I already use <a href="http://telldus.com/">Tellstick NET</a> to control the house i would like to integrate that system to control the blinds. That was however not possible since <a href="http://www.somfy.com/portail/index.cfm">Somfy</a>, which is the manufacturer of the blinds uses a proprietary protocol with rolling codes. I have seen some projects on GitHub that seem to address this but i went with the simple solution to start with.

So what I was aiming for, like many others on the internet, was going with controlling the remote directly via the output pins on their Arduino. However since attending Build 2015 I was lucky enough to get my hands on a free Raspberry Pi 2. So I thought, why not use that to control the Somfy remote. I set out to have somewhat the same configuration as many others with Arduinos on the internet, which essentially meant using an optocoupler so that you would not burn out the IO pins on your raspberry and controlling the Somfy remote directly by shorting the buttons on the remote.

You don´t need to be an expert on electronics when doing this, im certainly not but i know some of the basics.

Some of the more essential parts i used was:
<ul>
	<li>Raspberry Pi B+</li>
	<li>LTV847B DIP-16 optocoupler x4</li>
	<li>4x resistors 0.25W 330ohm (330R)</li>
	<li>wires</li>
	<li>Prototype shield</li>
</ul>
First off i set out to solder everything together, start with connecting one button on the remote all the way through. What I mean by that is solder all the wires and components together and hook them up to the correct IO-pin on the raspberry. When you are done with this you can try and run the code to check if you did it right. The upside with doing it this way instead of soldering everything is that you can check if it works, if it doesn´t there is no need to fix all the wireing again.

This is a picture of the board for the components.

<a href="http://www.hibbisoft.com/wp-content/uploads/2015/08/WP_20151110_09_47_56_Pro.jpg"><img class="alignleft size-medium wp-image-292" src="http://www.hibbisoft.com/wp-content/uploads/2015/08/WP_20151110_09_47_56_Pro-300x235.jpg" alt="WP_20151110_09_47_56_Pro" width="300" height="235" /></a>

As you can see im not the best at soldering, but as long as it works i guess its fine. :)
Instead of using traditional wires i used an old IDE-cable from a harddrive that connects nicely to the raspberry pi:s IO-pins. You need to be careful though and check that the cable actually has 40-wires. Some IDE-cables have 80-wires with 40-pins and they might damage your Rasperry Pi. I then soldered some pins to the prototype board which made it much easier to connect the wiring. It can be somewhat tricky with the IO-pin layout so i strongly advice to print out the Raspberry Pi IO-layout and keep it with you during soldering.

To sum it up i basically is IO-pins connected to an optocoupler where resistors limits the current for the optocoupler. The reason i am using an optocoupler is so that we will not power the Somfy remote directly from the IO-pins and we will eliminate interference from the Raspberry Pi. The optocoupler i used was with 4 different sensors, but you can also find other sizes and they will work fine.

&nbsp;

<img class="wp-image-311 alignright" src="http://www.hibbisoft.com/wp-content/uploads/2015/08/WP_20151110_09_47_10_Pro1-401x1024.jpg" alt="WP_20151110_09_47_10_Pro" width="295" height="753" />

&nbsp;

This is a basic schema for a optocoupler. So we power up the LED in the picture with the output from our Raspberry Pi which triggers a Phototransistor and lets power through from the 3,3V pin
connected on the remote to our ground on the Raspberry Pi. This means we will short the button on the remote. With the optocoupler we are essentially isolating the IO-pin from the Somfy remote.

<img class="size-medium wp-image-261 alignnone" src="http://www.hibbisoft.com/wp-content/uploads/2015/11/intcircuit-300x103.gif" alt="intcircuit" width="300" height="103" />

As you can see on my board i have also prepared to read the analog pins on the raspberry so that i can handle channels on the Somfy remote. I haven´t implemented anything on that yet so i will not go into changing channels on the remote at this time. This is the remote hooked up with all the wires. By connecting the the 3,3V ouput PIN on the raspberry to the remote there is no need for any battery. I also soldered a wire from the remotes internal antenna to increase range.

&nbsp;

&nbsp;

And now for some code. I made a really simple implementation of the server code without UI and such in node.js. It essentially is a webserver wich has some endpoints for controlling the blinds via HTTP. Inorder to get it to work with Tellstick NET you need to have the Premium subscription so that you can call external urls based on events. This is the temporary solution though. My long term goal is to be able to use a 433Mhz reciever connected to the raspberry and intercept a blinds code that i set up for another brand on the Tellstick. Interpreting that signal and then pressing the buttons on the Somfy remote accordingly.


[javascript]

var express = require('express');
var app = express();
var sleep = require('sleep');

var Gpio = require('onoff').Gpio,
upButton = new Gpio(18, 'out'),
downButton = new Gpio(24, 'in'),
stopButton = new Gpio(14, 'out'),
selectButton = new Gpio(23, 'out')
var standardButtonPress = 60000; //microseconds

app.get('/', function(req, res) {
     res.type('text/plain');
     res.send('');
});

app.post('/up', function(req, res) {
     res.type('text/plain');
     res.send('going up');
     upButton.writeSync(1);
     sleep.usleep(standardButtonPress);
     upButton.writeSync(0);
});

app.post('/down', function(req, res) {
     res.type('text/plain');
     res.send('going down');
     downButton.writeSync(1);
     sleep.usleep(standardButtonPress);
     downButton.writeSync(0);
});

app.post('/stop', function(req, res) {
     res.type('text/plain');
     res.send('stopping');
     stopButton.writeSync(1);
     sleep.usleep(standardButtonPress);
     stopButton.writeSync(0);
});

app.post('/select', function(req, res) {
     res.type('text/plain');
     res.send('changed channel');
     selectChannel();
});

function selectChannel() {
     selectButton.writeSync(1);
     sleep.sleep(2);
     selectButton.writeSync(0);
}

app.listen(4747);

function exit() {
     stopButton.writeSync(0);
     stopButton.unexport();
     upButton.writeSync(0);
     upButton.unexport();
     downButton.writeSync(0);
     downButton.unexport();
     selectButton.writeSync(0);
     selectButton.unexport();

     process.exit();
}
process.on('SIGINT', exit);

[/javascript]


A more descriptive schema of the hardware.
<a href="http://www.hibbisoft.com/wp-content/uploads/2015/08/SomfyProxy-schematics1.png">
</a><a href="http://www.hibbisoft.com/wp-content/uploads/2015/08/SomfyProxy-schematics1.png"><img class="size-large wp-image-341 aligncenter" src="http://www.hibbisoft.com/wp-content/uploads/2015/08/SomfyProxy-schematics1-1024x873.png" alt="SomfyProxy schematics" width="720" height="614" /></a>

The reason we use sleep() is that it seems the remote does not react to the button press unless it is pushed in for a set amount of milliseconds.We also need to close down the application correctly when exiting the process or the io-pins on the raspberry might end up in a pushed state.

<a href="http://www.hibbisoft.com/wp-content/uploads/2015/11/IMG_2864.jpg"><img class="alignleft size-medium wp-image-241" src="http://www.hibbisoft.com/wp-content/uploads/2015/11/IMG_2864-300x200.jpg" alt="IMG_2864" width="300" height="200" /></a>

I hope it will at least inspire someone out there to create more cool gadgets.
