# ledger-app-ardor

This is the official Ardor wallet app for the Ledger Nano S and X

## More documentation

You can use [This]: https://buildmedia.readthedocs.org/media/pdf/ledger/latest/ledger.pdf as a resource for info
Also Ledger has a slack channel where you can ask questions

## Debug Prints

In order to have the printf functions work from the code, you need to install debug firmware 
https://ledger.readthedocs.io/en/latest/userspace/debugging.html

2. turn on the debug in the make file - make sure not to commit this

todo fix this one

## How to switch between different target build

todo write content here

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

## Compiler warnings

Make sure you have handled all warning that come out of calling make clean and make load before release
todo: We might want to raise the warning level, some day
You can ignore warnings coming out of OS library files, curve25519_i64.c, curveConversion.c and the aes folder
since they are externaly imported

## Javascript lib

You use lib found [here]: https://gitlab.com/haimbender/ardor-ledger-js in order to comunicate with the app, some test examples can be found there too


## Stackoverflow canary

To get the amount of memeory used in the app call the following:
readelf -s bin/app.elf | grep app_stack_canary 
that will output the canary (which is at the end of the memory space) location then subtruct 0x20001800 from it to get the actuall used up space for the app
the NanoS has 4k of memory for the app and the stack #todo rewrite and fill in for NANOX

0xda7a0000 is the start address for the nanoX

The app uses the SDK's built in app_stack_canary, it's activated in the makefile by the define HAVE_BOLOS_APP_STACK_CANARY
I advise to keep this flag always on, it just gives extra security and doesn't take up much CPU.
The way this works is it defines an int at the end of the stack, inits it at startup and then check's against it every call to io_exchange,
if it changes it throws an EXCEPTION_IO_RESET, which should reset the app.
In order to debug a stack overflow call check_canary() add different parts of the code to check if you have overflowed the stack


## Error handling

Errors are propegated though the call stack and it's the command handler's or button handler's job to respond accordingly -> Clear the state if they manage it
and return the error back to the caller

All return values for functions should be checked in every function


## How key derivation works

We wanted to create some kind of derivation scheme simular to BIP32/44 for Ardor, but since it's using EC-KCDSA on Curve25519,
we had to get creative with already existing schemes.

The solution we came up with was to ride on top of [BIP32-Ed25519 Hierarchical Deterministic paper](https://cardanolaunch.com/assets/Ed25519_BIP.pdf) derivation scheme for ED25519 on a twisted edwards curve using SLIP10 initialization on 512 bits master seed from bip39/bip32 24 words, which is supported on most platforms, then converting the key pairs from ED25519 on the twisted edwards curve to EC-KCDSA on Curve25519 (It's complex, you might need to read it a few times, it's ok)

Just to recall for the readers who don't know, a public key is a Point(X,Y) on a curve C, X,Y are integers modolu some field F with a base point on the curve G
(C, F, G) define "curves", in this paper we are dealing with the twisted edwards curve and curve25519

There is also a morphe function between the twisted edwards curve (remember, we are talking about the actual curve itself, base point and field) and the curve25519
such that: if Apoint = Knumber * BasePointED25519 on the twisted edwarads curve then morphe(Apoint) = Knumber * BasePointECKCDSA on curve255119
Implementation for this function can be found in curveConversion.c

ED25519 Keys are defined as: PublicKeyED25519Point = CLAMP(SHA512(privateKey)[:32]) * ED25519BasePoint

Lets refer to CLAMP(SHA512(privateKey)[:32]) as KL

The derivation composition flow is for path P:

1. os_perso_derive_node_bip32 derives KLKR and chaincode for P (will explain later why we need this) acording to the [BIP32-Ed25519 Hierarchical Deterministic paper](https://cardanolaunch.com/assets/Ed25519_BIP.pdf) using SLIP10 initialization on 512 bits master seed from bip39/bip32 24 words
2. derive PublicKeyED25519 using cx_eddsa_get_public_key and KL, the point is encoded as 65 bytes 04 XBigEndian YBigEndian
3. PubleyKeyED25519YLE = convert(YBigEndian) - just reverse the bytes
4. PublicKeyCurve25519X = morphe(PubleyKeyEED25519YLE)

Points on Curve25519 can be defined by just the X point (cuz each X has only 1 Y), so PublicKeyCurve25519X and
KL should hold PublicKeyCurve25519X = KL * Curve25519BasePoint

In EC-KCDSA publickey = privatekey^-1 * BasePoint (don't ask me why the private is in reverse), privateKey ^ -1 is refered to as the keyseed, so KL is the keyseed for the PublicKeyCurve25519X public key for path P, whop, we've done it!

Extra Notes:

* ED25519 public keys are usually compressed into a Y point in little endian having the MSB bit encode the parity of X, (since each Y has two possible X values, X and -X in feild F which means if one is even the second is odd)

* In order to derive public keys outside of the ledger (Masterkey derivation), all we need is the ED25519 public key and chaincode, described in the derivation scheme

* Code for the derivation implementation can found in [Python](https://cardanolaunch.com/assets/Ed25519_BIP.pdf) //todo add javascript and javfa implementations

* You can read the actuall paper on derivation [here](https://www.jelurida.com/sites/default/files/kcdsa.pdf)

* The curves don't really look like curves, just a cloud of points