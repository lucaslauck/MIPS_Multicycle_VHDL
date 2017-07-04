# VHDL
MIPS em VHDL com as funções necessárias para executar um BubbleSort.   

library IEEE;
use IEEE.std_logic_1164.all;
use IEEE.numeric_std.all;
use IEEE.std_logic_unsigned.all; 


entity Mips_Multicycle is

generic (
        PC_START_ADDRESS    : integer := 0
    );
    port ( 
        clock, reset        : in std_logic;
        
        -- Instruction memory interface
        instructionAddress  : out std_logic_vector(31 downto 0);
        instruction         : in  std_logic_vector(31 downto 0);
        
        -- Data memory interface
        dataAddress         : out std_logic_vector(31 downto 0);
        data_i              : in  std_logic_vector(31 downto 0);      
        data_o              : out std_logic_vector(31 downto 0);
        wbe            	    : out std_logic_vector(3 downto 0);
		ce		    		: out std_logic
    );
end Mips_Multicycle;

architecture behavioral of Mips_Multicycle is
	type State is (S0, S1, S2, S3, S4, S5, S6, S7, S8, S9, S10) ;
	type Instruction_type is (ADD, SUB, ADDI, SW, LW, SLT, SLLL, BEQ, SRAV, J, LUI, ORI, BNE, INVALID_INSTRUCTION);
	
	signal currentState: State;
	signal DecodedInst : Instruction_type ; ---- Instrução
	signal RegWr	: std_logic;
	signal wrReg	: std_logic_vector(4 downto 0);	

	signal i_8bits	: integer;
	signal IR, SignalExt32, rigtS  : std_logic_vector(31 downto 0); -- entrada registrador de instruções	

	signal MEMwrite, Cin : std_logic; -- entrada registrador de instruções	
	
	signal writeData, A ,B, ALU_out,result, Somador, PC, Op_1 , OP_2, ShiftExt, AdressJump, Shifter,ShiftArit : UNSIGNED(31 downto 0);
	
	
	
	alias rs: std_logic_vector(4 downto 0) is IR(25 downto 21);
        
    -- Retrieves the rt field from the instruction
    alias rt: std_logic_vector(4 downto 0) is IR(20 downto 16);
        
    -- Retrieves the rd field from the instruction
    alias rd: std_logic_vector(4 downto 0) is IR(15 downto 11);
    
	-- Alias to identify the instructions based on the 'opcode' and 'funct' fields
    alias  opcode: std_logic_vector(5 downto 0) is IR(31 downto 26);
    alias  funct: std_logic_vector(5 downto 0) is IR(5 downto 0);
	
	
	 -- Register file
    type RegisterArray is array (natural range <>) of UNSIGNED(31 downto 0);
    signal registerFile: RegisterArray(0 to 31);

begin
	
	-------- Instruction decoding ------------------------------
    	DecodedInst      <= ADD     when opcode = "000000" and funct = "100000" else
				SUB     when opcode = "000000" and funct = "100010" else
				SLLL	when opcode = "000000" and funct = "000000" else
				SLT     when opcode = "000000" and funct = "101010" else
				SRAV	when opcode = "000000" else
				SW      when opcode = "101011" else
				LW      when opcode = "100011" else
				ADDI    when opcode = "001000" else
				ORI     when opcode = "001101" else
				BEQ     when opcode = "000100" else
				BNE 	when opcode = "000101" else
				J       when opcode = "000010" else
				LUI     when opcode = "001111" and rs = "00000" else
				INVALID_INSTRUCTION ;    -- Invalid or not implemented instruction
            
    	assert not (decodedInst = INVALID_INSTRUCTION and reset = '0')    
    	report "******************* INVALID INSTRUCTION *************"
    	severity error;
	-------------------------------------------------------------	


process(clock,reset)
	
	begin 
		if reset = '1' then

			PC <= (TO_UNSIGNED(PC_START_ADDRESS,32));
			
			----- Reg FILE ----
                	for i in 0 to 31 loop   
                registerFile(i) <= (others=>'0');  
            end loop;
            		
			--------------------
			currentState <= S0;

	
		elsif rising_edge(clock)then

			
			case currentState is


				When S0 =>					
			  	PC <= Somador ; -- Soma PC para a proxima leitura 
					IR <= instruction ; -- Dados da memoria de instruções são gravso sno registrador de instruções 
					currentState <= S1 ;
	
				When S1 =>
							
						ALU_out	<= Somador;		
					
						if decodedInst = LW or DecodedInst = SW then	-- Se for um Load ou Store Vai para S2
								currentState <= S2;
						
						elsif decodedInst = BEQ  then					-- Senão, se for um Branch Equal vai para o S8
								currentState <= S8;
						elsif decodedInst = BNE then 
								currentState <= S9;
						
						elsif decodedInst = J then						-- Senão, se for um Jump incondicional vai para o S9
								currentState <= S10;
	
						else
						currentState <= S6 ;							-- Senão por default vai para o S6
						
					end if;
					
				When S2 =>  -- Se for LW or SW
			
						ALU_out <= Somador ;
						if decodedInst = LW then 
						currentState <= S3 ;  
						
						else   -- Se nao for LW é SW
						currentState <= S5;
					end if;
					
				When S3 =>  -- LW
			
						-- Acesso a memoria 
					dataAddress	<=	STD_LOGIC_VECTOR(ALU_out);
					currentState <= S4;
				
				When S4 => 
					
							if RegWr = '1' and UNSIGNED(wrReg) /= 0 then	-- Register $0 is read-only (constant 0)
						registerFile(TO_INTEGER(UNSIGNED(wrReg))) <= UNSIGNED(data_i);
						
							end if;
						currentState <= S0;	
				When S5 =>
						
							dataAddress	<=	STD_LOGIC_VECTOR(ALU_out);
							data_o <= STD_LOGIC_VECTOR(B);
						
						currentState <= S0;
				
				When S6 =>
					
						ALU_out <= Somador;
						if decodedInst = SLLL then	
							registerFile(TO_INTEGER(UNSIGNED(wrReg))) <= Shifter;
							currentState <= S0;
							
						elsif decodedInst = SRAV then	
							registerFile(TO_INTEGER(UNSIGNED(wrReg))) <= ShiftArit;	
							currentState <= S0;
							
						else	
						currentState <= S7;
						
						end if;
						
				when S7 =>
						
						if RegWr = '1' and UNSIGNED(wrReg) /= 0 then	-- Register $0 is read-only (constant 0)
						registerFile(TO_INTEGER(UNSIGNED(wrReg))) <= ALU_out;
					end if;
						
						currentState <= S0;
						
				when S8 =>		-- Branch Completion
					if (A = B) then
						PC	<= ALU_out;
						currentState <= S0;
					else 
						currentState <= S0;						
					end if;
			when S9 =>
					if (A /= B) then
						PC <= ALU_out;
						currentState <= S0;
					else
				    currentState <= S0;
					end if;
						
						
				when others =>
				PC	<= AdressJump;
					currentState <= S0 ;
				
				end case;
		end if ;
end process;

A		<= registerFile(TO_INTEGER(UNSIGNED(rs)));	-- A <- R[rs]
B		<= registerFile(TO_INTEGER(UNSIGNED(rt)));	-- B <- R[rt]

instructionAddress <= STD_LOGIC_VECTOR(PC);	 -- Endereçamento da memoria de instrução pelo PC
	
RegWr	<=	'1' when currentState = S4 or currentState = S7 else '0'; -- Sinal de Escrita no Registrador habilitado somente em S4 e S7

wrReg	<= 	rd when opcode = "000000" else rt; -- wrReg é o endereço de acesso do registrador que deseja usar, vai ser rd em op tipo R se nao sera rt

--Memory

wbe <= "1111" when decodedInst = SW else (others=>'0'); -- WBE em 1111 no SW Grava em Word's
	
ce <= '1' when decodedInst = SW or decodedInst = LW else '0'; -- Memoria ativada quando for operações de load ou Store se não desativada

-----

SignalExt32		<=	(x"FFFF" & IR(15 downto 0)) when IR(15) = '1' else 
				(x"0000" & IR(15 downto 0));

ShiftExt	<=	UNSIGNED(SignalExt32(29 downto 0) & "00");  

AdressJump	<=	(PC(31 downto 28) & UNSIGNED(IR(25 downto 0)) & TO_UNSIGNED(0,2)); -- Endereço de Jump obtido a partir da comcatenação do PC e e da Intrução
				
----------------- ULA --------------------
				
OP_1		<= 	A when currentState = S2 or currentState = S6 or currentState = S8     
				else(A(31 downto 16) & x"0000") when currentState = S6 and decodedInst = ORI 
				else PC;

OP_2		<=	x"00000004" 	when currentState = S0
				else UNSIGNED(SignalExt32)	when currentState = S2 
				else ShiftExt	when currentState = S1 
				else B	when opcode = "000000"
				else UNSIGNED(SignalExt32);
				
Shifter <= (SHIFT_LEFT(UNSIGNED(registerFile(TO_INTEGER(UNSIGNED(wrReg)))), 24)) when currentState = S6 else (others => '0');    -- Faz o deslocamento do Dado (Esquerda 24(Decimal))

	
i_8bits <= (TO_INTEGER (UNSIGNED(registerFile(TO_INTEGER(UNSIGNED(rs)))(7 downto 0))))when DecodedInst = SRAV else 0; 

rigtS <=  STD_LOGIC_VECTOR(SHIFT_RIGHT(SIGNED(registerFile(TO_INTEGER(UNSIGNED(rt)))), i_8bits)) when decodedInst = SRAV else  (others => '0');
ShiftArit <= UNSIGNED (rigtS);

Somador		<=  OP_1 - OP_2 		when decodedInst = SUB 	and currentState = S6	else
               OP_1 or  OP_2		when decodedInst = ORI and currentState = S6			else 
                (0=>'1', others=>'0') 	when decodedInst = SLT and (SIGNED(OP_1) < SIGNED(OP_2)) and currentState = S6		else
                (others=>'0') 			when decodedInst = SLT and not (SIGNED(OP_1) < SIGNED(OP_2))	and currentState = S6 else
				OP_2(15 downto 0) & x"0000" 	when decodedInst = LUI		and currentState = S6				else
				OP_1 + OP_2;
				
 	
	end behavioral ; 
