# propeller

```spin
{{
      IR_Remote_NewCog.spin
      Tom Doyle
      2 March 2007

      Panasonic IR Receiver - Parallax #350-00014

      Receive and display codes sent from a Sony TV remote control.
      See "Infrared Decoding and Detection appnote" and "IR Remote for the Boe-Bot Book v1.1"
      on Parallax website for additional info on TV remotes.

      The procedure uses counter A to measure the pulse width of the signals received
      by the Panasonic IR Receiver. The procedure waits for a start pulse and then decodes the
      next 12 bits. The entire 12 bit result is returned. The lower 7 bits contain the actual
      key code. The upper 5 bits contain the device information (TV, VCR etc.) and are masked off
      for the display.

      Most TV Remotes send the code over and over again as long as the key is pressed.
      This allows auto repeat for TV operations like increasing volume. The volume continues to
      increase as long as you hold the 'volume up' key down. Even if the key is pressed for a
      very short time there is often more than one code sent. The best way to eliminate the
      auto key repeat is to look for an idle gap in the IR receiver output. There is a period of
      idle time (20-30 ms) between packets. The getSonyCode procedure will wait for an idle period
      controlled by the gapMin constant. This value can be adjusted to eliminate auto repeat
      while maintaining a fast response to a new keypress. If auto repeat is desired the indicated
      section of code at the start of the getSonyCode procedure can be commented out.

      The procedure sets a tolerance for the width of the start bit and the logic level 1 bit to
      allow for variation in the pulse widths sent out by different remotes. It is assumed that a
      bit is 0 if it is not a 1.

      The procedure to read the keycode ( getSonyCode ) is run in a separate cog. This allows
      the main program loop to continue without waiting for a key to be pressed. The getSonyCode
      procedure writes the NoNewCode value (255) into the keycode variable in main memory to
      indicate that no new keycode is available. When a keycode is received it writes the keycode
      into the main memory variable and terminates. With only 8 cogs available it seems to be a
      good idea to free up cogs rather than let them run forever. The main program can fire off
      the procedure if and when it is interested in a new keycode.
        
}}


CON

  _CLKMODE = XTAL1 + PLL16X        ' 80 Mhz clock
  _XINFREQ = 5_000_000

  NoNewCode    =  255               ' indicates no new keycode received

  gapMin       =   2000             ' minimum idle gap - adjust to eliminate auto repeat
  startBitMin  =   2000             ' minimum length of start bit in us (2400 us reference)
  startBitMax  =   2800             ' maximum length of start bit in us (2400 us reference)
  oneBitMin    =   1000             ' minimum length of 1 (1200 us reference)
  oneBitMax    =   1400             ' maximum length of 1 (1200 us reference)

  ' Sony TV remote key codes
  ' these work for the remotes I tested however your mileage may vary
  
  one   =  0
  two   =  1
  three =  2
  four  =  3
  five  =  4
  six   =  5
  seven =  6
  eight =  7
  nine  =  8
  zero  =  9

  chUp  = 16
  chDn  = 17
  volUp = 18
  volDn = 19
  mute  = 20
  power = 21
  last  = 59


VAR

  byte  cog
  long  Stack[20]  


PUB Start(Pin, addrMainCode) : result
{{
   Pin - propeller pin connected to IR receiver
   addrMainCode - address of keycode variable in main memory
}}

    stop
    byte[addrMainCode] := NoNewCode
    cog := cognew(getSonycode(Pin, addrMainCode), @Stack) + 1
    result := cog


PUB Stop
{{
   stop cog if in use
}}

    if cog
      cogstop(cog~ -1)

    
PUB getSonyCode(pin, addrMainCode) | irCode, index, pulseWidth, lockID

{{
   Decode the Sony TV Remote key code from pulses received by the IR receiver
}}

   ' wait for idle period (ir receiver output = 1 for gapMin)
   ' comment out "auto repeat" code if auto key repeat is desired
   
   ' start of "auto repeat" code section
   dira[pin]~
   index := 0
   repeat
     if ina[Pin] == 1
       index++
     else
       index := 0
   while index < gapMin
   ' end of "auto repeat" code section

   frqa := 1
   ctra := 0
   dira[pin]~
   
   ' wait for a start pulse ( width > startBitMin and < startBitMax  )
   repeat      
      ctra := (%10101 << 26 ) | (PIN)                      ' accumulate while A = 0  
      waitpne(0 << pin, |< Pin, 0)                         
      phsa:=0                                              ' zero width
      waitpeq(0 << pin, |< Pin, 0)                         ' start counting
      waitpne(0 << pin, |< Pin, 0)                         ' stop counting                                               
      pulseWidth := phsa  / (clkfreq / 1_000_000) + 1    
   while ((pulseWidth < startBitMin) OR (pulseWidth > startBitMax))

   ' read in next 12 bits
   index := 0
   irCode := 0
   repeat
      ctra := (%10101 << 26 ) | (PIN)                      ' accumulate while A = 0  
      waitpne(0 << pin, |< Pin, 0)                         
      phsa:=0                                              ' zero width
      waitpeq(0 << pin, |< Pin, 0)                         ' start counting
      waitpne(0 << pin, |< Pin, 0)                         ' stop counting                                               
      pulseWidth := phsa  / (clkfreq / 1_000_000) + 1
      
    if (pulseWidth > oneBitMin) AND (pulseWidth < oneBitMax)
       irCode := irCode + (1 << index)
    index++
   while index < 11

   irCode := irCode & $7f                                   ' mask off upper 5 bits

   byte[addrMainCode] := irCode
 
 
 
 
 '' ********************************
'' *  Parallax Serial LCD Driver  *
'' *  (C) 2006 Parallax, Inc.     *
'' ********************************
''
'' Parallax Serial LCD Switch Settings
''
''   ┌─────────┐     ┌─────────┐     ┌─────────┐
''   │   O N   │     │   O N   │     │   O N   │
''   │ ┌──┬──┐ │     │ ┌──┬──┐ │     │ ┌──┬──┐ │
''   │ │[]│  │ │     │ │  │[]│ │     │ │[]│[]│ │
''   │ │  │  │ │     │ │  │  │ │     │ │  │  │ │
''   │ │  │[]│ │     │ │[]│  │ │     │ │  │  │ │
''   │ └──┴──┘ │     │ └──┴──┘ │     │ └──┴──┘ │
''   │  1   2  │     │  1   2  │     │  1   2  │
''   └─────────┘     └─────────┘     └─────────┘
''      2400            9600            19200


CON

  LcdBkSpc      = $08                                   ' move cursor left
  LcdRt         = $09                                   ' move cursor right
  LcdLF         = $0A                                   ' move cursor down 1 line
  LcdCls        = $0C                                   ' clear LCD (follow with 5 ms delay)
  LcdCR         = $0D                                   ' move pos 0 of next line
  LcdBLon       = $11                                   ' backlight on
  LcdBLoff      = $12                                   ' backlight off
  LcdOff        = $15                                   ' LCD off
  LcdOn1        = $16                                   ' LCD on; cursor off, blink off
  LcdOn2        = $17                                   ' LCD on; cursor off, blink on
  LcdOn3        = $18                                   ' LCD on; cursor on, blink off
  LcdOn4        = $19                                   ' LCD on; cursor on, blink on
  LcdLine0      = $80                                   ' move to line 1, column 0
  LcdLine1      = $94                                   ' move to line 2, column 0
  LcdLine2      = $A8                                   ' move to line 3, column 0
  LcdLine3      = $BC                                   ' move to line 4, column 0

  #$F8, LcdCC0, LcdCC1, LcdCC2, LcdCC3, LcdCC4, LcdCC5, LcdCC6, LcdCC7 


VAR

  word  tx, bitTime, lcdLines, started 


PUB start(pin, baud, lines)

'' Qualifies pin, baud, and lines input
'' -- makes tx pin an output and sets up other values if valid

  started~ 
  if (pin => 0) and (pin < 28)                          ' qualify tx pin
    if lookdown(baud : 2400, 9600, 19200)               ' qualify baud rate setting
      if (lines == 2) or (lines == 4)                   ' qualify lcd size
        tx := pin
        outa[tx]~~
        dira[tx]~~
        bitTime := clkfreq / baud
        lcdLines := lines
        started~~ 

  return started


PUB stop

'' Makes serial pin an input

  if started
    dira[tx]~                                           ' make pin an input
    started~                                            ' set to false


PUB putc(txbyte) | time

'' Transmit byte

  if started
    txbyte := (txbyte | $100) << 2                      ' add stop bit 
    time := cnt                                         ' sync
    repeat 10                                           ' start + eight data bits + stop
      waitcnt(time += bitTime)                          ' wait bit time
      outa[tx] := (txbyte >>= 1) & 1                    ' output bit (true mode)
    

PUB str(str_addr)

'' Transmit string

  if started
    repeat strsize(str_addr)                            ' for each character in string
      putc(byte[str_addr++])                            '   write the character


PUB cls

'' Clears LCD and moves cursor to home (0, 0) position

  if started
    putc(LcdCls)
    waitcnt(clkfreq / 200 + cnt)                        ' 5 ms delay 


PUB home

'' Moves cursor to 0, 0

  if started
    putc(LcdLine0)
  
  
PUB clrln(line)

'' Clears line

  if started
    line := 0 #> line <# 3                              ' qualify line input
    putc(LinePos[line])                                 ' move to that line
    if lcdLines == 2                                    ' check lcd size
      repeat 16
        putc(32)                                        ' clear line with spaces
    else
      repeat 20
        putc(32)
    putc(LinePos[line])                                 ' return to start of line  


PUB newline

'' Moves cursor to next line, column 0; will wrap from line 3 to line 0

  putc(LcdCR)
  

PUB gotoxy(col, line) | pos

'' Moves cursor to col/line

  if started
    line := 0 #> line <# 3                              ' qualify line
    if lcdLines == 2
      col := 0 #> col <# 15                             ' qualify column
    else
      col := 0 #> col <# 19
    putc(LinePos[line] + col)                           ' move to target position


PUB cursor(crsr_type)

'' Selects cursor type
''   0 : cursor off, blink off  
''   1 : cursor off, blink on   
''   2 : cursor on, blink off  
''   3 : cursor on, blink on

  case crsr_type
    0..3  : putc(DispMode[crsr_type])                   ' get mode from table
    other : putc(LcdOn3)                                ' use serial lcd power-up default

      
PUB custom(char, chr_addr)

'' Installs custom character map

  if started
    if (char => 0) and (char =< 7)                      ' make sure char in range
      putc(LcdCC0 + char)                               ' write character code
      repeat 8
        putc(byte[chr_addr++])                          ' write character data


PUB backlight(status)

'' Enable (1) or disable (0) LCD backlight

  if started
    if (status & 1)
      putc(LcdBLon)
    else
      putc(LcdBLoff)
  

DAT

  LinePos     byte      LcdLine0, LcdLine1, LcdLine2, LcdLine3
  DispMode    byte      LcdOn1, LcdOn2, LcdOn3, LcdOn4
  

  
 
 
 
 
 
 
 
'' *****************************
'' *  Simple_Numbers           *
'' *  (c) 2006 Parallax, Inc.  *
'' *****************************
''
'' Provides simple numeric conversion methods; all methods return a pointer to
'' a string.


CON

  str_max = 64                                          ' 63 chars + zero terminator

  
VAR

  long  idx                                             ' pointer into string
  byte  nstr[str_max]                                   ' string for numeric data


PUB dec(value) | div, zpad

'' Convert signed decimal to string

  clrstr(@nstr, str_max)                                ' clear output string  
  return decstr(value)                                  ' return pointer to numeric string
    

PUB decf(value, width) | tval, field

'' Put signed decimal value in fixed-width (space padded) string

  clrstr(@nstr, str_max)
  width := 1 #> width <# 63                             ' qualify field width 

  width #>= 1                                           ' must be at least 1
  tval := ||value                                       ' work with absolute
  field~                                                ' clear field

  repeat while tval > 0                                 ' count number of digits
    field++
    tval /= 10

  field #>= 1                                           ' min field width is 1
  if value < 0                                          ' if value is negative
    field++                                             '   bump field for neg sign indicator
  
  if field < width                                      ' need for space pad?
    repeat (width - field)                              ' yes
      nstr[idx++] := " "                                '   print space(s)

  return decstr(value)


PRI decstr(value) | div, zpad   

'' Converts value to to signed decimal string
'' -- does not clear output string; caller must do that  

  if (value < 0)                                        ' negative value? 
    -value                                              '   yes, make positive
    nstr[idx++] := "-"                                  '   and print sign indicator

  div := 1_000_000_000                                  ' initialize divisor
  zpad~                                                 ' clear zero-pad flag

  repeat 10
    if (value => div)                                   ' printable character?
      nstr[idx++] := (value / div + "0")                '   yes, print ASCII digit
      value //= div                                     '   update value
      zpad~~                                            '   set zflag
    elseif zpad or (div == 1)                           ' printing or last column?
      nstr[idx++] := "0"
    div /= 10 

  return @nstr


PUB hex(value, digits)

'' Print a hexadecimal number

  clrstr(@nstr, str_max) 
  digits := 1 #> digits <# 8                            ' qualify digits
  value <<= (8 - digits) << 2                           ' prep MS digit
  repeat digits
    nstr[idx++] := lookupz((value <-= 4) & %1111 : "0".."9", "A".."F")

  return @nstr


PUB ihex(value, digits)

'' Print and indicated hexadecimal number

  clrstr(@nstr, str_max)
  nstr[idx++] := "$"
  digits := 1 #> digits <# 8
  value <<= (8 - digits) << 2
  repeat digits
    nstr[idx++] := lookupz((value <-= 4) & %1111 : "0".."9", "A".."F")

  return @nstr    
    

PUB bin(value, digits)

'' Print a binary number

  clrstr(@nstr, str_max)
  digits := 1 #> digits <# 32                           ' qualify digits 
  value <<= 32 - digits                                 ' prep MSB
  repeat digits
    nstr[idx++] := (value <-= 1) & 1 + "0"              ' move digits (ASCII) to string

  return @nstr        


PUB ibin(value, digits)

'' Print an indicated binary number

  clrstr(@nstr, str_max)
  nstr[idx++] := "%"                                    ' preface with binary indicator
  digits := 1 #> digits <# 32 
  value <<= 32 - digits
  repeat digits
    nstr[idx++] := (value <-= 1) & 1 + "0"

  return @nstr      


PRI clrstr(str_addr, size)

  bytefill(str_addr, 0, size)                           ' clear string to zeros
  idx~                                                  ' reset index
  
  
  
  
  
  
  
{{

      Sony_Send.spin
      Tom Doyle
      9 March 2007

      Use counter A to send a Sony remote keycode to an IR led.
      The counter is used in NCO mode and is turned on and off
      by changing ctra. 

      The Sony standard is used
      
         12 data bits
         25 mS gap between messages
         2.4 mS start bit
         1.2 mS data bit 1
         0.6 mS data bit 0
         0.6 mS gap between bits
     
}}
      

CON

  _CLKMODE = XTAL1 + PLL16X        ' 80 Mhz clock
  _XINFREQ = 5_000_000

  PNA4602M  =  38_000              ' IR Receiver Carrier Frequency


OBJ

  time      : "Timing"


PUB SendSonyCode(Pin, Code) | index

{{
   send Sony TV remote code
   Pin - pin connected to IR LED
   Code - decimal code for Sony tv remote
}}

  SynthFreq(Pin, PNA4602M) ' initialize counter A
  
  ' pause between messages
  ctra := 0
  time.pause1ms(25)

  ' start bit 2.4 mS
  ctra := %00100 << 26 | Pin
  time.pause10us(240)

  index := 0
  repeat
    if ((Code >> index) & 1)  == 1
      SendSonyBit(Pin, 1)
    else
      SendSonyBit(Pin, 0)
    index++
  while index < 12

  
PRI SendSonyBit(Pin, Bit) 

{{
   send a single bit to the IR led
   Pin - pin connected to IR led
   Bit - 1 or 0
}}

  ' gap between bits 0.6 mS
  ctra := 0
  time.pause10us(60)

  if Bit == 1                        ' 1.2 mS
    ctra := %00100 << 26 | Pin
    time.pause10us(120)
  else                               ' 0.6 mS
    ctra := %00100 << 26 | Pin
    time.pause10us(60)

  ctra := 0
        

PRI SynthFreq(Pin, Freq) | a, b, f

{{
   use counter A to generate a square wave
   Pin - pin connected to IR led
   Freq - frequency for counter (Freq < 500 kHz)
}}

  dira[Pin]~~                          ' output

  ctra := 0                            ' counter off till ready
   
  a := Freq <<= 1                      ' perform long division of a/b
  b := clkfreq
 
  repeat 32                            
    f <<= 1
    if a => b
      a -= b
      f++           
   a <<= 1
 
  ctra := %00100 << 26 | Pin            ' CTRA - NCO mode and PinA
  frqa := f                             'set FRQA                   
  dira[Pin]~~                           'make counter pin A output
 

 
 
 
 
 
 

{{

    Sony_Send_Test.spin
    Tom Doyle
    9 March 2007

    Counter A is used to send Sony TV remote codes to an IR led
    (Parallax #350-00017).

    The test program uses a Panasonic IR Receiver (Parallax #350-00014)
    to receive and decode the signals. Due to the power of multiple cogs
    in the Propeller the receive object runs in its own cog  waiting for
    a code to be received. When a code is received it is written into the
    IRcode variable and processed by the main program. The code is read
    by the receive object in a cog as it is transmitted by another cog.
    This works fine at the relatively low data rates used by Sony TV remote.

    A Parallax LCD display is used to display the received codes as the
    Propeller talks to itself.

    See Sony_Send.spin and IR_Remote.spin for more information. 
}}
      

CON

  _CLKMODE = XTAL1 + PLL16X        ' 80 Mhz clock
  _XINFREQ = 5_000_000

  IRdetPin  =       23             ' IR Receiver - Propeller Pin
  IRledPin  =       22             ' IR Led - Propeller Pin


OBJ

  irTx      : "Sony_Send"
  irRx      : "IR_Remote"
  lcd       : "Serial_Lcd"
  num       : "simple_numbers"
  time      : "Timing"


VAR

  byte IRcode                           ' keycode from IR Receiver

    
PUB Init |  freq, index, rxCog, txCog, lcode

  if lcd.start(0, 9600, 4)                               
    lcd.cls
    lcd.putc(lcd#LcdOn1)                                                                            ' setup screen
    lcd.backlight(1)                       
    lcd.str(string("Sony Send"))

  
  ' start IR receiver using a new cog
  rxCog := irRx.Start(IRdetPin, @IRcode)  '  pin connected to the IR receiver, address of variable
  time.pause1ms(60)                       ' wait for IR Receiver to start
  
  index := 0
  repeat

    irTx.SendSonyCode(IRledPin, index)  ' send a code
        
    If IRcode <> irRx#NoNewCode         ' check for a new code 
        
      lcode := IRcode
      irRx.Start(IRdetPin, @IRcode)     ' set up for next code
                       
      lcd.gotoxy(0,1)
      lcd.str(num.bin(lcode, 7))
      lcd.str(string("  "))
      lcd.str(num.dec(lcode))
      lcd.str(string("  "))

      case lcode
        irRx#one   :  lcd.str(string("<1>   "))
        irRx#two   :  lcd.str(string("<2>   "))
        irRx#three :  lcd.str(string("<3>   "))
        irRx#four  :  lcd.str(string("<4>   "))
        irRx#five  :  lcd.str(string("<5>   "))
        irRx#six   :  lcd.str(string("<6>   "))
        irRx#seven :  lcd.str(string("<7>   "))
        irRx#eight :  lcd.str(string("<8>   "))
        irRx#nine  :  lcd.str(string("<9>   "))
        irRx#zero  :  lcd.str(string("<0>   "))
        irRx#chUp  :  lcd.str(string("chUp "))
        irRx#chDn  :  lcd.str(string("chDn "))
        irRx#volUp :  lcd.str(string("volUp"))
        irRx#volDn :  lcd.str(string("volDn"))
        irRx#mute  :  lcd.str(string("mute "))
        irRx#power :  lcd.str(string("power"))
        irRx#last  :  lcd.str(string("last "))
       other       :  lcd.str(string("      "))
          
   waitcnt((clkfreq / 2000) * 1500 + cnt)
   
   index := index + 1
   
   if index > 9
      index := 0
        

 
'' *****************************
'' *  Timing                   *
'' *  (C) 2006 Parallax, Inc.  *
'' *****************************
''
'' This object provides time delay and time synchronization functions.


CON
  
  _10us = 1_000_000 /        10                         ' Divisor for 10 us
  _1ms  = 1_000_000 /     1_000                         ' Divisor for 1 ms
  _1s   = 1_000_000 / 1_000_000                         ' Divisor for 1 s


VAR

  long delay
  long syncpoint
  long clkcycles


PUB pause10us(period)

'' Pause execution for period (in units of 10 us)
 
  clkcycles := ((clkfreq / _10us * period) - 4296) #> 381    ' Calculate 10 us time unit
  waitcnt(clkcycles + cnt)                                   ' Wait for designated time


PUB pause1ms(period)

'' Pause execution for period (in units of 1 ms).

  clkcycles := ((clkfreq / _1ms * period) - 4296) #> 381     ' Calculate 1 ms time unit
  waitcnt(clkcycles + cnt)                                   ' Wait for designated time
  

PUB pause1s(period)

'' Pause execution for period (in units of 1 sec).

  clkcycles := ((clkfreq / _1s * period) - 4296) #> 381      ' Calculate 1 s time unit
  waitcnt(clkcycles + cnt)                                   ' Wait for designated time

  
PUB marksync10us(period)

  delay := (clkfreq / _10us * period) #> 381                 ' Calculate 10 us time unit 
  syncpoint := cnt

  
PUB waitsync
 
  waitcnt(syncpoint += delay)

```
