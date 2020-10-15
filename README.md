lgt不能用avr-gcc自带的`_delay_ms()`和`_delay_us()`，因为指令周期不同，延时严重不准。
arduino的`delay()`是基于系统时钟的，是准的，但`delayMicroseconds()`是阻塞的。官方并没有修正它的代码，所以它是不准的。
用下面的代码替换`wiring.c`中的相应代码，即可让`delayMicroseconds()`一个周期也不差(由于系统时钟使用了timer中断，实际会有几us误差，但是用`cli()`关闭中断之后就不会有任何误差了)：
```C
/* Delay for the given number of microseconds.  Assumes a 16 MHz clock. */
void __attribute__ ((noinline)) __attribute__((optimize("O1"))) delayMicroseconds(unsigned int us)
{
    // call = 2 cycles + 2 to 4 cycles to init us(2 for constant delay, 4 for variable)

    asm volatile ("nop\n\t"); // 1 cycle
    asm volatile ("nop\n\t"); // 1 cycle
    asm volatile ("nop\n\t"); // 1 cycle
    asm volatile ("nop\n\t"); // 1 cycle

#if F_CPU == 16000000L
    // for the 16 MHz clock on most Arduino boards

    // for a one-microsecond delay, simply return.  the overhead
    // of the function call takes 14 (16) cycles, which is 1us
    if (us <= 1) return; //  = 3 cycles, (4 when true)

    // the following loop takes 1/4 of a microsecond (4 cycles)
    // per iteration, so execute it four times for each microsecond of
    // delay requested.
    us <<= 2; // x4 us, = 4 cycles

    // account for the time taken in the preceeding commands.
    // we just burned 19 (21) cycles above, remove 5, (5*4=20)
    // us is at least 8 so we can substract 5
    us -= 5; // = 1 cycle
    asm volatile ("nop\n\t"); // 1 cycle
#else
    #error only support F_CPU == 16000000L
#endif

    // busy wait
    __asm__ __volatile__ (
        "1: sbiw %0,1" "\n\t" // 1 cycle
        "nop\n\t" // 1 cycle
        "brne 1b" : "=w" (us) : "0" (us) // 2 cycles
    );

    // return = 2 cycles
}
```

下面是这段代码编译后的汇编代码：
```asm
; nop(4)
$011C  NOP
$011E  NOP
$0120  NOP
$0122  NOP

; if (us <= 1) return;
$0124  CPI R24, $02
$0126  CPC R25, R1
$0128  BRCS $013C

; us <<= 2;
$012A  LSL R24
$012C  ROL R25
$012E  LSL R24
$0130  ROL R25

; us -= 5;
$0132  SBIW R25:R24, 05
$0134  NOP

; busy wait
$0136  SBIW R25:R24, 01
$0138  NOP
$013A  BRNE $0136

$013C  RET
```

调用处的代码为：
```asm
; delayMicroseconds(1)。如果参数是变量，会编译成LDS
LDI R24, $01
LDI R25, $00
CALL $011C
```
