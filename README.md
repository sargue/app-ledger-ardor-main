# ledger-app-ardor

This is the official Ardor wallet app for the Ledger Nano S and X

## Enforcments to hold as a developer

There are a few things that a dev must make sure the app is doing, and there is no way to enforce this in code
bacause of the platform limitations, so it's desribed here.

Because we use a union for all of the command handler's states (called state, defined in ardor.h) in order to save memory, the app is vulnerable to
an attack in which the state is filled using one command and then exploited using a different command's interpretation of the same state
so when switching between command types we HAVE TO MAKE SURE WE CLEAR STATES, this is done by the isLastCommandDifferent parameter which is passed
to the handler functions, when ever it's true, the handler has to clear the state first

As an extention to the paragraph above, STATE HAS TO BE CLEARED whenever we get some sort of an error in a handler function which manages state

## Code design:

Includes dont contain includes to other includes, meaning the .C files have to contain includes to all the dependencys to other includes
this just makes things easier

Consts and hard coded things are stored in config.h

txnTypesList.c is autogenerated by createTxnTypes.py from txtypes.txt, this list is generated from the java code
and the whole process is used to sync between the 2 Ledger App code and Java implementation

The code flow starts at ardor_main which is a try/catch (global exception catch, so that the app wont crash) loop on io_exchange,
waiting for the next command buffer and the calling the appropriate handler function which is implemented in the different C files


## APDU Protocol

Commands are in the format of

	0xE0 <command id byte> <p1 byte> <p2 byte> <sizeof the databuffer byte> <data buffer>

Responce buffers are usually in the form of

	<return value byte> <buffer> <0x90> <0x00>

Return status meanings can be found in returnValeus

## Compilation

If you compiling for the first time or making a change to list of txn types

	 python createTxnTypes.py > src/txnTypesList.c

To compile just call

	make

In order to compile an upload onto ledger call

	make load

## Javascript lib

You use lib found [here]: https://gitlab.com/haimbender/ardor-ledger-js in order to comunicate with the app, some test examples can be found there too


## Stackoverflow canary

The app uses the SDK's built in app_stack_canary, it's activated in the makefile by the define HAVE_BOLOS_APP_STACK_CANARY
I advise to keep this flag always on, it just gives extra security and doesn't take up much CPU.
The way this works is it defines an int at the end of the stack, inits it at startup and then check's against it every call to io_exchange,
if it changes it throws an EXCEPTION_IO_RESET, which should reset the app.
In order to debug a stack overflow call check_canary() add different parts of the code to check if you have overflowed the stack