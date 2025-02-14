# 流水灯

> by NginW

## 写在前面

实验完成之，为了能够让大家更好的理解实验中各个模块的实现细节，也是为了现在依旧没有验收的同学能够完成验收，助教会给实验写个简单的文档，对实验做一个简单的分析。如有错漏之处，欢迎同学们在评论区指正。写完代码之后，点击rtl仿真，得到rtl图如下图所示：

![image-20211123224510884](../pic.asset/image-20211123224510884.png)



## 模块的功能

第一个流水灯实验中，用到了三个模块，led控制模块`led_ctl`，计数器模块`counter`和顶层模块`flash_led_top`,`led_ctl`和`counter`作为具体的功能模块,能够硬件功能,`flash_led_top`模块作为电路设计的顶层,完成模块的连接,对外暴露引脚端口的功能.

### flash_led_top

`flash_led_top`模块的端口定义如下,`flash_led_top`有三个外部输入端口`input`和一个16位宽的输出端口`led`,这些端口就代表我们设计的电路和外部实际的硬件资源连接的引脚.说明`flash_led_top`模块接收来自外部的三个输入,输出一个16位宽度的信号

```verilog
module flash_led_top(
    input  wire        clk,
    input  wire        rst_n,
    input  wire        sw0,
    output wire [15:0] led
);
```

然后,在top内部,分别对之前编写完成的两个模块进行了一次实例化,并提供了相应的输入,输出驱动信号.同时,观察可以得出,`counter`的输出信号`clk_bps`同时又作为了`led_ctl`的输入信号.这表明了`flash_led_top`模块的另外一个作用,指定信号,将各个独立的模块,通过信号连接起来,形成一个整体.

在定义各个模块的时候,实际上只是定义了模块的输入,输出方向,数据宽度和类型,并没有指明具体的信号从何而来(类似于c++中的函数概念,但又不同).顶层模块`flash_led_top`的作用,就是提供模块所需的具体信号驱动,达到想要实现的功能.

```verilog
module flash_led_top(
    input  wire        clk,
    input  wire        rst_n,
    input  wire        sw0,
    output wire [15:0] led
);
    wire clk_bps;
    wire rst;

    assign rst = ~rst_n;
    
    counter counter(
        .clk     ( clk     ),
        .rst     ( rst     ),
        .clk_bps ( clk_bps )
    );

    flash_led_ctl flash_led_ctl(
        .clk     ( clk     ),
        .rst     ( rst     ),
        .dir     ( sw0     ),
        .clk_bps ( clk_bps ),
        .led     ( led     )
    );
endmodule
```

### counter

`counter`是计数器模块,因为使用的开发板上的时钟频率太高,如果直接使用时钟的频率,由于*视觉暂留*效应,会让led看上去全部点亮,无法实现流水灯的效果.所以需要分频,得到一个适当的频率.

`counter`模块的功能非常简单,它设置了两个计数器`first`和`second`.当开发板上的时钟上升沿时,触发计数器,如果`first`小于$10000$,则将`first`加一,等于$10000$则清零.

第二个计数器依赖于第一个计数器,它加一条件,除了本身`second`小于$10000$以外,还需要`first`等于$10000$,才会加一,因为`first`加一之后会在下一个时钟周期清零,所以`second`两次加一之间,间隔为`first`从0加到$10000$需要的时间.

最后就是输出信号`clk_bps`,使用了一个三元表达式-判断`clk_bps`的输出,如果`second`等于$10000$,则输出1, 否则输出0

`counter`模块将两个$10000$计数器级联,同时采用同步清零法,在计数器达到$10000$之后,下一个时间周期才会清零.所以单个计数器会经历$0\sim 10000$一共$10001$个状态,采用了两个计数器级联的方式,所以两次输出`clk_bps`之间,会间隔$10001\times 10001$个时钟周期

```verilog
module counter(
  input  wire clk,
  input  wire rst,
  output wire clk_bps
);
  reg [13:0] counter_first,counter_second;

  always @(posedge clk or posedge rst) begin
    if (rst) 
        counter_first <= 14'b0;
    else 
        if (counter_first == 14'd10000) 
            counter_first <= 14'b0;
        else 
            counter_first <= counter_first +1;
  end

  always @(posedge clk or posedge rst) begin
    if (rst) 
        counter_second <= 14'b0;
    else        
        if (counter_second == 14'b10000) 
            counter_second <= 14'b0;
        else 
            if (counter_first == 14'b10000) 
            counter_second <= counter_second + 1;
  end

  assign clk_bps = counter_second == 14'b10000;

endmodule
```

### led_ctl

一位是一个流水灯的控制信号,有16个流水灯需要控制,所以输出一共有16位信号,每一位通过管脚约束,和一个led灯绑定.当某一位为1,led点亮,否则熄灭.通过按键开关`dir`控制流水灯的方向.

因为16位寄存器类型数据`led`每一位都和led绑定,所以实质上流水灯控制就是对`led`数据的控制.当点亮的led不在边界时,将数据左移或者右移一位,当遇到边界时,依据运动的方向进行置位(1或者0x8000)

```verilog
module flash_led_ctl(
    input  wire        clk,
    input  wire        rst,
    input  wire        dir,
    input  wire        clk_bps,
    output reg  [15:0] led
);
    always @( posedge clk or posedge rst )
        if ( rst )
            led <= 16'h8000;
        else
            case ( dir )
                1'b0:               //从左向右
                    if ( clk_bps )
                        if ( led != 16'd1 )
                            led <= led >> 1'b1;
                        else
                            led <= 16'h8000;
                1'b1:               //从右向左
                    if ( clk_bps )
                        if ( led != 16'h8000 )
                            led <= led << 1'b1;
                        else
                            led <= 16'd1;
            endcase
endmodule

```


