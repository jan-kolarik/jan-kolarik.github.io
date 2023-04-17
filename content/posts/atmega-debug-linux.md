---
title: "Debugging Arduino without additional hardware"
date: "2023-02-16"
summary: "Using Visual Studio Code to debug programs for Arduino Mega 2560"
tags: ["arduino", "vscode"]
ShowCodeCopyButtons: true
ShowToc: true
ShowReadingTime: true
ShowPostNavLinks: true
---

![arduino-logo](/posts/images/arduino-logo.png "Arduino logo")

## Intro

On a bit different note, recently I was trying to help my father with adding
debugging support for Arduino Mega 2560 microcontroller board without need
to use any additional hardware kit. 

For Windows, there is a great [tutorial](https://www.codeproject.com/Articles/5150391/Creating-and-Debugging-Arduino-Programs-in-Visual) for that, but we need to setup several things differently in the Linux environment
and I didn't find any easy-to-use guide for that.

So I want to share my approach with you, definitely nothing world-shattering, but could be useful for anyone with 
similar intention or might save you some time from being stuck in one place for too long.

---

## Environment

- Fedora Linux (but should be applicable to any other distro)
- [Visual Studio Code](https://code.visualstudio.com/) + Arduino extension
- [Arduino CLI](https://github.com/arduino/arduino-cli)
- [avr_debug](https://github.com/jdolinay/avr_debug)

---

## Building and uploading

First we need to setup the building of our project and make it possible
to upload the binary into the board.

### Prepare VS Code

Install the **Arduino CLI** tool to provide an all-in-one solution for
Arduino boards.

Then we need to install & enable the **Arduino** extension in the VS Code.
In the extension's configuration we need to modify some fields:
- Arduino: Command Path (`arduino.commandPath`) -- set `arduino-cli`
- Arduino: Path (`arduino.path`) -- set `/path/to/your/arduino-cli/binary`
- Arduino: Use Arduino Cli (`arduino.useArduinoCli`) -- set `true`

Initialize the Arduino project with the **Arduino: Initialize** command, f.e.
by selecting it from the command palette using the **F1** key in VS Code. 
This will generate the empty `.ino` source file and the default `arduino.json`
extension config for our project in the `.vscode` folder. You will also need to **rename**
this existing `.ino` file to match the name of the project.

Open the **Arduino: Board Manager** command and install the **Arduino AVR Boards**
package to add support for our microcontroller board.

Using the bottom status bar in VS Code, select the **Arduino Mega** board type.

Now you can try to compile the empty project using the **Arduino: Verify**
command which is also available under the icon in the editor's top bar.

### Deploy a simple program

To be able to upload our program to the actual board, it might be needed
to add appropriate user rights by adding the current user to the
`dialout` or `tty` group:

```bash
sudo usermod -a -G dialout $USER
```

It will need a logout or restart.

Connect the board to the USB and select the `/dev/ttyUSB0` port
from the bottom status bar. 

We should be able now to upload the program to the board with the
**Arduino: Upload** command.

To see if it is really working you can add some simple blinking
to the `.ino` source:

```c
void setup() {
    pinMode(LED_BUILTIN, OUTPUT);
}

void loop() {
    digitalWrite(LED_BUILTIN, HIGH);
    delay(1000);
    digitalWrite(LED_BUILTIN, LOW);
    delay(1000);
}
```

---

## Debugging

Here we'll add support for debugging our board from the VS Code
using just the connected USB cable.

### Setup the debugger

We will prepare the **GDB** debugger that will be running on our
local computer and the **avr_debug**, remote stub which will be
deployed to the ATMega and communicating with the **GDB** to 
allow debugging from VS Code.

Unfortunately, for Fedora Linux, there is no maintained package 
with **GDB** for **AVR** architectures anymore, therefore we need to
build it by ourselves. Luckily it's very simple.

Download the latest **GDB** [sources](https://www.sourceware.org/gdb/download/), 
compile it for target **AVR** architecture and install it in the defined
directory like this:

```bash
./configure --target=avr --prefix=${HOME}/Programs/avr-gdb
make
make install
```

Then unpack the **avr_debug** library into the Arduino user libraries
directory at `${HOME}/Arduino/libraries/avr-debugger`. Put there just the
library [sub-directory](https://github.com/jdolinay/avr_debug/tree/master/arduino/library/avr-debugger)
from the default branch.

### Configure the project

Include the debugging library into our project by going to the
**Arduino: Library Manager**, filtering the `avr-debugger` and
clicking the **Include Library** button.

Now we have the sources prepared for adding the debugging support
which is as simple as adding the `debug_init()` function at the
beginning of the `setup()`:

```c
#include <app_api.h>
#include <avr_debugger.h>
#include <avr8-stub.h>

void setup() {
    debug_init();
    pinMode(LED_BUILTIN, OUTPUT);
}

void loop() {
    digitalWrite(LED_BUILTIN, HIGH);
    delay(1000);
    digitalWrite(LED_BUILTIN, LOW);
    delay(1000);
}
```

Last thing we need to do is to create a debugging configuration
for VS Code, so it knows what we want to debug and how.

I am sharing my `launch.json` example below. The key parts there:
- `program` points to our built program which is the `.elf` binary in the `build` folder
- `miDebuggerPath` addresses the **AVR GDB** binary we have built
- `setupCommands` section tells **GDB** what port and baud rate to use

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Debugger launch",
            "type": "cppdbg",
            "request": "launch",
            "program": "${workspaceFolder}/build/example.ino.elf",
            "cwd": "${workspaceFolder}",
            "externalConsole": true,
            "MIMode": "gdb",
            "miDebuggerPath": "${userHome}/Programs/avr-gdb/bin/avr-gdb",
            "setupCommands": [
                {
                    "description": "Set remote serial baud",
                    "text": "set serial baud 115200",
                    "ignoreFailures": false
                },
                {
                    "description": "Attach to serial port",
                    "text": "target remote /dev/ttyUSB0",
                    "ignoreFailures": false
                }
            ]
        }
    ]
}
```

You can put it in the `.vscode` subfolder and modify it according to
your environment.

### Let's add some breakpoints!

And so now we can verify and upload our program and then debug
the running program in VS Code as with any other project. Just
setup the breakpoints and then attach to the board using the
**Run and Debug** from the Activity Bar.

**Note:** there is one important downside of this approach and
it's that no `Serial` functions could be used when the debugging stub 
is attached. You could use the conditional compilation to
enable them when not debugging. For more info, refer to the
upstream project of the **avr_debug** author.

---