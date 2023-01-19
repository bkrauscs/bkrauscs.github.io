---
title: "Taming Swing"
date: "2023-01-19"
author: Brian
tags: ["java", "swing", "game"]
categories: GameDev
---

_The Five main ways I was able to get frames to render in 1-3 ms for a 2d game using Swing._

Swing is an outdated UI system for Java based on the even more outdated AWT system.  I chose to use it because I had experience with it and the book [Filthy Rich Clients](https://www.oreilly.com/library/view/filthy-rich-clients/9780132413930/) gave me the confidence I could do 2d drawing and animations with it.  It's backbone is the Event Dispatch Thread (EDT) where all drawing and event handling should happen.  I say should since it's really up to the programmer, but you run the risk of a crash or graphical issue if you disobey this.  My game loop first executes all game actions, then calls and waits for the EDT to repaint the window.  Events caused by input still happen outside of this redraw time, but the effect is only visible when the frame is rendered.  Assuming this happens 60 times a second, the delay is not noticeable.

#### How do I control when swing renders the JFrame?
This was an issue I dealt with for a while, because UI events would cause redraws outside of the game's render call, and slow down the system with needless work.  Thankfully, Swing provides a way to override the RepaintManager.  This manager is responsible for all repaint calls for top level Swing objects, like a JFrame.  I simply created a subclass of RepaintManager that did nothing for the overridden methods, and added a new method that only I knew about that does the rendering.  I then set the GameRepaintManager via `RepaintManager.setCurrentManager(gameRepaintManager);`.  Then my game loop can call the secret method on that instance and control precisely when the frame should render.

#### Stopping redraws while resizing the game.
Swing will try to redraw the frame while the user is resizing the window.  For the game that redraws on a fixed schedule this looks poor because it flashes between sizes once every 1/60 of a second.  Also, any computation based on the size of the screen changing happens many times quickly and can waste resources.  An early version of the game would OOM during this time for that exact reason.  Using `Toolkit.getDefaultToolkit().setDynamicLayout(false);` I can tell the frame to not update the drawn size of the window until the user has finished dragging to avoid these issues
Swing with hardware support  Swing by default does all rendering with the CPU.  Even for normal event based UIs this can be a problem.  The JVM lets you set a property where Swing will try to use a graphics device if available and submit primitive rendering calls to that device instead of computing the outcome with the CPU.  The code for that is  `System.setProperty("sun.java2d.opengl", "true");`.  If no hardware device is present and compatible, then Swing reverts to using the CPU.  By primitive rendering call I mean, given a Graphics2D object, calls such as `g.drawRect(x, y, width, height)`.  All of my custom drawing is done using those primitives or pre-rendered images drawn using `g.drawImage(image)`.

#### Double Buffering
In order to avoid screen tear, where the game is rendering data to the screen when the monitor is updating what it is showing, and shows the partial progress of the game render, we can set double buffering so that the game renders to an offscreen buffer and the monitor shows the onscreen.  When the game is done rendering, it flips the two pointers to the buffers and the monitor shows the recently finished buffer.  I did this with:
    
	JFrame frame = ...;
    frame.setVisible(true);
    final BufferCapabilities caps = new BufferCapabilities(new ImageCapabilities(true),
    new ImageCapabilities(true), BufferCapabilities.FlipContents.UNDEFINED);
    frame.createBufferStrategy(2, caps);

This sets the buffer strategy for the frame, and uses two accelerated (on graphics hardward if possible) images as the buffers.  The content of the "back" (or flipped out) buffer are undefined.  Not all systems support this though, so I try-catch for AwtException and set the buffer strategy with frame.createBufferStrategy(2); as a backup.  The frame must be visible to set the buffer strategy, for some reason.  It likely has to do with knowing which graphics device it is on, which it won't know until it is displayed.

#### Volatile Images
If I want to render a predrawn image onto the screen, I had typically used BufferedImages.  BufferedImages are images who's image data is in memory (buffered).  We can get a BufferedImage from loading an image file on disk, or creating one manually and drawing on it.  While the rendering operation is using the Graphics2D object, the calls were using up considerable CPU.  This is because the image data is in main memory and not graphics device memory, so the CPU is either copying the data to the device, or drawing it manually.  The solution is to use VolatileImages.  VolatileImages are volatile because at any time the contents could be lost, so you have to monitor the state of the image.  It can be lost because the image data exists on the graphics device, and the device may choose to purge the buffer or the user drags the window across monitors operated by different devices.  The upside of this though, is this happens infrequently, and while the volatile image data is not lost, drawing the image to screen is incredibly fast.  I made a wrapper class that checks the volatile image state and re-writes the buffer data as necessary before calling g.drawImage(image).  The wrapper class is given an implementation of an interface that can render the volatile buffer data on command, so if the data is lost in gameplay it can be refreshed.  The typical use is to provide the BufferedImage as a way of providing the volatile data.

#### Lessons Learned

 * I wouldn't use Swing for anything other than simple 2D graphics (like my game).
 * Keeping the built in event handling system (button presses, mouse-over) has made development much easier than using a full 3D system and re-writing my own event handler and layout manager.
 * Always render and change UI elements on the EDT.  I use SwingUtilities.invoke(Runnable r) with a lambda to wrap my code to make it easy.
 * Make life easier for yourself.  I use the GridBagLayout everywhere.  The constraints class is GridBagConstraints.  One of the oldest classes I have is called Gbc and just extends that class and makes the name shorter.  I then added convenience methods to create common constraints types quickly.  This turned working with the Swing UI from the thing I hated most and begrudged doing, to something that's pretty fast and I can iterate quickly.

Why not use an OpenGL Canvas?  I tried doing this, but as it turns out using the opengl library outside of what Swing provides with the graphics objects runs into resource contention issues.  The opengl library and swing have to share the device in order to render, and this slowed down the game enough that any increase in drawing with opengl was lost.

Why not use a game engine?  I didn't want to, I like learning how things work under the hood and this is a great experience.  If I was going to do a more serious game in 3D, I would absolutely use an engine. 
