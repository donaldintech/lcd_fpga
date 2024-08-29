# Interfacing a 16x2 LCD with an FPGA board.
## Background Info ##
-Unlike the case with an Arduino, an FPGA presents a more challenging task when interfacing with an LCD.

First we are aware how for an FPGA we have to tell the hardware how it should implement our logic through an Hardware Description Language(HDL), and how higher level & simple libraries make working with a micro-controller easier.(for instance ,look at the code below for an arduino used to implement 'hello world')

![image](https://github.com/user-attachments/assets/442612fa-0654-4c4e-96de-bdf6f4b2e684)

It is also important to note the processes involved in an fpga design where a clock takes care of timing requirements and delays</br>
Our design below uses a collection of blocks including; D-Flip Flops for memory and counters, ROM and a Multiplexer, with a FSM used to switch between the 'idle', 'init_cmd', 'display_char' and 'done' states.

-To understand more about the building blocks , read this article [lcd with fpga](https://www.allaboutcircuits.com/technical-articles/how-to-interface-mojo-v3-fpga-board-16x2-lcd-module-FPGA-tutorial/)

## Requirements & Installation ##

1.	Quartus Web II software (This project uses ver. 13.0.1)
2.	Cyclone II FPGA Development kit (EP2C20F484C7N)
3.	10k Potentiometer-(Or 2 Resistors with the Vo pin of the LCD connected between them)
4.	Connector wires-male to female
5.	LCD -with a HD44780 driver
   
-Our LCD pins are explained below;

![image](https://github.com/user-attachments/assets/7b32fcdc-03e3-4b7f-80e3-19369c920a18)

**RS (Register Select)**

-RS is a command pin for LCD. The LCD command and RW operations are determined by RS pin. The LCD contains two registers; the data and command register.

When command writes on the LCD, the data register is used. When the data either reads or writes on the LCD, the command register is used. The selections of register are determined by the logic status of RS.

-If the logic state of RS is ‘1’, the data register is selected. If the logic state of RS is ‘0’, the command register is selected.

**Enable (E)**

-When this pin is set to LOW, the LCD ignores activity on the R/W, RS, and data bus lines; when it is set to HIGH, the LCD processes the incoming data. D0-D7 (Data Bus) pins carry the 8 bit data we send to the display.

**Data pins (D0-D7)**

-The D0-D7 is data I/O pins. The LCD accepts the 8 bit data as a parallel form. The format of data stream is that the first bit must be LSB bit.

**LED backlight**

-The pin numbers 15 & 16 are allocated for LCD backlight. The supply of LED backlight is +5v & 0v. It makes brightness of LCD display posssible. The LED backlight pins denoted as (Anode (A) or LED+, Cathode (K)) or LED- on the LCD.
-In a Timing Diagram, 

 ![image](https://github.com/user-attachments/assets/aecfc8dd-7c3b-4964-b96c-067bb8815b71)

-Moving on, the installation steps are similar to the tutorial on 4 segment display, which you can find in the following repo.
 [fpga segment display](https://github.com/donaldintech/fpga_segment_display)



## Programming in VHDL ##

Breaking down our program into sections provide an easier way to understand what each section does and represent.

**•	Libraries**

Define the standard libraries which facilitate design reuse and standard data types for model exchange, reuse, and synthesis. 
```
library IEEE;
use IEEE.STD_LOGIC_1164.ALL;
use IEEE.STD_LOGIC_ARITH.ALL;
use IEEE.STD_LOGIC_UNSIGNED.ALL;
```
 
**•	Entity**

Define an entity which contains all the input & output ports to be used in the program
```
entity test_lcd is
    port (
        clk    : in std_logic;                          -- clock input
        lcd_rw : out std_logic;                         -- read & write control
        lcd_e  : out std_logic;                         -- enable control
        lcd_rs : out std_logic;                         -- data or command control
        data   : out std_logic_vector(7 downto 0)       -- data line
    );
end test_lcd;
```
 
**•	Architecture**

The main part of our program, contains two sections: 

**1.Signal declaration** 

-This is where you declare all the signals and variables to be used in the processes that follow

```
    constant N: integer := 15;  -- Number of commands and characters (Corrected to 16)
    type arr is array (1 to N) of std_logic_vector(7 downto 0);
    constant datas : arr := (
        X"38", X"0C", X"06", X"01",  -- Initialization commands
        X"48", X"45", X"4C", X"4C", X"4F", -- "HELLO"
        X"20", -- Space
        X"57", X"4F", X"52", X"4C", X"44"  -- "WORLD"
        -- Removed the extra space to match the count
    );

    -- FSM state declaration
    type state_type is (idle, init_cmd, display_char, done);
    signal state_reg, state_next : state_type := idle;
    
    -- Additional signals
    signal display_done : std_logic := '0';  -- Control signal to stop after one display
    signal i : integer := 0;
    signal j : integer := 1;
```
-NB: You can represent your desired characters , refer to these website to check correct codes.
[ascii-code](https://www.ascii-code.com/)

 **2.Process definition** 


-Now to our  main process, where counters i and j are introduced to implement delays.
```
  process(state_reg, i, j)
    begin
        case state_reg is
            when idle =>
                -- Initial state: Transition to init_cmd state
                state_next <= init_cmd;
                lcd_e <= '0';  -- Disable LCD initially

            when init_cmd =>
                -- Send initialization commands
                if i <= 1000000 then
                    i <= i + 1;
                    lcd_e <= '1';
                    data <= datas(j)(7 downto 0);
                elsif i > 1000000 and i < 2000000 then
                    i <= i + 1;
                    lcd_e <= '0';
                elsif i = 2000000 then
                    j <= j + 1;
                    i <= 0;
                end if;

                -- Transition to display_char state after initialization
                if j = 5 then
                    state_next <= display_char;
                else
                    state_next <= init_cmd;
                end if;

                lcd_rs <= '0';  -- Command signal during initialization

            when display_char =>
                -- Display characters
                if i <= 1000000 then
                    i <= i + 1;
                    lcd_e <= '1';
                    data <= datas(j)(7 downto 0);
                elsif i > 1000000 and i < 2000000 then
                    i <= i + 1;
                    lcd_e <= '0';
                elsif i = 2000000 then
                    j <= j + 1;
                    i <= 0;
                end if;

                -- Data signal during display
                lcd_rs <= '1';  -- Data signal for character display

                -- Check if all characters have been displayed
                if j = N + 1 then
                    state_next <= done;
                else
                    state_next <= display_char;
                end if;

            when done =>
                -- Stop the process after displaying the entire string
                lcd_e <= '0';  -- Disable LCD enable after displaying the message
                display_done <= '1';
                state_next <= done;  -- Remain in done state

            when others =>
                state_next <= idle;
        end case;
    end process;
```
-To understand the logic above; we need to keep in mind our 4 states & what happens in each.

**1.Idle State:**
-The FSM starts in the idle state, where no active operation occurs.

-The lcd_e signal is set to '0' to disable the LCD enable signal and the FSM immediately transitions to the init_cmd state to begin the initialization process by setting state_next <= init_cmd;.

**2.Initialization State:**
-This state handles sending the initialization commands to the LCD.

-If the counter i is less than or equal to 1,000,000(which is a delay of 20mS),the lcd_e signal is set to '1', which enables the LCD. The data bus data is loaded with the current command from the datas array (datas(j)(7 downto 0)).
The Counter i is incremented to track the duration of the enable signal.

-If i is between 1,000,001 and 2,000,000,the lcd_e signal is set to '0' to disable the LCD enable signal.

-If i equals 2,000,000,the counter i is reset to 0, and the index j is incremented to point to the next command in the datas array.

When all initialization commands are sent (when j = 5), the FSM transitions to the display_char state,otherwise, it remains in the init_cmd state to send the next command.

**3. Display Character State:**
-This state handles sending the characters for "HELLO WORLD" to the LCD.

-If the counter i is less than or equal to 1,000,000, the lcd_e signal is set to '1', which enables the LCD. The data bus data is loaded with the current character from the datas array (datas(j)(7 downto 0)).
The Counter i is incremented to track the duration of the enable signal.

-If i is between 1,000,001 and 2,000,000,the lcd_e signal is set to '0' to disable the LCD enable signal.

-If i equals 2,000,000,the counter i is reset to 0, and the index j is incremented to point to the next character in the datas array.

-lcd_rs is set to '1' to indicate that the data being sent is a character (not a command).

If all characters have been sent (when j = N + 1), the FSM transitions to the done state, otherwise, it remains in the display_char state to send the next character.

**4. Done State:**
-This state is reached once the entire "HELLO WORLD" message has been displayed.

-The lcd_e signal is set to '0' to disable the LCD enable signal and the display_done signal is set to '1', indicating that the process is complete.

-The block diagram of the states is shown below; (in progress...)



<br/><br/>
-FIND THE WHOLE CODE BELOW: 

```
library IEEE;
use IEEE.STD_LOGIC_1164.ALL;
use IEEE.STD_LOGIC_ARITH.ALL;
use IEEE.STD_LOGIC_UNSIGNED.ALL;

entity test_lcd is
    port (
        clk    : in std_logic;                          -- clock input
        lcd_rw : out std_logic;                         -- read & write control
        lcd_e  : out std_logic;                         -- enable control
        lcd_rs : out std_logic;                         -- data or command control
        data   : out std_logic_vector(7 downto 0)       -- data line
    );
end test_lcd;

architecture RTL of test_lcd is
    constant N: integer := 15;  -- Number of commands and characters (Corrected to 16)
    type arr is array (1 to N) of std_logic_vector(7 downto 0);
    constant datas : arr := (
        X"38", X"0C", X"06", X"01",  -- Initialization commands
        X"48", X"45", X"4C", X"4C", X"4F", -- "HELLO"
        X"20", -- Space
        X"57", X"4F", X"52", X"4C", X"44"  -- "WORLD"
        -- Removed the extra space to match the count
    );

    -- FSM state declaration
    type state_type is (idle, init_cmd, display_char, done);
    signal state_reg, state_next : state_type := idle;
    
    -- Additional signals
    signal display_done : std_logic := '0';  -- Control signal to stop after one display
    signal i : integer := 0;
    signal j : integer := 1;

begin
    lcd_rw <= '0';  -- LCD write mode

    -- FSM State Register Process
    process(clk)
    begin
        if rising_edge(clk) then
            state_reg <= state_next;
        end if;
    end process;

    -- FSM Next State Logic and Output Logic
    process(state_reg, i, j)
    begin
        case state_reg is
            when idle =>
                -- Initial state: Transition to init_cmd state
                state_next <= init_cmd;
                lcd_e <= '0';  -- Disable LCD initially

            when init_cmd =>
                -- Send initialization commands
                if i <= 1000000 then
                    i <= i + 1;
                    lcd_e <= '1';
                    data <= datas(j)(7 downto 0);
                elsif i > 1000000 and i < 2000000 then
                    i <= i + 1;
                    lcd_e <= '0';
                elsif i = 2000000 then
                    j <= j + 1;
                    i <= 0;
                end if;

                -- Transition to display_char state after initialization
                if j = 5 then
                    state_next <= display_char;
                else
                    state_next <= init_cmd;
                end if;

                lcd_rs <= '0';  -- Command signal during initialization

            when display_char =>
                -- Display characters
                if i <= 1000000 then
                    i <= i + 1;
                    lcd_e <= '1';
                    data <= datas(j)(7 downto 0);
                elsif i > 1000000 and i < 2000000 then
                    i <= i + 1;
                    lcd_e <= '0';
                elsif i = 2000000 then
                    j <= j + 1;
                    i <= 0;
                end if;

                -- Data signal during display
                lcd_rs <= '1';  -- Data signal for character display

                -- Check if all characters have been displayed
                if j = N + 1 then
                    state_next <= done;
                else
                    state_next <= display_char;
                end if;

            when done =>
                -- Stop the process after displaying the entire string
                lcd_e <= '0';  -- Disable LCD enable after displaying the message
                display_done <= '1';
                state_next <= done;  -- Remain in done state

            when others =>
                state_next <= idle;
        end case;
    end process;

end RTL;


```  


## Preparing the Code ##

-Again the steps for preparing your code can be found in a similar project in the repo: 
[fpga segment display](https://github.com/donaldintech/fpga_segment_display) </br>
-However, for the pinout, refer to the manual or datasheet (or simply use the pins I used ) where you can access the GPIO pins and the VCC & ground pins 

![image](https://github.com/user-attachments/assets/a6a6b168-7804-4000-9ec5-d5a588a78c1d)

-Once you're satisfied with your code & simulation , go ahead to program your code and emjoy the outcome.


