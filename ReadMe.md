# Sorc-80 - How To Run TRS-80 Machine Language Programs Natively On The Exidy Sorcerer

This repository seeks to document my 1981-era method of porting TRS-80 Model I machine language software (namely games) over to the Exidy Sorcerer computer. In essense it is a fairly simple process when presented properly.  From 1981 to 1983 I was utterly fixated on trying to turn the Exidy Sorcerer into a Tandy TRS-80 Model I computer so that it could run native programs without any changes. Please refer to my [mirror repository](<https://github.com/rcl9/Sorc-80--How-To-Run-TRS80-BASIC-Programs-Natively-On-The-Exidy-Sorcerer>) whereby I document my 1981-1983 work to port TRS-80 Level II BASIC over to the Sorcerer. 

<div style="text-align:center">
<img src="/Images/Android-Nim.webp" alt="" style="width:75%; height:auto;">
</div>
<br>Android NIM running on the Exidy Sorcerer + Level II BASIC.
<br><br>

There are two approaches that can be taken to port TRS-80 machine language programs, making it either a simple or a bit more complex porting process:

1) Modify the Exidy Sorcerer's hardware with a (relatively) simple change which will automatically remap any access to TRS-80 video memory over to that of the lower 16 lines of the Exidy Sorcerer. I used this method for most of my ported game applications. 

2) Or, modify the machine language program itself at the bit level and with the inclusion of some support porting routines of mine. I followed this approach to port Sublogic's **[T80-FS1 Flight Simulator](<https://www.trs-80.org/t80-fs1.html>)** over to the Sorcerer; it has been available to the general Exidy Sorcerer community since 2011 when I first made it available to the author of the then-new jSorcerer emulator to test his system.

## Addressing the main 'Elephant in the Room' issue - Video Display Mapping (via hardware changes)

Let's address the main issue at hand. The TRS-80 uses a 64x16 memory mapped video display located at 3C00H-3FFFH. On the other hand the Exidy Sorcerer uses a 64x40 memory mapped video display located at F400H-F7FFH. At least the width of each line is identical at 64 characters.

The best method to address this issue is with the addition of a hardware change to the Exidy Sorcerer computer itself:

<div style="text-align:center">
<img src="/Images/TRS80-Circuit-Modification-for-Exidy-Sorcerer.webp" alt="" style="width:75%; height:auto;">
</div>

When the front panel SPDT switch is in "Sorcerer mode" then no changes happen to the video circuitrty. But when the switch is flipped over to  "TRS-80 mode":

- Any time a program accesses 3C00H-3FFFH in memory then pin 8 of the 74LS30 goes low to signal that we are accessing TRS-80 video memory.

- The video circuitry is enabled by intercepting the Sorcerer's /F000 line.

- Pin 8 of the 3D (A11) is brought low so that screen accesses to 3C00H-3FFFH will be transferred to F400H-F7FFH.

Accessing 3C00H to 3FFFH in the Exidy Sorcerer is then exactly the same as accessing its main video memory at F400H-F7FFH. In TRS-80 mode, the Sorcerer's first 24 lines from F080H to F3FFH are unaffected. 

## A Quick Primer About the Runtime 'Memory Map'

A key part to achieving a successful port is to understand how the TRS-80 memory map works for our intended purposes. In simple terms, the first 12k of memory from 0000 to 2FFFH is the Level II BASIC in ROM. The next 4K from 3000H to 3FFFH is TRS-80 video memory and other hardware mapping (which we can use for our own purposes). The remainder of memory from 4000H to 7FFFH (32k upper limit) will be available for the machine language program or a BASIC file. Our start-up [support code] will be placed after the end of the program (usually at 8000H).

As a side note that is explained in my [mirror repository], CP/M requires access and control over much of page zero (from 0000 to 00FFH). That's a big problem because the TRS-80 Level II BASIC ROM also sits in page zero. Back in the 1981-83 era (and even in modern times) I had to play a lot of games and tricks to overcome this issue. 

## Overview of the Porting Process

Using the following procedure a well versed Z80 machine language programmer will be able to convert a TRS-80 Model I machine language program to run on the Sorcerer. An exception is any programs over 16k in length or any requiring disk I/O. Note that I said MODEL I  because the TRS-80 Model I Level II BASIC-in-ROM is not compatible with MODEL III machine language programs in general but that may not be important if the machine language program does not have any dependencies on the BASIC in ROM.

I would highly recommend you acquiring the book ‘Microsoft Basic Decoded’ by James Farvour. Please do not buy ANY OTHER! I have many other books describing the innards of the TRS-80 Level II BASIC but this IS THE BEST by any standards. It will save you hours and hours of debugging and conversion problems as it contains a complete and accurate disassembly of the Level II BASIC. Please note that the scanned PDF copies leave out the all-important assembly listing section.

Another criteria for conversion is the need of 48k of memory (minimum), CP/M and ZDT.COM or SID.COM (the Z80 version of DDT). 

- It is assumed that you have a binary copy of the TRS-80 machine language program which is to be converted. 

- [This step might be optional] First, we need the "host" TRS-80 BASIC operating system that the game might depend upon for function calls. In a real TRS-80 Model I computer this is the 12k Level II BASIC in ROM located at page zero. In your hex editor load up "trs80v2.com" at 0100H which is my port of the 12k Level II BASIC that can run on the Exidy Sorcerer and with CP/M in memory. Note: my "**air-raid.com**" port did not require BASIC in memory whereas my other ports of software do have it in memory.

- Merge your machine language program to 4000H via your hex editor. 

- Change 0100H to be a jump to 8000H via 'C3 00 80', or whereever you have chosen to place your custom support code. This will cause our newly ported program to immediately jump to our support code upon start-up. 

- Lastly look over and get to know my supplied [support runtime code]. What you need to do is modify it for your specific application then insert the Z80 compiled binary code to 8000H, or any other location that you deem to be 'safe' for its continued execution. You will need to change the "XXXX" address in the .mac file with the start-up execution address of your machine language program- this may be found on the cassette label in decimal or in addresses 40DFH (LSB) and 40E0H (MSB) of your program.

- Following my methods in the "Program-Level Changes" sections below. 

- Save out the final file from your hex editor as the first runtime candidate. Jump to 0100H to execute.

## 1) Program-Level Changes -- Addresses from 3C00H-3FFFH

The most tedious part of the porting process will be to locate and change all of the TRS-80 video mapped memory accesses over to that of the Exidy Sorcerer. I did that will a diassembler (or the 'L' command in SID.com) to carefully find and document those memory accesses. 

Accesses to addresses 3C00H to 3FFFH in the program probably point to the TRS-80 memory mapped screen. These are to be changed, when come across, by adding an offset of B8H to the MSB of the screen address. Thus they will now point to the Sorcerer screen. Two examples are: 3C00H would change to F400H and 3FFFH would change to F7FFH. The TRS--80 screen accesses will be transferred to the lower half of the Sorcerer screen. Watch out though as some addresses could just be a counter for a loop or odd data. Here are three examples.

```
	TRS-80 	      ------>>>> 	Sorcerer 

	LD A,(3C00H) 		LD A,(F400H)
	ADD 80H 			ADD 80H
	LD (3C00H),A 		LD (F400H),A

	LD HL,3DF0H 		LD HL,F5F0H
	LD DE,3F10H 		LD DE,F710H
	LD BC,0020H 		LD BC,0020H
	LDIR 				LDIR

	LD DE,3D4FH 		---> We must be careful in this case.
	LD A,(49AFH)	 		DE looks like it points to the
	AND D 					TRS-80 screen but in fact just
	OR E 					holds two constant values and not a screen address.
```

## 2) Program-Level Changes -- Memory mapped keyboard addresses

The TRS-80 machine has a memory mapped keyboard whereas the Sorcerer has a port-I/O keyboard for which a machine language program must scan the keyboard to see if any key was pressed. Thus, we must somehow emulate a memory mapped keyboard on the Sorcerer. The way I have got around this is to write a small driver (found in the [support code]) which, when given a TRS-80 keyboard location in memory, will search for that key in the Sorcerer keyboard port then set a bit in A register that corresponds to the key in the TRS-80’s memory. 

Instead of changing addresses or adding calls to make this work I have come up with a very easy solution. We use RESTARTS. 

```
	TRS-80 					Sorcerer 

	3A 10 38 	LD A, (3810H) 		EF 	RST 28H
							10 38 	DEFB 10,38
```

The 3810H comes from the chart found in the ‘KEY(X)’ conversion chart in the TRS-80 Level II BASIC’s documentation. On the left the program is looking at the keys in memory location 3810H. All we have done to convert it is add an ‘EF’ in place of the ‘3A’. When my subroutine is called with the RESTART 28, it pops the stack and the fetches the 2 byte keyboard address from the memory locations after the ‘EF’. Therefore you only have to change one byte per keyboard access. 

Note that this method only works when the program is accessing the keyboard through the ‘A’ register (which is standard procedure). I have once seen a programmer use HL register to hold the keyboard location, then a ‘LD A,(HL)’ to get the contents of that location. The only way around such a situation is to delete this routine and rewrite it using ‘EF XX YY’ in the space provided between 8070H to 80DFH or whereever room is available in high memory then placing a call to this routine from the old routine in the program.

Please refer to the KEY$() command chart in my [mirror repository](<https://github.com/rcl9/Sorc-80--How-To-Run-TRS80-BASIC-Programs-Natively-On-The-Exidy-Sorcerer>). The addresses you must look out for when searching through the program are:

	3801,3802,3804,3808,3810,3820,3840,3880

Location 38FFH is non-zero when any key on the TRS-80 keyboard is pressed. You could replace this check with a call to the monitor’s keyboard input routine. Any key pressed on the Sorcerer would automatically set the Zero flag which is equivalent to what they do when location 38FFH is looked at and OR’ed.

## 3) Program-Level Changes -- SOUND routines

I have saved you a bit of trouble by incorporating an OUT command controller in the [support code]. When you come across a ‘D3 FB’ or ‘D3 FF’, then just replace it with ‘FF 00’. This causes a RESTART 38 which jumps to my sound driver, which in turn sends out the appropriate byte to port FF on the Sorcerer. Sound will come from bit 2 of the parallel port. Use the sound interface found on the [schematic sheet].

## 4) Program-Level Changes -- Cassette I/O changes

We now should have the machine language part of the program converted to the Sorcerer. One small exception may be programs such as EDTASM that require cassette I/O. The following changes will convert the cassette from the TRS-80 tape interface to the normal Sorcerer 300/1200 baud interface. First find the cassette portion of code in the program. Then make the following changes:

WRITE data: change the SENDNULLS call (027E or 0284) to E2C2.
 
	- Change tape byte output call (0264) to E2EE.

READ data: change the TAPEWAIT call (0293) to E759.

	- Change tape byte input call (0235) to E2DA.

Go through the entire program and check for any more cassette routines. I have found in more than one instance two sets of routines. If the addresses in the brackets don’t match the ones in your program then check out your addresses in the ‘Microsoft Basic Decoded’ book and insert appropriate Sorcerer calls. Any calls to turn on or off the cassette recorder can be left alone as I have already converted them to run on the Sorcerer.

## 5) Program-Level Changes -- Data area table conversions

The last part of the conversion is to check the data areas for any tables that may point to the TRS-80 screen. Some programs I have come across, such as ASYLUM, have these tables. They can be determined by a screen address (ie. 3DF2) then some characters and another screen address. Also an entire table of addresses may be found (ie. ...FF 3C 10 3F 44 3E ....). To change these, find the start and end of the table then change the MSB byte of the address by adding the usual screen offset of B8H to make it go to the Sorcerer screen (FF 3C --> FF F4, 10 3F --> 10 F7, etc.). Go through all these data areas and make similar changes. This is the worst area to make mistakes in. If we accidently changed a piece of data that really is a part of the machine language program then it will probably crash when it is run. This type of bug is next to impossible to fix so write down all the changes you make - it may help in the long run.

A better way of finding where the data tables are is to disassemble the program and search for code that is dealing with data. Here are two examples:

```
1) 	LD HL,5200H
	LD DE,3C00H
	LD BC,0200H
	LDIR

2) 	LD IX,4950H
	.
	.
	LD L,(IX+0)
	LD H,(IX+1)
	LD (HL),A
	INC IX
	.
	.
```

In example #1 the data table is definitely holding data for the screen since it is being moved to 3C00H. We can see that it starts at 5200H and ends at 5400H. So we are not to touch this table at all.

In example #2 we can deduce that 4950H could point to a data table containing two byte screen addresses. Looking directly at the data table would confirm our suspicions. User discretion IS YOUR GREATEST TOOL in converting these programs. You will get better and get to know how such TRS-80 programs are converted as time goes on.

I would suggest, from my experience, that after each section you should save the program and try running the program. After section 1 you should see something on the screen. Section 2 should allow you to play some of the game, and section 3 and the data area changes should finish up the package.

If it doesn’t work, which is 90% probable, go through all the fixes and see what could be wrong. The conversion process may take anywhere from 2 hours to infinite time (there’s some programs I’ve never been able to get working!). If is works the first time - congratulations you are the first person ever to accomplish such a feat!

## Example Converted Programs

I had used the techniques in this tutorial to port [several TRS-80 machine language games](</Examples/Example_Ported_Programs.zip>) to the Exidy Sorcerer in the early 1980s. However, all of them required the use of my hardwar circuit to run except for the FS1 Flight Simulator which will run on any Exidy Sorcerer.

1) [FS1 Flight Simulator](<https://www.trs-80.org/t80-fs1.html>) - **flitsimu.com**. This port does not require the TRS-80 BASIC runtime system. At startup (0100H) the program is relocated to 4000H and then a sub-set of my "start-up routine" code executed at 7500H. I have a PDF file documenting all of the byte-level changes I made to the program to allow it to run natively on the Sorcerer. 

2) The other programs require my hardware circuit to be enabled and hence will not run on a generic Exidy Sorcerer. Each of them has my [modified TRS-80 BASIC] incorporated as the runtime system (from 0 to 2FFFH). The following table provides a short overview of them:

| [Program Name](</Examples/Example_Ported_Programs.zip>) | Offset to 'Start-Up Routine'  | 
| :---: | :---: | 
| Air-raid.com	| 8000H |
| Asylum16.com	| 4200H |
| Asylum32.com	| 4200H |
| Defense.com	| 8000H |
| Eliminat.com	| 8000H |
| Robot.com	| 8000H |

## See also

- [Microsoft BASIC Decoded & Other Mysteries](https://www.msxarchive.nl/pub/msx/mirrors/msx2.com/sources/trs80basic.pdf). Note that the 'all important' chapter 7 is missing from this PDF scan, as it is the actual source code listing of the Level II BASIC. 

- [Radio Shack Level II BASIC reference manual](https://ia803209.us.archive.org/31/items/Level_II_BASIC_Reference_Manual_1st_Ed._1978_Radio_Shack/Level_II_BASIC_Reference_Manual_1st_Ed._1978_Radio_Shack_text.pdf)

- If you want to try porting a TRS-80 machine language game to the Exidy Sorcerer then Wayne Westmoreland [released all of their TRS-80 games](<http://www.trs-80.org/games-by-wayne-westmoreland-and-terry-gilman.html>) into the public domain in 1995. 

