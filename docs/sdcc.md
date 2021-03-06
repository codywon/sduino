## Compiler

Tutorials:
http://www.cnx-software.com/2015/04/13/how-to-program-stm8s-1-board-in-linux/

STM8-Support only started with Version 3.4 in Ubuntu 14.10. For Ubuntu 14.4:

	add-apt-repository ppa:laczik/ppa
	apt-get update
	apt-get install sdcc

But even this version is fairly old and contains some known bugs. Better
download a current snapshot build from http://sdcc.sourceforge.net/ and
unpack it to `/opt/sdcc`. This requires a current version of libstdc++6:

	add-apt-repository ppa:ubuntu-toolchain-r/test
	apt-get update
	apt-get install libstdc++6

If you prefer to compile stm8flash yourself instead of using the Linux
binaries in the `tools` directory:

	git clone https://github.com/vdudouyt/stm8flash.git
	cd stm8flash
	make
	sudo make install

Download some example code:

	git clone https://github.com/vdudouyt/sdcc-examples-stm8.git
	cd sdcc-examples-stm8

The examples are meant for the STM8L, not the STM8S. This requires some
changes to account for the different pinout and register addresses (see below).
Finally upload the binary to the CPU:

	stm8flash -c stlinkv2 -p stm8s103?3 -w blinky.ihx



### Mixing assembler code with C code

c-code:

	stacktest(0x1234, 0x5678);

assember:

	push    #0x78
	push    #0x56
	push    #0x34
	push    #0x12
	call    _stacktest

resulting stack content (starting at [SP], using simulator sstm8):

	0> dch 0x17f9
	0x017f9 c0 80 ab 12 34 56 78 5b ....4Vx[

=> first paramter starts at [SP+3], MSB first.

#### Register assignment

**return values**:
8 bit values in A, 16 bit values in X, 32 bit values in Y/X (Y=MSB, X=LSB)

**register preservation**:
Not implemented for the STM8 (yet?). For some architectures SDCC implements
the possibility to mark a function that it does not effect the contents of
some registers:

	void f(void) __preserves_regs(b, c, iyl, iyh);


### Notes on SDCC

The linker `sdld` does not automatically link the object file for main.c if it
is part of a library. It must be part of the list of object files. (Important
for the build process with Arduino.mk)

Befehl `__critical{..}` sollte eigentlich den vorherigen Interrupt-Zustand
wiederherstellen, es wird aber einfach ein festes Paar sim/rim produziert.
Mit "push cc; sim" und "pop cc" klappt es im Simulator, aber nicht in der
Realität.

Für jeden benutzten Interrupt __muss__ ein Prototyp in der Datei stehen, in
der auch main() definiert ist. Aber für jeden Prototypen, für den es keine
Funktion gibt, ergibt einen Linkerfehler. Das erklärt den Sinn von stm8s_it.h
im Projektverzeichniss. Eine Arduino-ähnliche Umgebung muss diese Datei also
nach Analyse aller Sourcen selber erzeugen.

#### Simulator sstm8

Does not account for different cpu models (work in progress).
base address for UART1 is 0x5240, not 0x5230
TX and RX interrupt vectors 0x804C and 0x8050.


Compilieren: braucht libboost-graph:
libboost-graph1.54-dev - generic graph components and algorithms in C++  
libboost-graph1.54.0 - generic graph components and algorithms in C++  
libboost-graph1.55-dev - generic graph components and algorithms in C++  
libboost-graph1.55.0 - generic graph components and algorithms in C++  

#### Missing peephole optimisations

Directly connected sequences of 'addw x,#' and 'subw x,#' should be
combined into one operation.

Multiplication by two is done by 'mul' instead of a bitshift. Important for
array access.

**Interrupt routine preamble**:
Why is there a 'clr a/div x,a' sequence?

**Indirect 16 bit access**:
'ldw x,#addr/ldw x,(x)' should be 'ldw x,[addr]'

**Indirect function call**:
'ldw x,#addr/ldw x,(x)/call (x)' should be 'call [addr]'

Example (Interrupt routine calls a function from a jump table):

```c
static volatile voidFuncPtr intFunc[EXTERNAL_NUM_INTERRUPTS] = {
    nothing,
    nothing,
};

void TIM1_CAP_COM_IRQHandler(void) __interrupt(ITC_IRQ_TIM1_CAPCOM)
{
    intFunc[1]();
}
```


```asm
;	-----------------------------------------
;	 function TIM1_CAP_COM_IRQHandler
;	-----------------------------------------
_TIM1_CAP_COM_IRQHandler:
	clr	a
	div	x, a
	ldw	x, #_intFunc+2
	ldw	x, (x)
	call	(x)
	iret
	.area CODE
	.area INITIALIZER
__xinit__intFunc:
	.dw _nothing
	.dw _nothing
	.area CABS (ABS)
```


#### Missing compiler features

  - _ _preserves_regs() function attribute not supported
  - _ _attribute_ _((weak))
  - _ _critical{} generates sim/rim instead of push cc,sim/pop cc
  - dead code elimination: Does not recognize tables of const values. Using a
  const table would still pull in the whole object file, even when all
  accesses to the table have been eleminated by the optimizer. Only way out
  is to use `#define` statements instead.

