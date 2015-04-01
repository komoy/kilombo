# Introduction
This tool is a simulator for kilobot robots. The kilobot c program is compiled natively on the host computer where the simulator is to be run. The robot program is linked with a simulator library, which provides the functions from the kiloblib API. Since the robot program is compiled to native code, the simulator is very fast. 

The simulator uses SDL for grapical output. All configuration and result files are written in JSON.

The program is distributed under the *WHICH LINCESE??* license, with no warranty.
The simulator was developed in the (SWARM Organ)[www.swarm-organ.eu] project.

Contact: 
- Fredrik Jansson fjansson@abo.fi
- Matthew Hartley 


# Installation
Installation instructions are found in the file `README.md` in the root directory of the project.

# Adapting code
The simulator aims to be compatible with the [kilolib API.](https://www.kilobotics.com/docs/index.html)

Initialization, messaging, motion, and random number generation work with the same function calls in the simulator and in the real kilobot. Two topics that require special care when using the simulator are 
global variables and timing.

## Variables
Kilobot C code usually makes use of static variables (to allow these to persist across repeated calls to the user supplied function) or global variables.  These variables demand special treatment when run in the simulator.  The simulator handles all robots in a single program, so a global or static variable ends up being common to all robots. A workaround implemented in the simulator is to keep all global variables inside a `struct`, defined in the robot's header file. 

    typedef struct 
    {
        int N_Neighbors;
        ...
    } USERDATA;

When used in the simulator, the robot program declares a  USERDATA *mydata;. The simulator ensures that this pointer points to the data of the correct robot before calling any of the user program's functions. Additionally, the simulator needs to know the size of the USERDATA structure (however, it does not need to know the contents of it, and it will never change it). The size of the structure is communicated to the simulator by declaring a variable UserdataSize, and setting it to the size of the structure:

#ifdef SIMULATOR
int UserdataSize = sizeof(USERDATA);
USERDATA *mydata;
#endif

In the program, the variables can then be accessed using `mydata->N_Neighbors`. The simulator ensures that `mydata` points to the data of the currently running bot. Note that local variables (i.e. regular variables defined inside a function) can be used in the usual way. 

When the program is compiled for a real kilobot, the same way of accessing the variables can be used, provided that the program declares one instance of the `USERDATA` structure. 

    #ifndef SIMULATOR
      USERDATA myuserdata;
      USERDATA *mydata = &myuserdata;
    #endif


## Timing and delays 
The simulator does not implement the `delay()` function at all, since it would be difficult.  The delay() function exists, but returns immediately. The simulator simply calls the bot's main loop  function  once every simulator time step, for every bot. The main loop function is the one specified when calling `kilo_init()`.

Note that in programs for real kilobots, the kilolib API documentation states that it is best to use `delay()` only for short times, like when spinning up motors, and that  for timing the bot's behaviour, one should instead use the global variable `kilo_ticks`. `kilo_ticks` is incremented 31 times per second, and is  implemented in the simulator as well.


## Data types
A difference between the AVR c compiler used for the kilobots and the native c compiler used when compiling with the simulator is the size of datatypes. For example, `int` is 16 bits on the AVR and 32 bits on a standard 32 or 64 bit PC. This should normally not be a problem, unless integer overflow is used on purpose. However it may lead to code working as intended in the simulator while overflowing on the kilobot.
A solution is to explicitly specify the size of the types, e.g. declaring variables as `uint8_t i`. This is good practice on the AVR anyway, since it lets the programmer conserve RAM memory by using the smallest possible type.


## Callback functions
For convenience, the simulator implements a system of callback functions for communication with the user-specified robot program. The callbacks are registered by calling `register_callback(type, function_pointer)`, typically at the beginning of main() before calling `kilo_init()`. All callback functions are optional. If the bot does not register them, they are mot used.

|type  | function definition  |  use | 
| ------------- |-------------| -------|
|`CALLBACK_PARAMS` |   `void  callback_F5(void)` |  Reload configuration parameters from file. Called once, not for every bot, when F5 is pressed.|
| `CALLBACK_RESET`      |   `void  callback_F6(void)` |  Reset bot. Called for every bot, when F6 is pressed. |
| `CALLBACK_BOTINFO` | `char *botinfo()` |   Return a string describing the internal state of the current bot, used for the simulator status bar.|
|`CALLBACK_JSON_STATE` | `json_t* json_state(void)`| Return a json object describing the bot's internal state. Used to store snapshots of the simulation. |

 Since the callback system is present only in the simulator and not in the real kilobot, the callback functions and the call to register_callback() should be conditionally compiled. An example:

    #ifdef SIMULATOR
    register_callback(CALLBACK_BOTINFO,  (void(*)(void))botinfo);
    #endif

`(void(*)(void))` here is a typecast to avoid compiler warnings about the function type. The functions need to be defined with return types as specified in the table above. For an example of the botinfo callback, see `examples/orbit/orbit.c`.


# Controls

The following keybindings are active during simulation:

* ESC: terminate simulation
* Numpad +: zoom in
* Numpad -: zoom out
* Arrow keys : move display
* Space : pause/resume simulation
* Mouse-left: drag bots around
* Mouse-right: rotate a bot
* s   : Take screenshot. The screenshot is stored as screenshots/<bot name><number>.bmp
* F1  : disperse the bots
* F2  : compress the bots
* F5  : Call the bot's parameter reload function (if implemented in the bot)
* F6  : Call every bot's reset function, (if implemented in the bot)
* F11 : Toggle full speed simulation (no delay between frames)
* F12 : Toggle fast communication (message passing every kilotick

# Configuration file settings

The following keywords are recognized in the simulator JSON configuration file:

* `botName` : the name of this bot type
* `randSeed` : a random seed, for repeatable simulations
* `nBots` : number of bots to simulate     
* `timeStep` : time step. Also determines the displayed frame rate (currently, will be configurable)
* `simulationTime` : how long to run the simulation
* `showComms` :  whether or not to draw a line between each pair of bots in communication range, whenever a message is passed between them.
* `showCommsRadius` :  whether or not to draw a circle for the communications range of each bot
* `distributePercent` :
* `displayWidth`  : absolute width of the window, pixels
* `displayHeight` : absolute height of the window, pixels
* `showHist` : whether to show the paths the robots have moved
* `imageName` :  File name for storing images during the simulation. Format example: `movie/f%04d.bmp`,
  where the `%...d` will be replaced by an increasing number.

#Command line options
* `-p parameterfile.json` : Simulator parameters. Mandatory
* `-b bots.json` : starting positions for the bots. Optional.

At the end of the simulation, the simulator stores the final state of the bots in a file named `endstate.json`. This file can be given as a starting state for the next simulation, simply copy it to a new name, and pass that name to the simulator with the -b option. Thus the simulator can be used as an editor of bot starting configurations as well.

# Implementation decisions
The following are some justifications for the way the simulator was implemented, and some thoughts on other ways to do it.

There is already a kilobot simulator implemented in Python, kbsim by Antti Halme, https://github.com/ajhalme/kbsim .  This simulator implements a physical model of the kilobots, with motion, collisions and messaging between neighbors. However the program controlling the robots in the simulator is written in Python, making the programming significantly different from programming actual kilobots.

We wanted the simulator to run the same program as the actual kilobots, to make the simulator useful not only for trying out an algorithm in principle but also for testing an actual kilobot implementation. 

* A model of the physical world, where the kilobots have a position, a direction, and can move.
* An implementation of the kilolib API, coupled to the physical model
    * motion
    * LED color
    * communication
    * sensor input
* A way to run the programs of all robots in parallel, and synchronizedly

The kilobot API specifies that the user program should be written as a function, which is called repeatedly by the kilobot library as long as the robot is in the RUNNING state. 
A simple way to simulate many robots in parallel is to sequentially execute the loop function for each robot. This means that the robot program should be written in a way that is independent of how long it takes to execute the loop function. Also, it means a delay in the middle of the loop function is difficult to implement. We found that these limitations are possible to deal with in practice. First, timed events should be implemented using timers, not using delays. The kilobot API defines a varible kilo_ticks, which is incremented 31 times a second. The simulator implements kilo_ticks as well. This variable can be used for measuring times and for waiting for specific lengths of time. In the simulator, it is possible to control how many iterations of the loop function are run, before the kilo_ticks variable is incremented. This implementation means that the time it takes to run the loop function is not accurately simulated. Effectively, the simulation assumes that the loop function runs instantly. 

In order to simulate delays in the middle of the loop function, it would be possible to run the programs for all robots in parallel as separate threads. The threads would then need to be synchronized with each other and with the physical simulation, e.g. using barriers. We decided that for our application, this level of simulation is not needed, and therefore we did not pursue this option further. If the robots are running as separate threads the program could take advantage of multiple processor cores, but on the other hand the synchronization of the threads would introduce some overhead as well.

On the extreme end of the realism spectrum, it would be possible to run the same microcontroller program as the kilobot runs in an AVR simulator, such as `simavr`. Every robot would run in one instance of the AVR simulator, and all these instances would be coupled to a physical model of the world where the kilobots are present. This would however require a detailed physical model of the kilobot, on the level of how individual IO pins on the AVR control the motors etc.  For messaging to work, the separte AVR simulator instances would need to be tightly synchronized. This approach seems possible in principle, but was deemed to be too complicated to implement. It would result in a more accurate simulation, free of the restrictions on delay functions and the `mydata->` access to variables, but would also introduce a significant overhead in simulating the AVR microcontroller instead of compiling the robot C-language program to native machine code.





