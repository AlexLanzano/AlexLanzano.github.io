---
layout: post
title:  "Linux Kernel Driver Development for Fun and (Non)Profit"
date:   2024-09-04 16:29:09 -0400
categories: Linux Kernel Driver Development Mentorship
---

# Writing Linux Drivers for Fun and (Non)Profit

In this post I will go over my experience with writing two drivers for the Linux Kernel.
But first...

## Mentorship Experience
I want to give a shout out to the Linux Kernel Mentorship Program. I did this work while enrolled in the Linux Kernel 
Bug Fixing Program which was an amazing experience. One of the main reasons why I enrolled was to actually put a deadline
on myself to do this work because I'm the type of person to start 50 million projects at once and never see them through.
First and foremost you have access to highly experienced Kernel Developers who have been slinging kernel code since before
I was learning my ABC's. Pretty much any scenario I encountered where I was a bit stuck they'd be more than happy to help out
or point me to the right person to ask. They also provide extensive resources into various debugging techniques, how to contribute,
potential areas that could use development. Oh, one thing I should mention is that even though it's named "Bug fixing", you can
contribute whichever area of the Kernel interests you most. I decided on writing drivers because I like to cause bugs rather
than fixing them. Also, once in the program one essential resource I used that helped me a bunch was the discord server, specifically
searching for questions similar to mine from previous years. It also is a great way to connect with people if you're interested in
pursuing this as more than just a hobby. But please if you're interested please sign up. Here's a link to get you started.
https://wiki.linuxfoundation.org/lkmp

## The Drivers

Alright now onto the meat and potatoes of what I actually did. Ultimately, I wrote two drivers one for a monochrome eInk display made
by Sharp, the other is a driver for an Inertial Measurement Unit (IMU for short) which is just a little sensor that includes an accelerometer
and gyroscope.

### Sharp Memory Display

So on the first draft of this blog I wrote out a huge detailed overview of how the display operates but when I read it back
I almost fell asleep and it's not anything that isn't better represented in the data sheet. 
So, instead of boring you with a huge deep dive on how the hardware works, I'll give you a high level view of the various quirks of the display and how it relates to the driver.

- Display operates over SPI
- Maintains internal memory for pixel data
- Requires additional VCOM signal to be present

#### Putting pixels on the screen
So, it all starts when userspace tells the kernel that the framebuffer for your display has been updated. This is usually called from your window manager which keeps track of the "dirty" regions of your display. So, a request is made to the kernel with the specific region or regions of the framebuffer that the display needs to update with. This will hop through the DRM subsytem, eventually landing in sharp_memory_fb_dirty(). Here the driver will take in the updated pixel data from the framebuffer, create a SPI message with the pixel data, and finally send it to the display to be updated. And that's it!

#### The VCOM signal and keeping pixels from dying

This display requires a signal (VCOM) specific to Sharp Memory Displays to be fed into it externally. It's needed so the 
pixels of the display don't get stuck in one state, I.E. killing your pixels. The datasheet describes two ways this can be handled. The first method is to handle it in software 
using the "Maintain Data" command. So, you pretty much periodically send a specific SPI message to the display toggling the VCOM bit bit. The second method is to handle it completely in hardware by attaching a signal to the physical VCOM
pin on the display. The signal could be a clock generator, crystal oscillator, PWM gpio, etc.


Being the gracious person I am, I decided to implement support for either option. Even going above and beyond to support a third option for to be driven by a PWM enabled GPIO pin.



### Bosch BMI270 Inertial Measurement Unit

Despite the various health articles I had to sift through when googling this device, this was a pretty nice device to work with.

Alright, as stated above this device is a little sensor that measures acceleration, rotation, and angular velocity.
These devices are primarily used to track motion. For example an IMU was used to track all those missed swings in Wii Sports.

Learning my lesson from rewriting the data sheet for the Sharp Memory Display I'm deciding to do a similar high level view of the device.
- Operates over SPI or I2C bus
- Requires firmware binary blob to be loaded to it during initialization
- Data is read and written to via internal registers on the device

#### Using regmap 

Like most devices like this, they operate via internal registers to do various device configuration and read out sensor data. And being the gracious hardware engineers they are at Bosch, they decided to support either SPI or I2C communication to read and write these registers. Luckily for me there's a very nice kernel abstraction that makes it very easy to add driver support for either communication method.

The regmap abstraction is pretty much just a standardized way to represent and access memory mapped registers in kernel drivers. So you don't need to write out the read and write functions yourself, which would get old really quickly. It also takes care of the SPI or I2C bus comms for you.

With that, I ended up with a three file setup for this driver. First, bmi270_i2c.c which contains the regmap_config struct which details the register layout, the i2c probe function and some other device structs for the DTS. Second, the bmi270_spi.c, which is identical to the i2c version except containing the SPI specific regmap_config struct. Lastly is the bmi270_core.c, this is where the meat of the driver is. After I2C or SPI specific configuration happens, both drivers will call into this core file to complete device configuration using the regmap abstraction.


#### Loading blobs from userspace

One thing specific to this device is that it requires an external binary blob to be read into it during device initialization. Since this blob is pretty large I'd be a laughing stock of the Linux community if I stuck it in the driver code itself. Luckily for me and my reputation, there's a nice API to load files from userspace for this very purpose. Using the request_firmware(), and subsequently release_firmware(), I can request for the kernel to look for a specified file name and load its contents into a buffer. The kernel will look for the file by default in /lib/firmware/ but you can configure additional paths if you'd like. If the file can't be found it will return an error that you can handle in your driver. Lastly, when you're all done with the file you should call release_firmware() to free the buffer and clean up for you.

#### Accessing data in userspace via sysfs

There's multiple ways to get the sensor data from userspace. The following will be the simplest and probably incorrect way to do it.When you create and iio (Industrial I/O) device such as this, you configure various "channels" which detail the specific sensor data you can read from the device, in our case its acceleration and angle velocity from the accelerometer and gyroscope respectively. And since this is a 6-axis device there will be a channel for each dimension (X, Y, Z). So, when the device is all configured and ready to go the kernel will expose a sysfs interface (usually at /sys/bus/iio/devices/iio:deviceX/) that you can read from to get the sensor data for a specific channel.

## Closing Remarks

Writing Linux kernel drivers has been a wild ride, honestly. Getting to see these devices come to life and actually work is a feeling I can't even begin to describe.

One of the biggest takeaways for me is just how much of a team effort the kernel community is. Even though I was writing these drivers solo, I always felt like I had a safety net with the resources, mentorship, and the overall community vibe. It’s really cool to know that people with decades of experience were ready to help someone like me figure things out.

If you’re reading this and you’ve been thinking about contributing to the kernel, just do it. Pick some hardware you’ve got lying around (or something you’re interested in), start tinkering, and don’t stress too much about making everything perfect right away. Kernel devs love to review patches—trust me, they’ll definitely tell you if something needs fixing!

And hey, maybe one day your code will be running on some device out in the wild, too. How cool is that?

That’s all from me—thanks for sticking around. Now, go write some kernel code, and most importantly, don’t forget to have fun doing it.
