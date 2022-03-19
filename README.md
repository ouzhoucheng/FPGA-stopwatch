# FPGA-stopwatch
a simple stopwatch made by FPGA EGO1 board

用ego1板子做一个时钟,可以通过按键更改时间3,可以自动进位.

[【FPGA】EGO1做一个时钟](https://www.bilibili.com/video/BV1JP4y17797/)

# 文件关系

> 文件
> - `Key.v`: 顶层文件,负责按键检测和计时
> - `number.v`: 模块文件,负责数码管显示
> - `led.v`: 模块文件,负责led闪烁控制

# 按键检测与计时

按下到抬起时会有一个上升沿信号,我们用1us的周期检测这个信号,得到`key_rise_edge`
```verilog
always @(posedge clk) begin
    if(keycnt>=20'd999_999)
        begin
        keycnt <= 0;
        key_vc <= btn;
        end
    else    keycnt<=keycnt+20'd1;
end

always @(posedge clk) begin
    key_vp <= key_vc;
end

wire [4:0]key_rise_edge;
assign key_rise_edge = (~key_vp[4:0])&key_vc[4:0];
```

我们的时钟有两个状态,计时模式和修改模式,用寄存器`integer statue`记录,当中间键按下时切换模式
```verilog
parameter Sm = 5'b00100;   //中间
always~
    if(key_rise_edge==Sm) begin
        statue=~statue;
        if(!statue)led1[5:0]=6'b111111;
        else led1[5:0]=6'b000001;
    end
```


`statue=0`时是计时模式
```verilog
// 计时进位
always~
    if(statue==0 && timer_cnt>=100_000_000) begin
        timer_cnt<=0;
        if(seconds>=59) begin
            seconds=0;
            if(minutes>=59) begin
                minutes=0;
                if(hours>=23) begin
                    hours=0; end
                else    hours=hours+1; end
            else    minutes=minutes+1; end
        else    seconds=seconds+1; end
    else    timer_cnt <= timer_cnt+1;
```

`statue=1`时是修改模式.
```verilog
always~
    if(statue) begin
        case(key_rise_edge)
            Sl: // 当前选择位左移
            Sr: // 当前选择位右移
            Su: // 当前选择位增加
            Sd: // 当前选择位减小
        end
```

# 数码管显示
将时分秒赋到`data`寄存器,再连接`number.v`文件
```verilog
reg [31:0] data;
always @(seconds or minutes or hours) begin
    data[31:28] = (hours)/10;
    data[27:24] = (hours)%10;
    data[23:20] = 04'hf;
    data[19:16] = (minutes)/10;
    data[15:12] = (minutes)%10;
    data[11:8]  = 04'hf;
    data[7:4]   = (seconds)/10;
    data[3:0]   = (seconds)%10;
end
number u1(  .clk(clk), // 总时钟
            .rst(rst), // 复位信号
            .data(data), // 待显示数字
            .seg_data(seg_data), // 左边四个数字
            .seg_data2(seg_data2), // 右边四个数字
            .seg_cs(seg_cs) // 显示位
        );
```

由于数码管较多,这里用了分时复用,只要公共端控制信号的刷新速度足够快，人眼就分辨不出LED的闪烁，看起来数码管时同时点亮的.

先产生一个500Hz的信号
```verilog
always @(posedge clk) begin
    if(clk_cnt>=100_000) begin
        clk_cnt<=0;
        clk_500hz<=~clk_500hz; end
    else begin
        clk_cnt<=clk_cnt+1; end
end
```

显示位从右往左循环刷新
```verilog
always @(posedge clk_500hz) begin
    seg_cs<={seg_cs[6:0],seg_cs[7]};
    end

reg[7:0]dis_data;
always @(seg_cs)  begin
    case(seg_cs)
    8'b00000001:dis_data=data[3:0];
    8'b00000010:dis_data=data[7:4];
    ...
    default:dis_data=4'hf;
    endcase end
```

再定义和幅值数字对应的段码即可
```verilog
always @(dis_data ) begin
    case(dis_data )
    04'h0: seg_data = 8'h3F;
    04'h1: seg_data = 8'h06;
    ...
    default: seg_data = 8'h40;
    endcase end
```

# 文件下载
关注公众号**小电动车**下载工程文件.

![请添加图片描述](https://img-blog.csdnimg.cn/a2c04f6f9ed44c3bb0e27e6b844b24d8.png)

