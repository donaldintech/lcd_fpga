# Interfacing a 16x2 LCD with an FPGA board.
## Background Info ##
Unlike the case with an Arduino, an FPGA represents a more challenging task when interfacing with an LCD.
First we are aware how for an FPGA we have to tell the hardware how it should implement our logic through an Hardware Description Language(HDL), and how higher level & simple libraries make working with a micro-controller easier.(look at the code below for an arduino 
used to implement 'hello world')
![image](https://github.com/user-attachments/assets/442612fa-0654-4c4e-96de-bdf6f4b2e684)

It is also important to note the processes involved in an fpga design where a clock takes care of timing requirements and delays
Our design below uses a collection of blocks including; D-Flip Flops for memory and counters, ROM and a Multiplexer, with a FSM used to switch between the 'idle', 'init_cmd', 'display_char' and 'done' states.



## Requirements & Installation ##

1.	Quartus Web II software (This project uses ver. 13.0.1)
2.	Cyclone II FPGA Development kit (EP2C20F484C7N)

-The installation steps are similar to the tutorial on 4 segment display, which you can find in the following repo.
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

 **2.Process definition** 


-Now to our  main process, 
```
 
```
  


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
-Once you're satisfied with your code & simulation , go ahead to program your code and emjoy the outcome.


