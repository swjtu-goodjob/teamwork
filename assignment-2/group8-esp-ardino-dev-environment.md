# 一. Arduino+ESP32开发环境搭建及测试

## 前言

初次接触单片机，所以基本都是跟着网上的教程边学边做的。如有失误，欢迎指正。

- **地点：** 宿舍
- **时间：** 2020学年第2学期第四周
- **人员：** 第8组全体组员
- **目的：** 在本地基于Arduino安装ESP32开发环境并进行简单测试。

## 过程

### 1. Arduino环境安装

从[Arduino官网](https://www.arduino.cc/en/software)下载并安装Arduino IDE（本次使用版本1.8.13）

![arduino-1.png](https://xqhma.oss-cn-hangzhou.aliyuncs.com/image/arduino-1.png)

### 2. 在Arduino中安装ESP32开发环境

首先在首选项中添加ESP开发板管理器仓库

地址：`https://raw.githubusercontent.com/espressif/arduino-esp32/gh-pages/package_esp32_index.json`

![Snipaste_2021-03-28_18-00-46.png](https://xqhma.oss-cn-hangzhou.aliyuncs.com/image/Snipaste_2021-03-28_18-00-46.png)

注：因网络原因，在本次实验中设置了代理进行访问，代理在首选项-网络中进行设置。

添加完ESP32仓库后，在工具-开发板-开发板管理器中搜索esp32并安装

![Snipaste_2021-03-28_18-03-07.png](https://xqhma.oss-cn-hangzhou.aliyuncs.com/image/Snipaste_2021-03-28_18-03-07.png)

### 3. 配置参数并与ESP32开发板连接

在工具一栏中将开发板选择至`ESP32 Dev Module`，查阅ESP32模组（本次为Nodemcu\-32）的datasheet知默认串口速率为115200bps，因此将工具栏中的`Upload Speed`调整至`115200`，保持其余参数默认。

![Snipaste_2021-03-28_18-06-54.png](https://xqhma.oss-cn-hangzhou.aliyuncs.com/image/Snipaste_2021-03-28_18-06-54.png)

通过USB连接单片机，同时在工具-端口一栏中选择正确的连接端口。随后打开Arduino的串口监视器，将波特率调至`115200bps`，按下Nodemcu-32模块上的EN使能按钮，可从串口监视器中观察到单片机系统启动时的返回信息，至此单片机与Arduino连接成功，开发环境搭建完成。

![Snipaste_2021-03-28_18-19-44.png](https://xqhma.oss-cn-hangzhou.aliyuncs.com/image/Snipaste_2021-03-28_18-19-44.png)

### 4. 编译、上传测试程序

作为一个有梦想的小组，我们继续对ESP32单片机进行了一定的探索。首先使用Arduino写一个简单的程序并上传测试。这里遇到了一个小问题，在后文中会提到。

第一次测试代码：

```c
void setup() {
  Serial.begin(115200);
  Serial.println("Setup");
}

void loop() {
  Serial.println("SWJTU-GoodJob");
  delay(5000);
}
```

在Arduino中输入代码后上传，随后在串口监视器中监听

![Snipaste_2021-03-28_18-33-03.png](https://xqhma.oss-cn-hangzhou.aliyuncs.com/image/Snipaste_2021-03-28_18-33-03.png)

第一次测试成功。

### 5. 连接OLED屏（I2C）

第二次折腾开始了。下发的工具包中有一块OLED屏幕，通过网上搜索可知这块0.91寸的屏幕使用的是SSD1306驱动，首先通过工具-管理库进入到库管理器，搜索关键词ESP32和SSD1306，找到对应驱动并安装

![Snipaste_2021-03-28_21-51-05.png](https://xqhma.oss-cn-hangzhou.aliyuncs.com/image/Snipaste_2021-03-28_21-51-05.png)

参考此模块的[官方文档](https://github.com/ThingPulse/esp8266-oled-ssd1306)编写代码

```c
#include <Wire.h>
#include "SSD1306Wire.h"
#include "MyFont.h"
SSD1306Wire  display(0x3c, SDA, SCL, GEOMETRY_128_32);
void setup()
{
  Serial.begin(115200); 
  Serial.println("Start");
  display.init();
  display.displayOn();
  display.setFont(Satisfy_Regular_18);
  display.drawString(0, 0, "SWJTU-GoodJob");
  display.display();
}

void loop()
{
}
```

注：`MyFont.h`中的字体通过[生成器](http://oleddisplay.squix.ch/)生成

![B6C3C0C08D3FBA56C54BC824A95AB32D.jpg](https://xqhma.oss-cn-hangzhou.aliyuncs.com/image/B6C3C0C08D3FBA56C54BC824A95AB32D.jpg)

## 遇到的问题及解决方案

### 1. 网络连接不通畅

镜像/代理解决

### 2. 默认串口速率问题

在初次接上单片机时，串口监视器中获取到的均为乱码，同时上传时也经常失败。后来通过查阅资料得知应该是连接参数设置的不对，于是开始对照Nodemcu-32的文档，随即发现了各项参数设置。

### 3. 上传中卡在Connecting超时

这是后面几次尝试中遇到的问题，刚开始的几次上传是正常的，后来多次上传后发现上传总是卡在Connecting过程，一段时间后超时，在网上翻了许久，后来在针脚定义中看到了这个。

![Snipaste_2021-03-28_22-30-59.png](https://xqhma.oss-cn-hangzhou.aliyuncs.com/image/Snipaste_2021-03-28_22-30-59.png)

于是猜测是模式的问题，翻看原理图发现IO0按钮对应的即GPIO0针脚，上传时连接过程中按下单片机上的IO0按钮便能够解决问题。

### 4. OLED显示屏I2C针脚定义问题

针脚定义里没有显示屏控制需要的SDA/SCL定义，但看文档是支持I2C的，后来在一篇[博客](https://blog.csdn.net/quangui666/article/details/81483645)中找到了答案，SDA与SCL默认对应IO21/IO22针脚