# VHDL
MIPS em VHDL com as funções necessárias para executar um BubbleSort. 

library IEEE;        		--- Biblioteca básica 
use IEEE.std_logic_1164.all;	--- Pacote com tipos e funções lógicas básico
use work.cordic_package.all ;

  
entity ControlPath is
    port (
        clk, rst    		: in std_logic;
        data_av, start  	: in std_logic;
	if_else, condicao 	: in std_logic;
        done        		: out std_logic;
	en_Mem			: out std_logic; 
	cmd			: out Command 
	    
    );
        
end ControlPath;

architecture behavioral of ControlPath is
   type State is (S0,S1,S2,S3,S4,S5,S6,S7,S8,S9,S10,S11,S12);
   signal currentState, nextState: State;
begin

   -- State memory
   process(clk, rst)
   begin	
      if rst = '1' then
         currentState <= S0;
      
      elsif rising_edge(clk) then
         currentState <= nextState;	
      end if;
   end process;

-- Next state logic
process (currentState, data_av, start, if_else, condicao)
begin		
   case currentState is
       	when S0 =>
       	if data_av = '1' then		
            	nextState <= S1 ;
        end if;

        when S1 =>	
	if data_av = '1' then	
         	nextState <= S2 ;
	end if;	

        when S2 =>
        if start = '1' then	
         	nextState <= S3 ;
	end if;			
      
	When S3 =>
	if condicao = '1' then
		nextState <= S12 ;
	else 
	    nextState <= S4;
	end if;

	when S4 =>
	if if_else = '1' then
		nextState <= S5 ;
	else 
	    nextState <= S9;
	end if;
	
	when S5 =>
		nextState <= S6 ;

	when S6 =>
		nextState <= S7 ;

	when S7 => 
		nextState <= S8 ;

	when S8 =>
		nextState <= S3 ;
	
	when S9 =>
		nextState <= S10 ;

	when S10 =>
		nextState <= S11;
	
	when S11 =>
		nextState <= S8;

	when others => 	
         	nextState <= S0 ;
   
end case;		
end process;
	
	
		--OUTPUT_1 LOGIC CORDIC original
	cmd.x_new_en <= '1' when currentState = S5 or currentState = S9 else '0';	--Sinal do Reg XNEW e do MUX que controla Se é X ou Y a ser deslocado
	cmd.x_en 	 <= '1' when currentState = S0 or currentState = S8 else '0';	--Sinal do Reg X
	cmd.sinal_desl <= '1' when currentState = S9 or currentState = S6 else '0';	--Sinal do MUX que controla + ou - (x>>i_8bits or Y>>i_8bits)
	cmd.y_new_en   <= '1' when currentState = S10 or currentState = S6 else '0';	----Sinal do Reg YNEW
	cmd.sumAngle_en <= '1' when currentState = S0 or currentState = S7 or currentState = S11  else '0';  ----Sinal do Reg sumAngle
	cmd.mux_sm1   <= "00" when currentState = S5 or currentState = S6 or currentState = S9 or currentState = S10 else --Sinal do MUX de 2 bits conectado ao Bus_A do somador
		     "01" when currentState = S8 else 
		     "10" when currentState = S4 or currentState = S7 or currentState = S11 else
		     "11";
	cmd.sub <= '1' when currentState = S3 or currentState = S4 or currentState = S5 or currentState = S10 or currentState = S11  else '0'; -- Sinal de Carryin


		--OUTPUT_2 LOGIC CORDIC 
	cmd.mux_x   <= '1' when currentState = S0 else '0'; -- Sinal do MUX Que controla o valor da Entrada do Reg X
	cmd.it_en   <= '1' when currentState = S1 else '0'; -- Sinal Do Reg It
	
	cmd.mux_sm2   <= "000" when currentState = S5 or currentState = S9 else  
		     "001" when currentState = S6 or currentState = S10 else
		     "010" when currentState = S4 else
	             "011" when currentState = S7 else		-- Sinal do MUX de 3 bits conectado ao Bus_B do Somador
	             "100" when currentState = S11 else
		     "101" when currentState = S8 else
		     "110" when currentState = S3 else
		     "111";

       cmd.angle_en <= '1' when currentState = S0 else '0';	--Sinal do Reg Angle
       cmd.i_en <= '1' when currentState = S8 else '0';		--Sinal do Reg I e do Reg Y
       cmd.cos_en <= '1' when currentState = S12 else '0';	--Sinal do Reg Cos
       cmd.sin_en <= '1' when currentState = S12 else '0';	--Sinal do Reg Sin
       done <= '1' when currentState = S12 else '0';		--Sinal de Saída DONE
       en_Mem <= '1' when currentState = S10 or currentState = S6 else '0'; -- Sinal de habilitação da Memoria

	

end behavioral ;
