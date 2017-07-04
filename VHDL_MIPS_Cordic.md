# VHDL
MIPS em VHDL com as funções necessárias para executar um BubbleSort.   

library IEEE;						
use IEEE.std_logic_1164.all;
use IEEE.std_logic_unsigned.all;
use IEEE.numeric_std.all;
use work.cordic_package.all ;

entity Cordic is
	generic(
		ADDR_WIDTH	: integer := 32;
        	DATA_WIDTH  : integer := 32
		);
	port (
		clk		    	: in std_logic;
        	rst     	    	: in std_logic;
		start	 	    	: in std_logic;
		data_av		    	: in std_logic;
		data		    	: in std_logic_vector(7 downto 0);
		angleTable	  	: in std_logic_vector(DATA_WIDTH -1 downto 0);
		sin		   	: out std_logic_vector(DATA_WIDTH -1 downto 0);
      		cos		    	: out std_logic_vector(DATA_WIDTH -1 downto 0);
		adress		    	: out std_logic_vector(7 downto 0);
		done		    	: out std_logic ;
		en_Mem			: out std_logic 
	);
		
end Cordic;

architecture structural of Cordic is  
   signal if_else, condicao : std_logic;
   signal cmd : Command ;
   		
	 
begin
	CONTROL_PATH: entity work.ControlPath(behavioral)
		port map (
			
			clk		=> clk,
			rst		=> rst,
            		start   	=> start, 
			data_av 	=> data_av,
			done   		=> done,    
			cmd		=> cmd,
			if_else 	=> if_else,
			condicao	=> condicao,
			en_Mem		=> en_Mem
			);

	DATA_PATH: entity work.DataPath(behavioral)
		generic map (
			DATA_WIDTH	=> DATA_WIDTH,
           		ADDR_WIDTH      => ADDR_WIDTH
		)
		port map (
	
			clk	    	=> clk,
			rst	    	=> rst,
			angleTable 	=> angleTable,
			sin 	    	=> sin,
			cos	    	=> cos,
			data        	=> data,
			adress      	=> adress,
			cmd		=> cmd,
			if_else 	=> if_else,
			condicao	=> condicao
	
			);
end structural;

architecture behavioral of Cordic is 
	type State is (S0,S1,S2,S3,S4,S5);
	signal currentState: State;
		
	signal  Reg_Angle, Reg_sumAngle,data_desl,Reg_X, Reg_Y, Reg_It, Reg_I, data_ext : std_logic_vector (DATA_WIDTH -1 downto 0);
begin	
	process(clk, rst)
		begin
			if rst = '1' then
			currentState <= S0;
	
			elsif rising_edge(clk)then
			
			case currentState is
				when S0 =>
					
					Reg_X <= x"009b74ed";
					Reg_I <= (others => '0');
					Reg_Y <= (others => '0');	--Inicialização de Registradores
					Reg_sumAngle <= (others => '0');
					done <= '0';
					
					Reg_Angle <=data_desl;		--Reg Angle recebe o dado extendido e deslocado (pronto)
					
					if data_av = '1' then
					   currentState <= S1;
					else 
					   currentState <= S0;
					end if;
				when S1 =>	
					if data_av = '1' then
						currentState <= S2;
					Reg_It <=("000000000000000000000000" & data);   --Reg It recebe o dado extendido (pronto)
					
					else
						currentState <= S1;
					end if;
				when S2 =>
					if  start = '1' then
						currentState <= S3;
						else
						currentState <= S2;
						end if;

				when S3 =>
						if (Reg_It = Reg_I) then
							done <= '1';
							cos <= Reg_X;
							sin <= Reg_Y;
							currentState <= S5;
						else
							currentState <= S4;
						end if;
				when S4 => 				
						if Reg_angle > Reg_sumAngle then

						Reg_X <= Reg_X - STD_LOGIC_VECTOR(SHIFT_RIGHT(SIGNED(Reg_Y), TO_INTEGER(UNSIGNED(Reg_I(7 downto 0))))); --Reg X contem a subtração de Reg X com (Y>>->i)->X-Y>>i
							
						Reg_Y <= Reg_Y + STD_LOGIC_VECTOR(SHIFT_RIGHT(SIGNED(Reg_X), TO_INTEGER(UNSIGNED(Reg_I(7 downto 0))))); --Reg Y contem a soma de ReY X comX(Y>>->i)->X+Y>>i
					
						Reg_sumAngle <= Reg_sumAngle + (angleTable);
					
						else
						
						Reg_X <= Reg_X + STD_LOGIC_VECTOR(SHIFT_RIGHT(SIGNED(Reg_Y), TO_INTEGER(UNSIGNED(Reg_I(7 downto 0)))));  --Reg X contem a soma de Reg X com Y>>i -> X+(Y>>I)
							

					
						Reg_Y <= Reg_Y - STD_LOGIC_VECTOR(SHIFT_RIGHT(SIGNED(Reg_X), TO_INTEGER(UNSIGNED(Reg_I(7 downto 0)))));	--Reg Y contem a subtração de ReY X comX(Y>>i) -> X-(Y>>i)
					
						Reg_sumAngle <= Reg_sumAngle - (angleTable);
						
						end if;
						Reg_I <= Reg_I + x"0000001";	--Incremento de I
						adress <= Reg_I(7 downto 0) + 1;  --Enderaçamento da memoria, (I+1) para pegar o endereço seguinte, visto que o processandoacontece no mesmo ciclo de clock
						currentState <= S3;
				
				when others =>
					done <= '0';
					currentState <= S0;
			end case;
		end if;
	end process;
	
	data_ext <= ("000000000000000000000000"& data);
	data_desl <=STD_LOGIC_VECTOR(SHIFT_LEFT(UNSIGNED(data_ext), 24));	-- Faz o deslocamento do Dado (Esquerda 24(Decimal))
	en_MEM <= '1' when currentState = S3 else '0';	--Habilita memoria

end behavioral;
