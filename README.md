#  SDRAM Controller and SDRAM in Caravel User Project
## 目的
### 將LAB4-1內的wishbone-BRAM替換成Wishbond SDRAM+SDRAM-Controller
#### 1. firmware 描述 "matmul" 而後執行在SDRAM 

## SDRAM 背景知識
https://hackmd.io/AimapH7MQqaiewDWPtS43Q?both

## 下載安裝執行包與解說
https://github.com/bol-edu/caravel-soc_fpga-lab/tree/main/lab-sdram
確認更改程式後，最後輸出結果與以下結果相同表示更改路徑都是正確的
```
Reading counter_la.hex
counter_la.hex loaded into memory
Memory 5 bytes = 0x6f 0x00 0x00 0x0b 0x13
VCD info: dumpfile counter_la.vcd opened for output.
LA Test 1 started
Call function adder() in User Project BRAM (mprjram, 0x38000000) return value passed, 0x2233
(這一段會根據firmware的不同而有所更動)
LA Test 2 passed
```
解說
https://www.youtube.com/watch?v=VfGQYhy2aak&list=PLTA_T2FLzYNAvdvpEQvTjXaPARtmV-0GM&index=31
## SDRAM 結構
![image](https://hackmd.io/_uploads/BJUhtCbw6.png)

#### 1. SDRAM controller(紅框)
檔案位置 LABD/rtl/user/sdram_controller.v  
此lab主要更改此code

       user_project 收到的位址訊號進而決定"read & write address"
1. controller 與 wishbond interface之間需要有一個converter作為訊號轉換

#### 2. SDRAM device
檔案位置 LABD/rtl/user/sdr.v
基本不動它

    使用Bram儲存資料 
#### 3. Controller-Wishbone Singnal Converter
檔案位置 LABD/rtl/user/user_proj_example.counter.v
controller 和 wishbone之間的轉換code
![image](https://hackmd.io/_uploads/Hyo9ExfP6.png)

controller 包含 2個訊號
1. busy:(1) 表示controller正在執行指令，執行完成後busy 自動->0
2. out_valid:(0) 執行完成read cmd後，需要將自資料送回wishbond時將out_valid(主動) ->1
```
但是 wishbond 只有 "1"個output signal "wbs_ack_o"因此我們需要
> convert busy & out_valid into "one signal" 如此將SDRAM放入user_project 
```
## SDRAM Controller design 
### Flower Chart
![image](https://hackmd.io/_uploads/r12JbtRua.png)
### 架構圖
![11111](https://hackmd.io/_uploads/rJlIbKA_6.jpg)
![222](https://hackmd.io/_uploads/B1-r7K0dT.png)

## 	SDRAM bus protocol
### SDRAM DEVICE
![image](https://hackmd.io/_uploads/S15xQ9AuT.png)
[1]	Input /output pin:
1. CKE  : clk enable
2. CS_N : disable other input, except "CLK" "CKE" "DQM"
3. CAS_N: A[8:0] 列address
4. RAS_N: A[12:0] 行address
5. WE_N : write_enable
6. BA[1:0]: 總共4個BLANK
7. A[12:0]: address(不同命令定義不同)
8. DQ[15:0]:data register input/output

[2]	decode:  
![image](https://hackmd.io/_uploads/BkNzX90OT.png)  
經過decode會把各項需要執行的描述都在sdram.v裡面進行  

prech_enable :  
若要切換bank，必須要先將目前正在使用的bank關閉後才可以切換新的bank  

### Wishbone 和 Sdram controller
因為Wishbond 只有一個響應訊號 wbs_ack_o 而 Sdram會由ctrl_busy&valid共同決定，因此需要轉換  
[1]. 轉換 Wishbone 和 SDRAM controller 的訊號  
```
assign valid = wbs_stb_i && wbs_cyc_i;
assign ctrl_in_valid = wbs_we_i ? valid : ~ctrl_in_valid_q && valid;
assign wbs_ack_o = (wbs_we_i) ? ~ctrl_busy && valid : ctrl_out_valid; 
assign bram_mask = wbs_sel_i & {4{wbs_we_i}};
assign ctrl_addr = wbs_adr_i[22:0];
```
[2]	Wishbone address 的末 23 bits 會直接映射到 SDRAM address  
```
always @(posedge clk) begin
      if (rst) begin
          ctrl_in_valid_q <= 1'b0;
      end
      else begin
          if (~wbs_we_i && valid && ~ctrl_busy && ctrl_in_valid_q == 1'b0)
              ctrl_in_valid_q <= 1'b1;
          else if (ctrl_out_valid)
              ctrl_in_valid_q <= 1'b0;
      end
end
```
### Introduce the prefectch schme
觀察這三筆訊號之波形，可以看出將「讀取記憶體」任務增加 pipeline 功能，將能使讀取效率提升，可透過前一比地址資訊預測將會使用到的連續記憶體區域，提前讀取至 Cache 中  
![image](https://hackmd.io/_uploads/rJsAmcAdT.png)  
### Data Prefetch 並存入 Cache
[1]. 在原先的 Controller 沒有 Burst mode ，每次讀取資料都需要一定的訪問延遲外加發送address的時間 ( 1T )，若該資料的地址已經被激活 ( activate ) 則需要 6T ，否則需要 9T 才能得到資料  
![11111](https://hackmd.io/_uploads/SybH450OT.jpg)  
改善: (結果如下圖連續輸出)
1. 若預先將資料抓取，存在 1T 記憶體中，則訪問只會有 2T 延遲。  sent address ( 1T ) → sent data back & ack ( 1T )  
2. Wishbone 輸入 address、rw、in_valid 等資訊後，controller 只會經歷 1T Busy，因此可以連續處理多筆訪問  
3. 另外也新增了cache and burst(會在後面做補充)  
![image](https://hackmd.io/_uploads/rkzFr90_T.png)  

### 分別成difference bank
將Code and data兩種資料分開至不同的 Bank ，即可方便 prefetch 的設計，可以清楚知道哪些 address 存放的是將要 load 到 PE 做運算的資料。  
### Bank 判斷:  
1. WB address 的末 22 bits 會直接斷應到 SDRAM address 
2. 本實驗沒有調整 mapping 方式，各 bits 代表資訊如下表所示  

![image](https://hackmd.io/_uploads/H1cH85A_p.png)  
#### sections.lds
#### 分離前:  
![image](https://hackmd.io/_uploads/B17uL50OT.png)  
Data Fetch:( In Bank 2)  
![image](https://hackmd.io/_uploads/SkiKI5Cd6.png)  
Code Execution(In Bank 0 ~ 3)  
![image](https://hackmd.io/_uploads/Sys9LcC_a.png)  


---

#### 分離後:  
![image](https://hackmd.io/_uploads/HksALqAua.png)  
Data Fetch (In Bank 0)  
![image](https://hackmd.io/_uploads/Hkqfv9AuT.png)  
Code Execution(In Bank 1~3)  
![image](https://hackmd.io/_uploads/HydmP5CO6.png)  
### Ohter SDRAM access conflicts with SDRAM refresh (refresh period)
我們新增了Prefetch的新功能，收到地址後會預先判斷該地址之資料是否為 Code 或是 Data ，若為 Code 資訊，則會用 Burst Mode 連續讀取 8 個連續記憶體資料 （ 已經為本實驗任務特化，省略 program burst length 步驟 ，但是Prefetch 會衍生出 Refresh issue ：
* Refresh cycle：750T
* CAS：3T
* Prefetch data num：8 addresses  

若要連續 prefetch 多筆資料，且不想要被中斷，可能讓 Refresh 時間超出 750 T，並且不是只超出一點點。
> Ex : Time=749 T 時 Prefetch ，則實際 Refresh 時間為 749+24 = 773 T solution：將 Refresh cycle 縮短至 750- 3*8 T = 726 T  



## other - FSM:
> 將原先的 read mode 調整成 Burst read mode ，需要等待 SDRAM Device 處理，所以會經歷 Wait ，但是 burst 模式就能省略這些時間。

### read mode 
訪問地址是已經激活的 Row
`IDLE >ACTIVATE>WAIT>READ>WAIT>ReadRes>IDLE `
訪問地址是尚未經激活的 Row
`IDLE >READ>WAIT>ReadRes>IDLE `

### Burst read mode  

`IDLE >(ACTIVAT)>READ>ReadAndOutput>ReadRes>IDLE `
READ：只進行資料讀取，不輸出資料。
ReadAndOutput：進行資料讀取與輸出資料。
ReadRes：不進行資料讀取，只輸出資料。

## other - SDRAM address 與cache 地址之映射
SDRAM address 與 Cache 地址之映射
```
origin (column)
 DC:Dont care                                  DC
 user_addr                                [ 09 | 08 | 07 | 06 | 05 ]
0x3800_0100 --> 100 --> 0001_0000_0000  -->  0    1    0    0    0  --> 0
0x3800_0120 --> 120 --> 0001_0010_0000  -->  0    1    0    0    1  --> 1
0x3800_0140 --> 140 --> 0001_0100_0000  -->  0    1    0    1    0  --> 2
0x3800_0160 --> 160 --> 0001_0110_0000  -->  0    1    0    1    1  --> 3
0x3800_0180 --> 180 --> 0001_1000_0000  -->  0    1    1    0    0  --> 4
0x3800_01A0 --> 1A0 --> 0001_1010_0000  -->  0    1    1    0    1  --> 5
0x3800_01C0 --> 1C0 --> 0001_1100_0000  -->  0    1    1    1    0  --> 6
0x3800_01E0 --> 1E0 --> 0001_1110_0000  -->  0    1    1    1    1  --> 7
0x3800_0200 --> 200 --> 0010_0000_0000  -->  1    0    0    0    0  --> 8

offset (row)
          [ 4 | 3 | 2 ]
 0x00   --> 0   0   0
 0x04   --> 0   0   1
 0x08   --> 0   1   0
 0x0C   --> 0   1   1
 0x10   --> 1   0   0
 0x14   --> 1   0   1   
 0x18   --> 1   1   0
 0x1C   --> 1   1   1
```  
因為先前已訪問過 0x3800_0120，並將該地址資料存在 Cache中，因此僅需要 2T 就能回傳資料  

![11111](https://hackmd.io/_uploads/HJMjKqR_T.png)  

資料參考- from 厲害的組員  
https://irradiated-hellebore-357.notion.site/LAB-D-SDRAM-4128635388f44d099ccf5abd650cbda2?pvs=25#439c1e3d7d404ddf93b9e37921b017fd













