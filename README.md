# UART-Design
Design and Verification of a UART using System Verilog concepts.


## UART Design Module

``` systemverilog

//UART design

// ---------------------------------UART top module--------------------------

module uart_top #(parameter clk_freq = 1000000, 
                 parameter baud_rate = 9600)
  (clk,rst,rx,dintx,newd,tx,doutrx,donetx,donerx);
  
  input clk,rst,rx,newd;
  input [7:0] dintx;
  output tx, donetx, donerx;
  output [7:0] doutrx;
  
  //instantiating UART Tx module
  uarttx #(clk_freq,baud_rate)
  utx (.clk(clk),.rst(rst),.newd(newd),.tx_data(dintx),.tx(tx),.donetx(donetx));
  
  //instantiating UART Rx module
  uartrx #(clk_freq,baud_rate)
  rtx(.clk(clk),.rst(rst),.rx(rx),.done(donerx),.rxdata(doutrx));
  
endmodule

// ---------------------------------UART Transmitter module---------------------------

module uarttx #(parameter clk_freq = 1000000, 
                 parameter baud_rate = 9600)
  (clk,rst,tx_data,newd,tx,donetx);
  
  input clk,rst,newd;
  input [7:0] tx_data;
  output reg tx, donetx;
  
  localparam clkcount = (clk_freq/baud_rate); //for slow clock generation
  
  integer count = 0; // for slow clock
  integer counts = 0; // to keep track of data bit count
  
  reg uclk = 0; // clock for UART
  
  enum bit {idle = 1'b0, transmit = 1'b1} state;
  
  //UART clock generation - slow clock
  always @(posedge clk)
    begin
      if(count < clkcount/2)
        begin
          count <= count + 1;
        end else begin
          count <= 0;
          uclk <= ~uclk;
        end
    end
  
  reg [7:0] din; //to store data temporary
  
  //main logic
  always @(posedge uclk)
    begin
      if(rst)
        begin 
          state <= idle;
        end
      else 
        begin
          case(state)
            
            idle : begin
              		counts <= 0;
              		tx <= 1'b1;
              		donetx <= 1'b0;
              
              		if(newd)
                      begin
                        din <= tx_data; //store tx_data into din
                        tx <= 1'b0;     //pull tx pin low
                        state <= transmit;
                      end
              		else
                      state <= idle;
            	   end
            
            transmit : begin
                        if(counts <= 7)
                          begin
                            tx <= din[counts];   //transmit data serially
                            counts <= counts + 1;
                            state <= transmit;
                          end
                        else
                          begin
                            counts <= 0;
                            tx <= 1'b1;
                            donetx <= 1'b1;
                            state <= idle;
                          end
            		   end
            default : state <= idle;
            
          endcase
        end
    end
endmodule

// ---------------------------------UART Reciever module-------------------------------

module uartrx #(parameter clk_freq = 1000000, 
                parameter baud_rate = 9600)
  (clk,rst,rx,done,rxdata);
  input clk,rst,rx;
  output reg done;
  output reg [7:0] rxdata;
  
  localparam clkcount = (clk_freq/baud_rate); //for slow clock generation
  
  integer count = 0; // for slow clock
  integer counts = 0; // to keep track of data bit count
  
  reg uclk = 0; // clock for UART
  
  enum bit {idle = 1'b0, recieve = 1'b1} state;
  
  //UART clock generation - slow clock
  always @(posedge clk)
    begin
      if(count < clkcount/2)
        begin
          count <= count + 1;
        end else begin
          count <= 0;
          uclk <= ~uclk;
        end
    end
  
  //reciever block main logic
  always @(posedge uclk)
    begin
      if(rst)
        begin
          rxdata <= 8'h00;
          counts <= 0;
          done <= 1'b0;
        end
      else 
        begin
          
          case(state)
            
            idle : begin
              		rxdata <= 8'h00;
          			counts <= 0;
          			done <= 1'b0;
              
                  if(!rx)
                    state <= recieve;
                  else
                    state <= idle;
            	   end
            
            recieve : begin
                        if(counts <= 7)
                          begin
                            rxdata <= {rx, rxdata[7:1]}; // right shift rxdata to 														    accomodate rx bit by bit
                            counts <= counts + 1;
                          end
                        else
                          begin
                            count <= 0;
                            done <= 1'b1;
                            state <= idle;
                          end
            		  end
            
            default : state <= idle;
            
          endcase
          
        end
    end  
  
endmodule


//-----------------------------------------interface---------------------------

interface uart_if;
  
  logic clk,uclktx,uclkrx;
  logic rst,rx;
  logic [7:0] dintx;
  logic newd;
  logic tx;
  logic [7:0] doutrx;
  logic donetx, donerx;
  
endinterface



```


## TESTBENCH for UART Design

``` systemverilog

//UART TESTBENCH

// -----------------------------transaction class---------------------

class transaction;
  
  typedef enum bit { write = 1'b0, read = 1'b1} oper_type;
  randc oper_type oper;
  rand bit [7:0] dintx;
  
  bit rx;
  bit newd;
  bit tx;
  bit [7:0] doutrx;
  bit donetx, donerx;
  
  //deep copy 
  function transaction copy();
    copy = new();
    copy.rx = this.rx;
    copy.dintx = this.dintx;
    copy.newd = this.newd;
    copy.tx = this.tx;
    copy.doutrx = this.doutrx;
    copy.donetx = this.donetx;
    copy.donerx = this.donerx;
    copy.oper = this.oper;
  endfunction
  
endclass

// -----------------------------generator class------------------------------

class generator;
  
  transaction tr;
  
  mailbox #(transaction) mbx;
  
  event done, drvnext, sconext;
  
  int count = 0;
  
  //custom constructor
  function new(mailbox #(transaction) mbx);
    this.mbx = mbx;
    tr = new();  
  endfunction
  
  
  //main task
  task run();
    
    repeat(count)begin
      
      assert(tr.randomize) else $error("[GEN] : RANDOMIZATION FAILED!!");
      mbx.put(tr.copy);
      $display("[GEN] : Oper : %0s Din : %0d", tr.oper.name(),tr.dintx);
      @(drvnext);
      @(sconext);
      
    end
    
    ->done; // trigger the done event once all stimulus has been generated
    
  endtask
  
endclass

//----------------------------------driver class-------------------------------

class driver;
  
  virtual uart_if vif; // interface for driver to connect to DUT
  
  transaction tr;
  
  mailbox #(transaction) mbx;  // mbx between gen-drv
  mailbox #(bit [7:0]) mbxds; // mbx between drv-sco
  
  event drvnext;//event to mark completion of applying a stimuli to DUT
  
  
  bit [7:0] din; //to store data coming serially from tx
  
  bit [7:0] datarx; //data recieved during read
  
  
  //custom constructor
  function new(mailbox #(bit [7:0]) mbxds, mailbox #(transaction) mbx);
    this.mbx = mbx;
    this.mbxds = mbxds;
  endfunction
  
  
  //reset DUT task
  task reset();
    
    vif.rst <= 1'b1;
    vif.dintx <= 0;
    vif.newd <=  1'b0;
    vif.rx <= 1'b1;
    
    repeat(5) @(posedge vif.uclktx);
    vif.rst <= 1'b0;
    @(posedge vif.uclktx);
    $display("[DRV] : RESET DONE!!");
    $display("----------------------------------------------------------");
    
  endtask
  
  
  //main task for driver
  task run();
    
   forever begin
     
     mbx.get(tr);
     
     if(!tr.oper)              // write operation (data transmission)
       begin
         
         @(posedge vif.uclktx);
         vif.rst <= 1'b0;
         vif.newd <= 1'b1;
         vif.rx <= 1'b1;
         vif.dintx <= tr.dintx;  
         @(posedge vif.uclktx);
         vif.newd <= 1'b0;
         mbxds.put(tr.dintx); //send dintx data to scoreborad for comparision
         $display("[DRV] : Data Sent : %0d",tr.dintx);
         wait(vif.donetx);
         ->drvnext;
         
       end
     
     else
       begin
         // read operation (data recieve)
         @(posedge vif.uclkrx);
         vif.rst <= 1'b0;
         vif.rx <= 1'b0; //marks the start of data rx by pulling rx pin low
         vif.newd <= 1'b0;
         @(posedge vif.uclkrx);
         
         for(int i=0; i<=7; i++)
           begin
             @(posedge vif.uclkrx);
             vif.rx <= $urandom;   //generate random data
             datarx[i] = vif.rx;  
           end
         
         mbxds.put(datarx);
         
         $display("[DRV] : DATA RECIEVED : %0d", datarx);
         wait(vif.donerx); //wait till donerx is asserted
         vif.rx <= 1'b1; //make the rx pin high
         ->drvnext;
       end
      
   end
    
  endtask
  
endclass

// --------------------------------monitor class--------------------------------

class monitor;
  
  virtual uart_if vif;
  
  transaction tr;
  
  mailbox #(bit[7:0]) mbx;
  
  bit [7:0] srx; //send
  bit [7:0] rrx; //recieve
  
  //custom constructor
  function new(mailbox #(bit[7:0]) mbx);
    this.mbx = mbx;
  endfunction
  
  //main task for monitor
  task run();
    
    forever begin
      
      @(posedge vif.uclktx);
      if((vif.newd) && (vif.rx)) //if both newd and rx are high -> write op (data tx)
        begin
          
          @(posedge vif.uclktx); //start collecting tx data from next clk tick
          
          for(int i=0; i<=7; i++)
            begin
              @(posedge vif.uclktx);
              srx[i] = vif.tx;
            end
          
          $display("[MON] : DATA SEND ON UART TX %0d", srx);
          @(posedge vif.uclktx);
          mbx.put(srx); //send data to sco for comparision
          
        end
      else if((!vif.rx) && (!vif.newd)) //if both rx and newd are low -> read op (data rx)
        begin
          wait(vif.donerx);
          rrx = vif.doutrx; //transfer rx data to rrx variable
          $display("[MON] : DATA RECIEVED ON RX %0d", rrx);
          @(posedge vif.uclkrx);
          mbx.put(rrx); //send data to sco for comparision
        end
      
    end
    
  endtask
  
endclass

//-----------------------------------scoreboard class---------------------------------

class scoreboard;
  
  mailbox #(bit [7:0]) mbxds, mbxms;
  
  bit [7:0] ds;
  bit [7:0] ms;
  
  event sconext; //event to mark completion of applying stimuli to sco
  
  //custom constructor
  function new(mailbox #(bit [7:0]) mbxds, mailbox #(bit [7:0]) mbxms);
    this.mbxds = mbxds;
    this.mbxms = mbxms;
  endfunction 
  
  //main task for scoreboard
  task run();
    
    forever begin
      
      mbxds.get(ds);
      mbxms.get(ms);
      
      $display("[SCO] : DRV : %0d MON : %0d",ds,ms);
      if(ds == ms)
        $display("[SCO] : DATA MATCHED!!");
      else
        $display("[SCO] : DATA MISMATCHED!!");
      
      $display("-------------------------------------------------------");
      
      ->sconext; //trigger sconext event
     
    end
    
  endtask
  
endclass

// ------------------------------------environment class------------------------------

class environment;
  
  //instances for all class variables
  generator gen;
  driver drv;
  monitor mon;
  scoreboard sco;
  
  event nextgd; // gen-drv
  event nextgs; // gen-sco
  
  mailbox #(transaction) mbxgd; // mbx b/w gen-drv
  mailbox #(bit[7:0]) mbxds;  // mbx b/w drv-sco
  mailbox #(bit[7:0]) mbxms; //mbx b/w mon-sco
  
  virtual uart_if vif;
  
  
  //custom constructor
  function new(virtual uart_if vif);
    
    mbxgd = new();
    mbxds = new();
    mbxms = new();
    
    gen = new(mbxgd);
    drv = new(mbxds,mbxgd);
    mon = new(mbxms);
    sco = new(mbxds,mbxms);
    
    this.vif = vif;
    drv.vif = this.vif;
    mon.vif = this.vif;
    
    //connecting sconext in gen and sco to nextgs in env
    gen.sconext = nextgs;
    sco.sconext = nextgs;
    
    //connecting drvnext in gen and drv to nextgd in env
    gen.drvnext = nextgd;
    drv.drvnext = nextgd;
    
  endfunction
  
  
  //pretest - reset the DUT via driver
  task pre_test();
    drv.reset();
  endtask
  
  //test - run the main task of all the classes
  task test();
    fork
      gen.run();
      drv.run();
      mon.run();
      sco.run();
    join_any
  endtask
 
  // post_test - run the cleanup
  task post_test();
    wait(gen.done.triggered)
    $finish();
  endtask
  
  
  //main task for env clas
  task run();
    pre_test();
    test();
    post_test();
  endtask
  
endclass

// ----------------------------------testbench top-----------------------------------

module tb;
  
  uart_if vif();
  
  //instantiating the DUT
  uart_top #(1000000, 9600) 
  dut (.clk(vif.clk),.rst(vif.rst),.rx(vif.rx),.dintx(vif.dintx),.newd(vif.newd),
       .tx(vif.tx),.doutrx(vif.doutrx),.donetx(vif.donetx),.donerx(vif.donerx));
  
  initial begin
    vif.clk = 0;
  end
  
  always #10 vif.clk = ~vif.clk;
  
  environment env;
  
  initial begin
    env = new(vif);
    env.gen.count = 5;
    env.run();
  end
  
  
  initial begin
    $dumpfile("dump.vcd");
    $dumpvars;
  end
  
  assign vif.uclktx = dut.utx.uclk;
  assign vif.uclkrx = dut.rtx.uclk;
  
endmodule

// -----------------------------------------------------------------------------------

```

## Console Output

![image](https://github.com/user-attachments/assets/1feab8ae-fd45-4692-a24d-969e76511908)




