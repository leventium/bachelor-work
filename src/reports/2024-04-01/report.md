# Отчёт от 01.04.2024


## О чём?

[schoolRISCV Fork](https://github.com/leventium/schoolRISCV)


## Результаты

Разобраться с промышленными ядрами оказалось задачей неподъёмной для нас,
поэтому было принято решение начать с простого и взять schoolRISCV.

Его плюсы:
- Он достаточно прост
- Его микроархитектура и устройство полностью ясны

Минусы:
- **Не** реализует RV32I

Поставлена задача довести schoolRISCV до RV32I, на данный момент момент
реализованы 29 инструкций и осталось реализовать ещё 8 (ecall и ebreak, скорее
всего, будут пропущены).

Некоторые фрагменты реализации:

```verilog
// sr_ram.v

module sr_ram #(
    parameter DEPTH = 256
) (
    input             clk,
    input      [31:0] data_addr,
    input      [31:0] write_data,
    input             we,
    input             sign,
    input             op_word, op_half, op_byte,
    output reg [31:0] read_data
);

    reg [7:0] mem [DEPTH-1:0];

    always @(posedge clk) begin
        if (we) begin
            case ({op_word, op_half, op_byte})
                3'b001: begin
                    mem[data_addr]   = write_data[ 7: 0];
                end
                3'b010: begin
                    mem[data_addr]   = write_data[ 7: 0];
                    mem[data_addr+1] = write_data[15: 8];
                end
                3'b100: begin
                    mem[data_addr]   = write_data[ 7: 0];
                    mem[data_addr+1] = write_data[15: 8];
                    mem[data_addr+2] = write_data[23:16];
                    mem[data_addr+3] = write_data[31:24];
                end
                default: read_data = '0;
            endcase
        end
    end

    always @(*) begin
        case ({op_word, op_half, op_byte})
            3'b001: begin
                read_data = {
                    { 24 { sign ? mem[data_addr][7] : 1'b0 } },
                    mem[data_addr]
                };
            end
            3'b010: begin
                read_data = {
                    { 16 { sign ? mem[data_addr+1][7] : 1'b0 } },
                    mem[data_addr+1],
                    mem[data_addr]
                };
            end
            3'b100: begin
                read_data = {
                    mem[data_addr+3],
                    mem[data_addr+2],
                    mem[data_addr+1],
                    mem[data_addr]
                };
            end
            default: read_data = 0;
        endcase
    end

endmodule
```


```verilog
// sr_control.v

module sr_control
(
    input     [ 6:0] cmdOp,
    input     [ 2:0] cmdF3,
    input     [ 6:0] cmdF7,
    input            aluZero,
    output           pcSrc,     // pcBranch : pcPlus4
    output reg       regWrite,  // rf write enable
    output reg       aluSrc,    // immediate : rd2
    output reg       wdSrc,     // immU : execResult
    output reg       immPick,   // immS : immI
    output reg       memToReg,  // dmDataR : aluResult
    output reg       dmWe,      // data memory write enable
    output reg       dmSign,    // data memory signed read
    output reg [3:0] aluControl,
    output           dmOpByte,   // data memory operation mode
    output           dmOpHalf,
    output           dmOpWord
);
    reg          branch;
    reg          condZero;
    reg    [2:0] dmOpMode;

    assign {dmOpWord, dmOpHalf, dmOpByte} = dmOpMode;
    assign pcSrc = branch & (aluZero == condZero);

    always @ (*) begin
        branch      = 1'b0;
        condZero    = 1'b0;
        regWrite    = 1'b0;
        aluSrc      = 1'b0;
        wdSrc       = 1'b0;
        immPick     = 1'b0;
        memToReg    = 1'b0;
        dmWe        = 1'b0;
        dmSign      = 1'b0;
        aluControl  = `ALU_ADD;
        dmOpMode    = `DM_WORD;

        casez( { cmdF7, cmdF3, cmdOp } )
            default: aluControl = `ALU_ADD;
            { `RVF7_ADD,  `RVF3_ADD,  `RVOP_ADD   } : begin
                regWrite = 1'b1;
                aluControl = `ALU_ADD;
            end
            { `RVF7_OR,   `RVF3_OR,   `RVOP_OR    } : begin
                regWrite = 1'b1;
                aluControl = `ALU_OR;
            end
            { `RVF7_SRL,  `RVF3_SRL,  `RVOP_SRL   } : begin
                regWrite = 1'b1;
                aluControl = `ALU_SRL;
            end
            { `RVF7_SLTU, `RVF3_SLTU, `RVOP_SLTU  } : begin
                regWrite = 1'b1;
                aluControl = `ALU_SLTU;
            end
            { `RVF7_SUB,  `RVF3_SUB,  `RVOP_SUB   } : begin
                regWrite = 1'b1;
                aluControl = `ALU_SUB;
            end
            { `RVF7_XOR,  `RVF3_XOR,  `RVOP_XOR   } : begin
                regWrite = 1'b1;
                aluControl = `ALU_XOR;
            end
            { `RVF7_AND,  `RVF3_AND,  `RVOP_AND   } : begin
                regWrite = 1'b1;
                aluControl = `ALU_AND;
            end
            { `RVF7_SLL,  `RVF3_SLL,  `RVOP_SLL   } : begin
                regWrite = 1'b1;
                aluControl = `ALU_SLL;
            end
            { `RVF7_SRA,  `RVF3_SRA,  `RVOP_SRA   } : begin
                regWrite = 1'b1;
                aluControl = `ALU_SRA;
            end
            { `RVF7_SLT,  `RVF3_SLT,  `RVOP_SLT   } : begin
                regWrite = 1'b1;
                aluControl = `ALU_SLT;
            end


            { `RVF7_ANY,  `RVF3_ADDI, `RVOP_ADDI  } : begin
                regWrite = 1'b1;
                aluSrc = 1'b1;
                aluControl = `ALU_ADD;
            end
            { `RVF7_ANY,  `RVF3_XORI, `RVOP_XORI  } : begin
                regWrite = 1'b1;
                aluSrc = 1'b1;
                aluControl = `ALU_XOR;
            end
            { `RVF7_ANY,  `RVF3_ORI,  `RVOP_ORI   } : begin
                regWrite = 1'b1;
                aluSrc = 1'b1;
                aluControl = `ALU_OR;
            end
            { `RVF7_ANY,  `RVF3_ANDI, `RVOP_ANDI  } : begin
                regWrite = 1'b1;
                aluSrc = 1'b1;
                aluControl = `ALU_AND;
            end
            { `RVF7_ANY,  `RVF3_SLLI, `RVOP_SLLI  } : begin
                regWrite = 1'b1;
                aluSrc = 1'b1;
                aluControl = `ALU_SLL;
            end
            { `RVF7_SLLI, `RVF3_SLLI, `RVOP_SLLI  } : begin
                regWrite = 1'b1;
                aluSrc = 1'b1;
                aluControl = `ALU_SLL;
            end
            { `RVF7_SRLI, `RVF3_SRLI, `RVOP_SRLI  } : begin
                regWrite = 1'b1;
                aluSrc = 1'b1;
                aluControl = `ALU_SRL;
            end
            { `RVF7_SRAI, `RVF3_SRAI, `RVOP_SRAI  } : begin
                regWrite = 1'b1;
                aluSrc = 1'b1;
                aluControl = `ALU_SRA;
            end
            { `RVF7_ANY,  `RVF3_SLTI, `RVOP_SLTI  } : begin
                regWrite = 1'b1;
                aluSrc = 1'b1;
                aluControl = `ALU_SLT;
            end
            { `RVF7_ANY,  `RVF3_SLTIU,`RVOP_SLTIU } : begin
                regWrite = 1'b1;
                aluSrc = 1'b1;
                aluControl = `ALU_SLTU;
            end


            { `RVF7_ANY,  `RVF3_ANY,  `RVOP_LUI   } : begin
                regWrite = 1'b1;
                wdSrc  = 1'b1;
            end


            { `RVF7_ANY,  `RVF3_BEQ,  `RVOP_BEQ   } : begin
                branch = 1'b1;
                condZero = 1'b1;
                aluControl = `ALU_SUB;
            end
            { `RVF7_ANY,  `RVF3_BNE,  `RVOP_BNE   } : begin
                branch = 1'b1;
                aluControl = `ALU_SUB;
            end


            // load / store instructions
            { `RVF7_ANY,  `RVF3_LB,   `RVOP_LOAD  } : begin
                memToReg = 1'b1;
                regWrite = 1'b1;
                aluSrc = 1'b1;
                dmOpMode = `DM_BYTE;
                dmSign = 1'b1;
            end
            { `RVF7_ANY,  `RVF3_LH,   `RVOP_LOAD  } : begin
                memToReg = 1'b1;
                regWrite = 1'b1;
                aluSrc = 1'b1;
                dmOpMode = `DM_HALF;
                dmSign = 1'b1;
            end
            { `RVF7_ANY,  `RVF3_LW,   `RVOP_LOAD  } : begin
                memToReg = 1'b1;
                regWrite = 1'b1;
                aluSrc = 1'b1;
                dmOpMode = `DM_WORD;
                dmSign = 1'b1;
            end
            { `RVF7_ANY,  `RVF3_LBU,  `RVOP_LOAD  } : begin
                memToReg = 1'b1;
                regWrite = 1'b1;
                aluSrc = 1'b1;
                dmOpMode = `DM_BYTE;
            end
            { `RVF7_ANY,  `RVF3_LHU,  `RVOP_LOAD  } : begin
                memToReg = 1'b1;
                regWrite = 1'b1;
                aluSrc = 1'b1;
                dmOpMode = `DM_HALF;
            end

            { `RVF7_ANY,  `RVF3_SB,   `RVOP_STORE } : begin
                dmWe = 1'b1;
                immPick = 1'b1;
                aluSrc = 1'b1;
                dmOpMode = `DM_BYTE;
            end
            { `RVF7_ANY,  `RVF3_SH,   `RVOP_STORE } : begin
                dmWe = 1'b1;
                immPick = 1'b1;
                aluSrc = 1'b1;
                dmOpMode = `DM_HALF;
            end
            { `RVF7_ANY,  `RVF3_SW,   `RVOP_STORE } : begin
                dmWe = 1'b1;
                immPick = 1'b1;
                aluSrc = 1'b1;
                dmOpMode = `DM_WORD;
            end
        endcase
    end
endmodule
```
