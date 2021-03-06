# Introducing my latest rathole: PiTerm1986
**GOAL:** Convert 80s style user interface ( 8031 based ADP Product: chicklet keyboard and 2x20 lcd ) to pi zero based terminal/console using i2c based io expanders.

![](images/IMG_3780.jpg)
![](images/IMG_3784.jpg)
![](images/IMG_3783.jpg)
![](images/IMG_3782.jpg)
![](images/IMG_3781.jpg)

Looking under the keyboard membrane and tracing the connectors we can get the following key map under a 4x12 array (16 pins).

```
   0123456789A(10)B(11)
   --------------------
0) *# ZXCVBNM/    <STOP>
1) -LASDFGHJK<N/C><ENTER>
2) 0912345678<BS> <START>
3) POQWERTYUI<CAN><DIAL>
```
<table>
<tr><td>*</td><td>#</td><td> </td><td>Z</td><td>X</td><td>C</td><td>V</td><td>B</td><td>N</td><td>M</td><td>/</td><td>STOP</td></tr>
<tr>
<td>&#45;</td><td>L</td><td>A</td><td>S</td><td>D</td><td>F</td><td>G</td><td>H</td><td>J</td><td>K</td><td>N/C</td><TD>ENTER</TD></tr>
<tr>
<td>0</td><td>9</td><td>1</td><td>2</td><td>3</td><td>4</td><td>5</td><td>6</td><td>7</td><td>8</td><td>&LT;----</td><td>START</td></tr>
<tr>
<td>P</td><td>O</td><td>Q</td><td>W</td><td>E</td><td>R</td><td>T</td><td>Y</td><td>U</td><td>I</td><td>CANCEL</td><td>DIAL</td></tr>
</table>
### Keyboard

![](images/IMG_3779.jpg)

For this we look at the AW9523 GPIO expander with 16 pins of io.

### LCD/VFD 

Since the LCD is a 5v circuit requiring either 8 or 12 pins we look at the Microchip MCP23017 on the Adafruit GPIO Expander Bonnet which we can wire either using both ports or one port in nibble mode. 

![](images/HD44780.jpg)

### Attaching Both Boards to the Raspberry pi.

![](images/piterm1986pahts.jpg)
![](images/IMG_3789.jpg)

```
root@somepi1:/home/feurig# i2cdetect -y 1
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
00:          -- -- -- -- -- -- -- -- -- -- -- -- -- 
10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
20: 20 -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
30: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
40: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
50: -- -- -- -- -- -- -- -- 58 -- -- -- -- -- -- -- 
60: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
70: -- -- -- -- -- -- -- --                         
```

## Assembly and layout.
After putting the board in the box I realized that the grey paint was conductive and that I almost let the smoke out of the pi zero. (you could smell it). So I went out in the garage and found a piece of plastic from a previous project. And then I was like OH SHIT I have Noritake 2x24 VFD thats a close fit..


![](images/IMG_3826.jpg)

### So there's a circuitpython library for an mcp23017 connected to an lcd.

```
root@somepi1:/home/feurig# pip3 install adafruit-circuitpython-charlcd
...
```

![](images/raspberry_pi_pilcdplate.png)

I am not thrilled about the way its wired but, whatever, because wiring it up according to the above schematic should let us write away.
It't a bit jenky...
I would like to be able to easily remap the pins without rewriting the library but I am not there.

But it works.

![](images/IMG_3819.jpg)

Now on to the keyboard.
If it werent for the fact that there was a readymade lcd library for the mcp23017 I would have swapped the io expanders. The aw9523 has no internal pullup or pulldown resistors. Fortunately I have piles of 10k resistors and apparently I have time. (Not enough time to make fritzing make part to wire connections look good it seems)

![](images/adpkbdaw9523_schem.jpg)

Here is a simple circtuitpython program which presents a prompt and then puts characters scanned by the keyboard onto the display.

```
import time
import board
import busio
import adafruit_character_lcd.character_lcd_rgb_i2c as character_lcd
import adafruit_aw9523

keymap=[]
keymap.append('*# ZXCVBNM/\001')
keymap.append('-LASDFGHJK\r\n')
keymap.append('0912345678\010\002')
keymap.append('POQWERTYUI\030\004')

lcd_columns = 24
lcd_rows = 2
i2c = busio.I2C(board.SCL, board.SDA)
lcd = character_lcd.Character_LCD_RGB_I2C(i2c, lcd_columns, lcd_rows)
lcd.message = "PiTerm 1986 v0.0\nOk >"

aw = adafruit_aw9523.AW9523(i2c)
row_pins = [aw.get_pin(8),aw.get_pin(9),aw.get_pin(10),aw.get_pin(11)]
for pin in row_pins:
   pin.switch_to_output(value=True)

col_pins = [aw.get_pin(0),aw.get_pin(15),aw.get_pin(14),aw.get_pin(13),
            aw.get_pin(12),aw.get_pin(7),aw.get_pin(6),aw.get_pin(5),
            aw.get_pin(4),aw.get_pin(3),aw.get_pin(2),aw.get_pin(1)]

for pin in col_pins:
   pin.switch_to_input(value=True)


old_row=0;
for pin in row_pins:
    pin.value=1

switch_state = [[0,0,0,0,0,0,0,0,0,0,0,0],
                [0,0,0,0,0,0,0,0,0,0,0,0],
                [0,0,0,0,0,0,0,0,0,0,0,0],
                [0,0,0,0,0,0,0,0,0,0,0,0]
               ]
while True:
    for r in range(4):
        row_pins[old_row].value=1
        row_pins[r].value=0
        old_row=r
        for c in range(12):
            if (col_pins[c].value==0 and switch_state[r][c]==0):
                switch_state[r][c]=1
                print(keymap[r][c])
                lcd.message+=keymap[r][c]
            if (col_pins[c].value==1 and switch_state[r][c]==1):
                switch_state[r][c]=0
    time.sleep(0.01)  # debounce
```



# References
* [https://learn.adafruit.com/adafruit-aw9523-gpio-expander-and-led-driver](https://learn.adafruit.com/adafruit-aw9523-gpio-expander-and-led-driver)
* [AW9523 Datasheet](https://cdn-shop.adafruit.com/product-files/4886/AW9523+English+Datasheet.pdf)
* [https://learn.adafruit.com/gpio-expander-bonnet/overview](https://learn.adafruit.com/gpio-expander-bonnet/overview)
* [MCP23017 Datasheet](http://ww1.microchip.com/downloads/en/devicedoc/20001952c.pdf)
* [https://protostack.com.au/2010/03/character-lcd-displays-part-1/](https://protostack.com.au/2010/03/character-lcd-displays-part-1/)
* [https://www.electronicsforu.com/technology-trends/learn-electronics/16x2-lcd-pinout-diagram](https://www.electronicsforu.com/technology-trends/learn-electronics/16x2-lcd-pinout-diagram)


