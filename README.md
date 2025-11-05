# Basys3

## Archivo FSM
module FSM (
    input  logic clk,
    input  logic reset_n,
    input  logic coin025,
    input  logic coin050,
    input  logic coin1,
    input  logic coin5,
    input  logic coin10,
    input  logic coin20,
    input  logic sel_agua,
    input  logic sel_agua_mineral,
    input  logic sel_cafe_frio,
    input  logic sel_soda,
    input  logic sel_chocolatada,
    input  logic sel_cerveza,
    output logic [7:0] dinero_acumulado,
    output logic dinero_invalido,
    output logic dinero_retorno,
    output logic producto
);
    typedef enum logic [2:0] {
        S0, S1, S2, S3, S4, S5, S6
    } state_t;

    state_t estado, siguiente_estado;
    logic [7:0] total;
    always_ff @(posedge clk or negedge reset_n) begin
        if (!reset_n)
            estado <= S0;
        else
            estado <= siguiente_estado;
    end
    always_comb begin
        siguiente_estado = estado;
        case (estado)
            S0: if (coin025) siguiente_estado = S1;
                else if (coin050) siguiente_estado = S2;
                else if (coin1) siguiente_estado = S3;
                else if (coin5) siguiente_estado = S4;
                else if (coin10) siguiente_estado = S5;
                else if (coin20) siguiente_estado = S6;
            default: siguiente_estado = estado;
        endcase
    end

    always_ff @(posedge clk or negedge reset_n) begin
        if (!reset_n)
            total <= 0;
        else begin
            case (estado)
                S1: total <= 25;
                S2: total <= 50;
                S3: total <= 100;
                S4: total <= 500;
                S5: total <= 1000;
                S6: total <= 2000;
                default: total <= 0;
            endcase
        end
    end

    assign dinero_acumulado = total[7:0];
    assign producto = (sel_agua && total >= 15) || (sel_cerveza && total >= 10); 
endmodule




## Display
module Display (
    input  logic clk,
    input  logic reset_n,
    input  logic [15:0] valor,
    output logic [3:0] an,
    output logic [6:0] seg
);
    logic [1:0] sel;
    logic [3:0] digito;

    always_ff @(posedge clk or negedge reset_n) begin
        if (!reset_n)
            sel <= 0;
        else
            sel <= sel + 1;
    end

    always_comb begin
        case (sel)
            2'b00: digito = valor[3:0];
            2'b01: digito = valor[7:4];
            2'b10: digito = valor[11:8];
            2'b11: digito = valor[15:12];
        endcase
    end

    display_decoder decoder (.num(digito), .seg(seg));
    assign an = ~(4'b0001 << sel);
endmodule





## Decodificación para los segmentos
module Decod(
    input  logic [3:0] num,
    output logic [6:0] seg
);
    always_comb begin
        case (num)
            4'h0: seg = 7'b1000000;
            4'h1: seg = 7'b1111001;
            4'h2: seg = 7'b0100100;
            4'h3: seg = 7'b0110000;
            4'h4: seg = 7'b0011001;
            4'h5: seg = 7'b0010010;
            4'h6: seg = 7'b0000010;
            4'h7: seg = 7'b1111000;
            4'h8: seg = 7'b0000000;
            4'h9: seg = 7'b0010000;
            default: seg = 7'b1111111;
        endcase
    end
endmodule





## Modulos para basys
module Modulos(
    input  logic CLK100MHZ,
    input  logic [15:0] SW,
    input  logic [3:0] BTN,
    output logic [6:0] SEG,
    output logic [3:0] AN
);
    logic reset_n = ~BTN[0];
    logic [7:0] dinero;
    logic [15:0] valor_display;

    fsm_maquina_expendedora fsm(
        .clk(CLK100MHZ),
        .reset_n(reset_n),
        .coin025(SW[0]),
        .coin050(SW[1]),
        .coin1(SW[2]),
        .coin5(SW[3]),
        .coin10(SW[4]),
        .coin20(SW[5]),
        .sel_agua(SW[6]),
        .sel_agua_mineral(SW[7]),
        .sel_cafe_frio(SW[8]),
        .sel_soda(SW[9]),
        .sel_chocolatada(SW[10]),
        .sel_cerveza(SW[11]),
        .dinero_acumulado(dinero)
    );

    assign valor_display = {8'd0, dinero};
    mux_display display (
        .clk(CLK100MHZ),
        .reset_n(reset_n),
        .valor(valor_display),
        .an(AN),
        .seg(SEG)
    );
endmodule
