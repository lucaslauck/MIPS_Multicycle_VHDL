# VHDL
MIPS em VHDL com as funções necessárias para executar um BubbleSort.   

library IEEE;
use IEEE.std_logic_1164.all ;
use IEEE.std_logic_unsigned.all ;
use IEEE.numeric_std.all ;
use work.cordic_package.all ;

entity DataPath is
	generic(

	DATA_WIDTH : integer := 32;
	ADDR_WIDTH: integer := 32
	);
	port(
	clk, rst    		: in std_logic;
	data			: in std_logic_vector(7 downto 0);
	angleTable		: in std_logic_vector(DATA_WIDTH -1 downto 0);
	cmd			: in Command ;
	if_else, condicao 	: out std_logic;
	adress			: out std_logic_vector(7 downto 0);
	sin			: out std_logic_vector(DATA_WIDTH -1 downto 0);
	cos			: out std_logic_vector(DATA_WIDTH -1 downto 0)
);
end DataPath;

architecture behavioral of DataPath is

signal data_ext, sum_OUT, in_Angle, out_Angle, data_desl, in_it, out_it, in_sumAngle, out_sumAngle, in_ynew, out_ynew, in_xnew, out_xnew, in_x , out_x : std_logic_vector(DATA_WIDTH -1 downto 0);
signal in_y, out_y, in_i, out_i, in_sin, out_sin, in_cos, out_cos , x_y, desl_xy, opA, opB, sel_shift: std_logic_vector(DATA_WIDTH -1 downto 0);
signal sub : std_logic ;
signal i_8bits : integer;

begin

--Inicio do registrador ANGLE
	data_ext <= ("000000000000000000000000"& data);	-- Extende o Dado que era de 8 bits para 32 (Com zeros a esquerda)
	

	 Reg_Angle: entity work.RegisterNbits(behavioral)
        generic map (
            WIDTH   => ADDR_WIDTH
        )
        port map (
            clk   => clk,
            rst   => rst,
            en    => cmd.angle_en,
            d     => in_Angle,
            q     => out_Angle            
        );

	data_desl <= STD_LOGIC_VECTOR(SHIFT_LEFT(UNSIGNED(data_ext), 24));    -- Faz o deslocamento do Dado (Esquerda 24(Decimal)) 
	in_angle <= data_desl; 

		process (clk, rst)	
			begin 
				if rst = '1' then
					out_angle <= (others => '0');
				elsif rising_edge (clk) then
					if cmd.angle_en = '1' then
						out_angle <= in_angle;
					end if;
				end if;
			end process;
	
--Inicio do registrador IT
	 Reg_It: entity work.RegisterNbits(behavioral)
        generic map (
            WIDTH   => ADDR_WIDTH
        )
        port map (
            clk   => clk,
            rst   => rst,
            en    => cmd.it_en,
            d     => in_it,
            q     => out_it            
        );
	in_it <= data_ext;	--Recebe as Iterações
	
	process (clk, rst)
			begin 
				if rst = '1' then
					out_it <= (others => '0');
				elsif rising_edge (clk) then
					if cmd.it_en = '1' then
						out_it <= in_it;
					end if;
				end if;
			end process;

	--Inicio do registrador SUMANGLE
	 Reg_SumAngle: entity work.RegisterNbits(behavioral)
        generic map (
            WIDTH   => ADDR_WIDTH
        )
        port map (
            clk   => clk,
            rst   => rst,
            en    => cmd.sumAngle_en,
            d     => in_sumAngle,
            q     => out_sumAngle            
        );
	in_sumAngle <= sum_OUT when cmd.mux_x = '0' else x"00000000";  	-- Registrador irá receber a saida do Somador ou Zero "Reset"(mux)  

process (clk, rst)
			begin 
				if rst = '1' then
					out_sumAngle <= (others => '0');
				elsif rising_edge (clk) then
					if cmd.sumAngle_en = '1' then
						out_sumAngle <= in_sumAngle;
					end if;
				end if;
			end process;
	
	--Inicio do registrador Y_NEW
 	Reg_Y_NEW: entity work.RegisterNbits(behavioral)
        generic map (
            WIDTH   => ADDR_WIDTH
        )
        port map (
            clk   => clk,
            rst   => rst,
            en    => cmd.y_new_en,
            d     => in_ynew,
            q     => out_ynew            
        );
	
	in_ynew <= sum_OUT;		-- Irá receber a saída do Somador
		process (clk, rst)
			begin 
				if rst = '1' then
					out_ynew <= (others => '0');
				elsif rising_edge (clk) then
					if cmd.y_new_en = '1' then
						out_ynew <= in_ynew;
					end if;
				end if;
			end process;

	--Inicio do registrador X_NEW
Reg_X_NEW: entity work.RegisterNbits(behavioral)
        generic map (
            WIDTH   => ADDR_WIDTH
        )
        port map (
            clk   => clk,
            rst   => rst,
            en    => cmd.x_new_en,
            d     => in_xnew,
            q     => out_xnew            
        );
	
	in_xnew <= sum_OUT;		-- Irá receber a saída do Somador
		process (clk, rst)
			begin 
				if rst = '1' then
					out_xnew <= (others => '0');
				elsif rising_edge (clk) then
					if cmd.x_new_en = '1' then
						out_xnew <= in_xnew;
					end if;
				end if;
			end process;

	--Inicio do registrador X
	Reg_X: entity work.RegisterNbits(behavioral)
        generic map (
            WIDTH   => ADDR_WIDTH
        )
        port map (
            clk   => clk,
            rst   => rst,
            en    => cmd.x_en,
            d     => in_x,
            q     => out_x            
        );
	
	in_x <= x"009b74ed" when cmd.mux_x ='1' else out_xnew;	--Irá receber Constante de inicialização ou Xnew(mux)
	process (clk, rst)
			begin 
				if rst = '1' then
					out_x <= (others => '0');
				elsif rising_edge (clk) then
					if cmd.x_en = '1' then
						out_x <= in_x;
					end if;
				end if;
			end process;
	
	--Inicio do registrador Y
	Reg_Y: entity work.RegisterNbits(behavioral)
        generic map (
            WIDTH   => ADDR_WIDTH
        )
        port map (
            clk   => clk,
            rst   => rst,
            en    => cmd.i_en, 		--REAPROVEITAMENTO DE SINAL, MESMOS SINAL DO REG I
            d     => in_y,
            q     => out_y            
        );
	
	in_y <= (others =>'0') when cmd.mux_x ='1' else out_ynew;	--Irá receber Zero ou Ynew (mux)
	process (clk, rst)
			begin 
				if rst = '1' then
					out_y <= (others => '0');
				elsif rising_edge (clk) then
					if cmd.i_en = '1' then
						out_y <= in_y;
					end if;
				end if;
			end process;
	
	--Inicio do registrador I
	
	Reg_I: entity work.RegisterNbits(behavioral)
        generic map (
            WIDTH   => ADDR_WIDTH
        )
        port map (
            clk   => clk,
            rst   => rst,
            en    => cmd.i_en, 		
            d     => in_i,
            q     => out_i            
        );

	in_i <= (others => '0') when cmd.mux_x = '1' else sum_OUT;	--Irá receber Zero ou Saída do Somador(mux)
	
	process (clk, rst)
			begin 
				if rst = '1' then
					out_i <= (others => '0');
				elsif rising_edge (clk) then
					if cmd.i_en = '1' then
						out_i <= in_i;
					end if;
				end if;
			end process;

--Inicio do registrador SIN
	
	Reg_sin: entity work.RegisterNbits(behavioral)
        generic map (
            WIDTH   => ADDR_WIDTH
        )
        port map (
            clk   => clk,
            rst   => rst,
            en    => cmd.sin_en, 		
            d     => in_sin,
            q     => out_sin            
        );

	in_sin <= out_y;	-- Irá receber Y
		process (clk, rst)
			begin 
				if rst = '1' then
					out_sin <= (others => '0');
				elsif rising_edge (clk) then
					if cmd.sin_en = '1' then
						out_sin <= in_sin;

					end if;
				end if;
			end process;

--Inicio do registrador COS
	
	Reg_cos: entity work.RegisterNbits
        generic map (
            WIDTH   => ADDR_WIDTH
        )
        port map (
            clk   => clk,
            rst   => rst,
            en    => cmd.cos_en, 		
            d     => in_cos,
            q     => out_cos            
        );

	in_cos <= out_x; -- Irá receber X

		process (clk, rst)
			begin 
				if rst = '1' then
					out_cos <= (others => '0');
				elsif rising_edge (clk) then
					if cmd.cos_en = '1' then
						out_cos <= in_cos;

					end if;
				end if;
			end process;

	--FIM REGISTRADORES

	x_y <= out_y when cmd.x_new_en ='1' else out_x;	-- Net Que leva X ou Y para o Shifter Right (X>>i or Y>>i) (mux)
	
	i_8bits <= TO_INTEGER(UNSIGNED(out_i(7 downto 0)));   -- Net que contém os 8 bits de I, função que converte o valor para Inteiro 
	desl_xy <=  STD_LOGIC_VECTOR(SHIFT_RIGHT(SIGNED(x_y), i_8bits)); --Net que contém X ou Y deslocado em função dos 8 bits de I(X>>i_8its or Y>>i_8bits)(mux)
	
	sel_shift <= desl_xy when cmd.sinal_desl ='1' else (not desl_xy); -- Net que contem + or - (x>>i_8bits or Y>>i_8bits) 
	
	opA <= sel_shift when cmd.mux_sm1 = "00" else
		x"00000001" when cmd.mux_sm1 = "01" else	--Bus_A do Somador, contém Net sel_shift or constante 1 or sumAngle or It(mux 2 bits)
		out_sumAngle when cmd.mux_sm1 = "10" else	
		out_it;

	opB <= out_x when cmd.mux_sm2 = "000" else
		out_y when cmd.mux_sm2 = "001" else
		(not out_angle) when cmd.mux_sm2 = "010" else
		angleTable when cmd.mux_sm2 = "011" else  	--Bus_B do Somador, contém X or Y or !Angle or AngleTable or !AngleTable or I or !I(mux 3 bits)
		(not angleTable) when cmd.mux_sm2 = "100" else
		out_i when cmd.mux_sm2 = "101" else
		(not out_i);
	
	sub <= '1' when cmd.sub = '1' else '0' ;   --Sinal de carry para operações de subtração

	sum_OUT <= (opA) + (opB) + sub ;	-- Somador com CarryIn
	
		if_else <= sum_OUT(DATA_WIDTH -1);	-- Flag de Negativo (Bit MSB da saída do Somador)
			
		condicao <= '1' when sum_OUT = x"00000000" else '0'; --Flag de Zero (Porta NOR entre os Bits da saída do Somador)
	
	adress <= out_i(7 downto 0) ;	-- Endereçamento da memoria contém os primeiros 8 bits de I

	sin <= out_sin ;	--Saída Seno
	
	cos <= out_cos ;	--Saída Cosseno
		
	
end behavioral;
