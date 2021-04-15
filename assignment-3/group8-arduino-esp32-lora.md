# ESP32+GX18B20+LORA搭建远程测温系统

## 准备

### 材料

- ESP32单片机x2（NodeMCU32）
- Lora模块x2 (AI-Thinker Ra-01/02)
- GX18B20温度传感器
- 杜邦线+引脚若干及若干MicroUSB数据线

### Arduino开发环境

安装Arduino+ESP32开发板管理器，从库中安装`LoraNow`、`DallasTemperature`

详见assignment2

![Snipaste_2021-04-15_19-12-15.png](https://xqhma.oss-cn-hangzhou.aliyuncs.com/image/Snipaste_2021-04-15_19-12-15.png)

## 焊接

焊接引脚及LORA贴片，成品如下

![IMG_20210415_182224.jpg](https://xqhma.oss-cn-hangzhou.aliyuncs.com/image/IMG_20210415_182224.jpg)

## 连接

对于网关，按照下图连接

![24b95ba4-969b-45b7-8277-8479c4371198.png](https://xqhma.oss-cn-hangzhou.aliyuncs.com/image/24b95ba4-969b-45b7-8277-8479c4371198.png)

对于节点，按照下图连接

![Snipaste_2021-04-15_19-04-32.png](https://xqhma.oss-cn-hangzhou.aliyuncs.com/image/Snipaste_2021-04-15_19-04-32.png)

成品：

![IMG_20210415_190602](https://user-images.githubusercontent.com/37738631/114878337-bbb9c700-9e32-11eb-8630-71047da7cf17.jpg)

![IMG_20210415_190607](https://user-images.githubusercontent.com/37738631/114878356-bfe5e480-9e32-11eb-9240-cbac568f92d8.jpg)

## 上传代码

**网关代码：**

```cpp
#include <LoRaNow.h>
#include <WiFi.h>
#include <WebServer.h>
#include <StreamString.h>

#define MISO 19
#define MOSI 23
#define SCK 18
#define SS 5

#define DIO0 4

const char *ssid = "1029";
const char *password = "qwer1234";

WebServer server(80);

const char *script = "<script>function loop() {var resp = GET_NOW('loranow'); var area = document.getElementById('area').value; document.getElementById('area').value = area + resp; setTimeout('loop()', 1000);} function GET_NOW(get) { var xmlhttp; if (window.XMLHttpRequest) xmlhttp = new XMLHttpRequest(); else xmlhttp = new ActiveXObject('Microsoft.XMLHTTP'); xmlhttp.open('GET', get, false); xmlhttp.send(); return xmlhttp.responseText; }</script>";

void handleRoot()
{
  String str = "";
  str += "<html>";
  str += "<head>";
  str += "<title>ESP32 - LoRaNow</title>";
  str += "<meta name='viewport' content='width=device-width, initial-scale=1'>";
  str += script;
  str += "</head>";
  str += "<body onload='loop()'>";
  str += "<center>";
  str += "<textarea id='area' style='width:800px; height:400px;'></textarea>";
  str += "</center>";
  str += "</body>";
  str += "</html>";
  server.send(200, "text/html", str);
}

static StreamString string;

void handleLoRaNow()
{
  server.send(200, "text/plain", string);
  while (string.available()) // clear
  {
    string.read();
  }
}

void setup(void)
{

  Serial.begin(115200);

  WiFi.mode(WIFI_STA);
  if (ssid != "")
    WiFi.begin(ssid, password);
  WiFi.begin();
  Serial.println("");

  // Wait for connection
  while (WiFi.status() != WL_CONNECTED)
  {
    delay(500);
    Serial.print(".");
  }

  Serial.println("");
  Serial.print("Connected to ");
  Serial.println(ssid);
  Serial.print("IP address: ");
  Serial.println(WiFi.localIP());

  server.on("/", handleRoot);
  server.on("/loranow", handleLoRaNow);
  server.begin();
  Serial.println("HTTP server started");

  LoRaNow.setFrequencyCN(); // Select the frequency 486.5 MHz - Used in China
  // LoRaNow.setFrequencyEU(); // Select the frequency 868.3 MHz - Used in Europe
  // LoRaNow.setFrequencyUS(); // Select the frequency 904.1 MHz - Used in USA, Canada and South America
  // LoRaNow.setFrequencyAU(); // Select the frequency 917.0 MHz - Used in Australia, Brazil and Chile

  // LoRaNow.setFrequency(frequency);
  // LoRaNow.setSpreadingFactor(sf);
  // LoRaNow.setPins(ss, dio0);

  LoRaNow.setPinsSPI(SCK, MISO, MOSI, SS, DIO0); // Only works with ESP32

  while (!LoRaNow.begin())
  {
    Serial.println("LoRa init failed. Check your connections.");
    delay(5000);
  }

  LoRaNow.onMessage(onMessage);
  LoRaNow.gateway();
}

void loop(void)
{
  LoRaNow.loop();
  server.handleClient();
}

void onMessage(uint8_t *buffer, size_t size)
{
  unsigned long id = LoRaNow.id();
  byte count = LoRaNow.count();

  Serial.print("Node Id: ");
  Serial.println(id, HEX);
  Serial.print("Count: ");
  Serial.println(count);
  Serial.print("Message: ");
  Serial.write(buffer, size);
  Serial.println();
  Serial.println();

  if (string.available() > 512)
  {
    while (string.available())
    {
      string.read();
    }
  }

  string.print("Node Id: ");
  string.println(id, HEX);
  string.print("Count: ");
  string.println(count);
  string.print("Message: ");
  string.write(buffer, size);
  string.println();
  string.println();

  // Send data to the node
  LoRaNow.clear();
  LoRaNow.print("LoRaNow Gateway Message ");
  LoRaNow.print(millis());
  LoRaNow.send();
}
```

**节点代码：**

```cpp
#include <LoRaNow.h>

//vspi for lora radio module
#define MISO 19
#define MOSI 23
#define SCK 18
#define SS 5

#define DIO0 4

#include <OneWire.h>
#include <DallasTemperature.h>

// GPIO where the DS18B20 is connected to
const int oneWireBus = 25;     

// Setup a oneWire instance to communicate with any OneWire devices
OneWire oneWire(oneWireBus);

// Pass our oneWire reference to Dallas Temperature sensor 
DallasTemperature sensors(&oneWire);

float tmp;

void setup() {
  Serial.begin(115200);
  Serial.println("LoRaNow Simple Node");
  sensors.begin();

   LoRaNow.setFrequencyCN(); // Select the frequency 486.5 MHz - Used in China
  // LoRaNow.setFrequencyEU(); // Select the frequency 868.3 MHz - Used in Europe
  // LoRaNow.setFrequencyUS(); // Select the frequency 904.1 MHz - Used in USA, Canada and South America
  // LoRaNow.setFrequencyAU(); // Select the frequency 917.0 MHz - Used in Australia, Brazil and Chile

  // LoRaNow.setFrequency(frequency);
  // LoRaNow.setSpreadingFactor(sf);
  // LoRaNow.setPins(ss, dio0);

   LoRaNow.setPinsSPI(SCK, MISO, MOSI, SS, DIO0); // Only works with ESP32

  if (!LoRaNow.begin()) {
    Serial.println("LoRa init failed. Check your connections.");
    while (true);
  }

  LoRaNow.onMessage(onMessage);
  LoRaNow.onSleep(onSleep);
  LoRaNow.showStatus(Serial);
}

void loop() {
  sensors.requestTemperatures(); 
  tmp = sensors.getTempCByIndex(0);
  LoRaNow.loop();
}


void onMessage(uint8_t *buffer, size_t size)
{
  Serial.print("Receive Message: ");
  Serial.write(buffer, size);
  Serial.println();
  Serial.println();
}

void onSleep()
{
  Serial.println("Sleep");
  Serial.println(tmp);
  delay(5000); // "kind of a sleep"
  Serial.println("Send Message");
  LoRaNow.print("LoRaNow Node Message ");
  //Serial.println("LoRaNow Message sended");
  LoRaNow.print(millis());
  LoRaNow.print("-");
  LoRaNow.print(tmp);
  LoRaNow.send();
}
```

## 运行测试

![Snipaste_2021-04-15_19-08-34.png](https://xqhma.oss-cn-hangzhou.aliyuncs.com/image/Snipaste_2021-04-15_19-08-34.png)

![Snipaste_2021-04-15_19-08-58.png](https://xqhma.oss-cn-hangzhou.aliyuncs.com/image/Snipaste_2021-04-15_19-08-58.png)

## 问题与反思

注：环境搭建的问题见assignment2

### 1. 引脚定义问题

在最开始的时候，因为没有仔细查看传感器型号及引脚定义，导致VDD和GND反接，使温度传感器严重过热。

### 2. 数据手册不完整

本次实验中使用的传感器型号GX18B20的数据手册并不完整，未提到上拉电阻一事。最终通过咨询同学并翻看DX18B20的数据手册后解决。
