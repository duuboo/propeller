# propeller

```spin
CON
  _CLKMODE = xtal1 + pll16x
  _XINFREQ = 6_000_000

  MY_LED_PIN = 0 ' Use pin A0
  SECOND_PIN = 1 ' Use pin A1

VAR
  long stack[128] ' Allocate 128 longs of stack space for our new COG

PUB main
  cognew(blink_second_pin, @stack) ' Start a new COG
    
  DIRA[MY_LED_PIN] := 1 ' Set the LED pin to an output

  repeat ' Repeat forever
    OUTA[MY_LED_PIN] := 1   ' Turn the LED on
    waitcnt(cnt + clkfreq)  ' Wait 1 second 
                            ' cnt is the clock tick counter, 
                            ' clkfreq is the number of clock ticks in a second
    OUTA[MY_LED_PIN] := 0   ' Turn the LED MY_LED_PIN
    waitcnt(cnt + clkfreq)  ' Wait 1 second

PUB blink_second_pin
  DIRA[SECOND_PIN] := 1
  
  repeat
    OUTA[SECOND_PIN] := 1    ' Turn second LED on
    waitcnt(cnt + clkfreq/2) ' Wait half a second
    OUTA[SECOND_PIN] := 0    ' Turn second LED off
    waitcnt(cnt + clkfreq/2) ' Wait half a second
```
