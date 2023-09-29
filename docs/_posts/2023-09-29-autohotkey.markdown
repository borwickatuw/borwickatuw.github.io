---
layout: default
title: "AutoHotKey Excel + web site scripting"
---
[AutoHotKey](https://www.autohotkey.com/) is a Windows automation tool. It is a great tool of last resort when you have a monotonous task that cannot be automated in a better way. (If an API is available for what you're doing, use the API rather than AutoHotKey!)

It *is* possible to use [Chrome.ahk](https://github.com/G33kDude/Chrome.ahk) to script Chrome. However, sometimes web sites don't respond well to Javascript and you need to automate the keystrokes/clicks as if you actually did them yourself. That's the context for the below.

AutoHotKey scripts can be a form of "autonomation:" AutoHotKey is doing the boring parts but a human is still involved. You may want to design your scripts with some flexibility so that a human can still make judgment calls, e.g. if the script breaks. For example, I bind parts of the scripts to different F keys, e.g. I hit `F1` and then do some work and then hit `F2` to run the second script.

Also consider defensive programming: you may want to double-check that you have the correct window open using something like `WinWaitActive`, before sending a bunch of keystrokes. This helps if the user accidentally runs the script in a different state than you were expecting.

Here are a few patterns I've found helpful:

# Setting a standard screen layout #
 
Before building any scripts, build a standard screen layout. I use "Windows split screen" for this. I hit Windows key + Left arrow to make one app exactly the left half of the screen, and then I select the other app to be exactly the right half of the screen.

For me the left side is the browser and the right side is the Excel data I want to use.

# Preamble #

If I used AutoHotKey more I'm sure I'd have more than this. This makes the window title search use substrings and uses coordinates relative to the active window:

    SetTitleMatchMode, 2
	CoordMode, Mouse, Window
	
# Reading from Excel #

This takes the active cell from whatever Excel window is open if you click on the Excel icon.

    Excel := ComObjActive("Excel.Application")
	MyVar := Excel.ActiveCell.Value

# Conditional code at beginning #

If you want to do a search on a web site, and there's a chance you're not on the page that lets you do a search, you can check it:

    If WinExist("Sub Page Web Name")
	{
		WinActivate, Sub Page Web Name
		; Do the code that gets you to the search page,
		; e.g. search for `back`
	}

# "Resetting" the page #

If you're on a web page but you're not sure whether you're at the top, go to the top:

    Send {PgUp 10}

This hits the page up button 10 times.


# Waiting for the background not to be grey #

Maybe when you click on something, the web page background turns grey until the content is loaded. You can check for this:

    PixelGetColor, BackgroundColor, 123, 123
	While BackgroundColor != "0xFFFFFF"
	{
	    Sleep 1000
		PixelGetColor, BackgroundColor, 123, 123
	}

Use "AutoHotKey Window Spy" to find the coordinate (listed as `123, 123` above). Based on the preamble, above, use the "Window" coordinate value (vs. another coordinate such as the absolute screen position).

This checks for whether the pixel is white (`#0xFFFFFF`). It sleeps a second, checks again, and loops until the pixel is white.

# Entering a search in a web page search #

After "resetting the page" per the above:

    Click 123, 123
	Sleep 1000
	Send ^a {Del}
	SendInput,%SearchString%
	Send {Up}{Del}
	Send {Enter}

This:

  - clicks in the search box (coordinates `123, 123` in the example)
  - sleeps a second
  - sends ctrl+a DEL (which selects all text and deletes it)
  - types in whatever is in the `SearchString` variable
  - Sends up arrow and hits delete (which I needed because AutoHotKey was inserting a space at the beginning)
  - Send Enter to search

# Using Find to click buttons #

If you want to click on a certain button via a browser UI, you can use the browser's find command:

    Send ^f
	Sleep 1000
	SendInput button text goes here
	Send {Esc} {Enter}
	
This sends control+f (find), then sleeps in case the browser takes a sec to open the find window, then types in `button text goes here`, then hits escape, and then hits enter. Escape exits the find search and enter then clicks on the text that was found. At least in Chrome, if the text is on a button then it's clicked!

# Binding scripts to F keys #

When you're ready, you can add a hotkey for your script e.g.:

    F2::
	
on a line by itself prior to the rest of your script will bind the script to the `F2` key.

# Moving to the next cell #

The easiest way to move Excel to another cell is to use arrow keys:

    WinActivate,, Name Of Excel File
	Send {Down}

This works great if you want the user to be able to select the column and then the script just moves down that column.

However, if you need a certain column, you might consider doing something like:

    Send {Left 25}
	Send {Down} {Right 4}
	
This sends left arrow a whole bunch of times, which hopefully will get you to the first column of the cell. It then goes down a row and over 4 columns.
