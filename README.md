module sdram_controller(

input clk,
input reset,

input write_req,
input read_req,

input [23:0] addr,
input [15:0] wdata,

output reg [15:0] rdata,
output reg busy,

output sdram_clk,
output reg sdram_cke,
output reg sdram_cs_n,
output reg sdram_ras_n,
output reg sdram_cas_n,
output reg sdram_we_n,

output reg [12:0] sdram_addr,
output reg [1:0] sdram_ba,

inout [15:0] sdram_dq
);

assign sdram_clk = clk;

parameter TRCD = 2;
parameter TCL = 3;
parameter TRP = 2;

localparam IDLE = 0;
localparam ACTIVATE = 1;
localparam WAIT_TRCD = 2;
localparam READ = 3;
localparam WRITE = 4;
localparam WAIT_CL = 5;
localparam PRECHARGE = 6;
localparam WAIT_TRP = 7;

reg [3:0] state;
reg [7:0] timer;

reg dq_en;
reg [15:0] dq_out;

assign sdram_dq = (dq_en) ? dq_out : 16'bz;

always @(posedge clk) begin

if(reset) begin

    state <= IDLE;

    timer <= 0;

    busy <= 0;

    sdram_cke <= 1;

end

else begin

    case(state)

        IDLE: begin

            busy <= 0;

            dq_en <= 0;

            if(write_req || read_req) begin

                busy <= 1;

                state <= ACTIVATE;

            end

        end

        ACTIVATE: begin

            sdram_cs_n  <= 0;
            sdram_ras_n <= 0;
            sdram_cas_n <= 1;
            sdram_we_n  <= 1;

            sdram_addr <= addr[23:11];
            sdram_ba   <= addr[10:9];

            timer <= TRCD;

            state <= WAIT_TRCD;

        end

        WAIT_TRCD: begin

            if(timer == 0) begin

                if(write_req)
                    state <= WRITE;

                else
                    state <= READ;

            end

            else
                timer <= timer - 1;

        end

        WRITE: begin

            sdram_cs_n  <= 0;
            sdram_ras_n <= 1;
            sdram_cas_n <= 0;
            sdram_we_n  <= 0;

            sdram_addr <= addr[8:0];

            dq_en  <= 1;
            dq_out <= wdata;

            state <= PRECHARGE;

        end

        READ: begin

            sdram_cs_n  <= 0;
            sdram_ras_n <= 1;
            sdram_cas_n <= 0;
            sdram_we_n  <= 1;

            sdram_addr <= addr[8:0];

            dq_en <= 0;

            timer <= TCL;

            state <= WAIT_CL;

        end

        WAIT_CL: begin

            if(timer == 0) begin

                rdata <= sdram_dq;

                state <= PRECHARGE;

            end

            else
                timer <= timer - 1;

        end

        PRECHARGE: begin

            sdram_cs_n  <= 0;
            sdram_ras_n <= 0;
            sdram_cas_n <= 1;
            sdram_we_n  <= 0;

            timer <= TRP;

            state <= WAIT_TRP;

        end

        WAIT_TRP: begin

            if(timer == 0)

                state <= IDLE;

            else

                timer <= timer - 1;

        end

    endcase

end
end

endmodule
