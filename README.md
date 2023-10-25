# About

Python script for controlling BetaBrite PRISM USB-connected signs.  This project is a heavily-modified fork of jonathankoren's betabrite project.  You can find it here:  https://github.com/jonathankoren/betabrite

## Changes

My updates replace serial communication with USB.  They greatly enhance the functionality available from the command line, including command line args to:

* Set sign time and date.
* Initialize sign memory for STRING and TEXT labels.
* Clear sign memory.
* Return to normal run sequence after a PRIORITY (label 0) message was sent to the sign.
* Allow sending raw data to the sign for testing or command-line use of functionality not available within the script.  
* Some basic error checking is built in based on the BetaBrite Alpha protocol documentation.  

Additionally, I've changed the tagging methodology from +tag to \[tag\] (opening square bracket, tag text, closing square bracket).  There are no spaces in any valid tag.  I've also added additional tags to allow for some additional functions within your messages:

* Display the sign's date \[time\]
* Display the sign's time \[usdate\]
* Added the color blue as a format tag \[blue\]
* Added tags to allow STRING substitution within text labels.  This allows you to update values within messages without blanking the sign with each update. \[strA\]

# Usage

## Requirements
This script requires:

* Python 3.11 or higher (I used anaconda)
* pyusb

## Overview
To use the command-line implementation, see the examples below.  Note that for now, a space is required to separate each \[tag\] from neighboring tags or text.  This may change later, but for now it's required.  Please note for BetaBrite signs, you must initialize ALL TEXT and STRING memory and associate their labels before you can send any messages, with the exception of Label 0.  All must be set up at the same time; you can't come back later and "add one".  You must redefine all again.  Once you've initialized your message labels, send up your text, and then set the run sequence.  Please note message labels are case sensitive: "A" is not the same as "a".  

* Text labels can be at most 64K in size.  
* String labels are limited to 125 bytes.
* You can't configure memory for label 0.  Strings cannot use label "?".  
* Generally, this script limits you to labels a-z, A-Z, and 1-9 for configuring string and text memory.

**Please note:** This script does not directly support setting message run times or days of the week, though it could be done with RAW mode.

## General Init Procedure

Here's a basic procedure to initialize the sign memory, upload a couple of messages, and have set them all display in sequence.  This procedure includes a string substitution:

1. Clear the sign's message and string memory: ```sudo ./betabrite.py --clear a```
1. Initialize three messages as labels A, B, and C (255 bytes each) as well as two strings as 1 and 2 (100 bytes each): ```sudo ./betabrite.py --mode cfgmem t[A,255] t[B,255] t[C,255] s[1,100] s[2,100]```
1. Set the value of String 1: ```sudo ./betabrite.py --mode string --label 1 STRING```
1. Set message A: ```sudo ./betabrite.py --label A [green] [rotate] Hello from TJBChris [rotate]```
1. Set message B: ```sudo ./betabrite.py --label B [red] [rolldown] It is now... [hold] [rollup] [green] [time] [rotate] [amber] The date is...    [rollright] [blue] [usdate] [hold]```
1. Set message C: ```sudo ./betabrite.py --label C [amber] [rotate] This is message C with string A: [str1] [rotate]```
1. Set the run sequence (see protocol manual for info on the letters in the sequence): ```sudo ./betabrite.py --mode setsequence SUABC```

This should result in the above-three messages (one of which includes a substitution for the value of String 1) displaying on your sign.

## Message format
These signs are picky about the order in which formatting is presented.  Generally, your best bet for a given message is:

```[color] [transition] Whatever text you want [outgoingtransition]```

The date, time, and string substution tags can be used anywhere in the text section.  You can do multiple different formats in one labeled message by repeating the format above, as so:

```[color] [transition] Whatever text you want [outgoingtransition] [othercolor] [othertransition] Whatever text you want [otheroutgoingtransition]```

## Memory Config Tag Format
Tags used to configure memory have the following format:

**type**[**label**,**size_in_bytes**]

Where **type** is either **t** or **s** (for text or string, respectively), **label** is the message label you wish to assign (A-Z, a-z, or 0-9), and **size_in_bytes** is the number of bytes the sign should allocate for your message.  Note that strings can only be 125 bytes or less; text labels can be up to 64K (65535 bytes).  Example tags:

* A 1,000-byte text allocation as label 'F': ```t[F,1000]```
* A 75-byte string allocation as label 'q': ```s[q,75]```

## Tags
See the source code for a complete list of text formatting and color tags.  They can be found in the ```parse_text_message``` function.  Almost all are from the original author, with a few additions by myself.

## A Bit About Label 0
Label 0 is special in that it does not require pre-configuration, and in that it is the PRIORITY label.  Once set, its message will be displayed endlessly regardless of any pre-programming.  Therefore, you must clear that message or modify the run sequence to break out of it.  For this reason, I recommend avoiding Label 0.  It's less confusing that way.

## Raw Mode
You can use raw mode to send the sign sequences that can't be generated using one of this script's easy buttons.  For example, almost any functionality the sign is capable of can be accessed via raw mode.  This can be used to extend the functionality of this script without re-writing it, or for testing any functionality you're adding.  

As an example, you can make the sign sound a long beep with:
```sudo ./betabrite.py --raw E\( 0```

**E** is the character to indicate a special function to the sign, the **(** indicates the need to beep.  The leading \ is simply an escape character to allow the shell to pass the ( back to the script.  The trailing zero is a dummy data value which is required to pass the parser's check, but has no material effect.

## More Command Line Help
Help shows the overall format required to use this script.  It's shown below.  Please note that at least one dummy data element is required as the last command line argument even for those modes that don't use it (such as --clear or --runseq).  In this case, use ```sudo ./betabrite.py --clear a```

Please note that label is optional, but if not specified, the script defaults to A.  You use it only when setting text or string values.  Also note that --runseq, --raw, and --clear all exit immediately after running, overriding any other arguments specified.

The modes are as follows:

* **text** Updates the text of the label specified by --label (or A if none specified).
* **string** Updates the string contents of the label specified by --label (or A if none specified).
* **setdate** Sets the sign's date, using the format MM/DD/YY
* **setday** Sets the sign's day of the week (1=Sunday, 2=Monday, etc.)
* **settime** Sets the sign's time, using the format HH:MM (use 24-hour format for the time)
* **setsequence** Sets the sequence in which messages should be displayed.  The first two characters indicate run mode and lock status (for the IR remote).  Generally, using **SU** as the first two letters, followed by the remaining labels will work.
* **cfgmem** Configures text and string memory allocations.  This must be done for ALL string and text messages at one time.  If you need to update the config, you must send ALL again; you cannot add one-off allocations later without re-defining them all.  Once you configure the memory, you must re-send all data back to the affected labels.

The remaining args are described in the help text.

```
./betabrite.py --help
usage: betabrite.py [-h] [--mode {text,string,setdate,settime,setsequence,setday,cfgmem}] [--label LABEL] [--raw] [--runseq] [--clear] data [data ...]

positional arguments:
  data                  The tag-formatted message data or string value to send (settext, setstring) or data supporting a special function.

optional arguments:
  -h, --help            show this help message and exit
  --mode {text,string,setdate,settime,setsequence,setday,cfgmem}
                        Mode set: text, string, setdate, setday, settime, setsequence, cfgmem. See doc.
  --label LABEL         Text or string label: Which message or string (A-Z, 0-9) you wish to update. Default is A. Message 0 is the priority message and will
                        repeat until --runseq is used.
  --raw                 Allows sending of raw command code and matching data. Automatically adds packet header/footer. Requires at least two data parameters.
                        Ignores mode, label.
  --runseq              Tells the sign to resume running the sequence. Run this if the sign is stuck displaying the PRIORITY message (label 0). Ignores all
                        options. Requires one dummy data element.
  --clear               Clears all messages and strings. Ignores all other arguments. A dummy data element is required.
  ```

# Important Info

## Documentation
See the BetaBrite Alpha Protocol Manual for detailed documentation.  It is pracatically required reading for anyone who wants to understand the flow when working with these signs.  If you want more info or detail on anything I've said above, a quick Google search can find the doc.  I didn't link it here because it seems those links become invalid rather quickly.  It's a copyrighted document (and Adaptive is still in business), so I won't be sharing it here.

## Support
There is none.  This project is a quick-and-dirty side-of-desk project so I could make better use of my sign.  It was also something done as a learning exercise.  It is not perfect.  In fact, it is far from it.  I may decide to enhance/update/fix it.  Then again, some other project may grab my attention and I may never touch it again.  If it doesn't do what you want, feel free to do what I did, fork it, and modify it to your heart's content.  It won't protect you from yourself in every (or even many) cases.  It will probably not do exactly what you want, or do it the way you want.

It takes its roots from the parent project listed at the beginning of this document, so if you're thinking "why did they...", first look at the original code, and then realize that project was a good springboard for this Q&D fork to make my signboard "just go" with relatively little effort.
