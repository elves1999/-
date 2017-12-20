# -
智能控制加湿器
#include <Adafruit_NeoPixel.h>//include声明函数
#define INTERVAL_SENSOR   10             //定义传感器采样时间间隔  597000
#define INTERVAL_NET      10             //定义发送时间
//传感器部分================================
#include <Servo.h>
Servo myservo;   //调用舵机部分
#include <Wire.h>                                  //调用库  
#include "./ESP8266.h"
#include "I2Cdev.h"                                //调用库  
//温湿度   
#include <SHT2x.h>
//光照
#include <U8glib.h>
#include <IRremote.h>
int RECV_PIN = 10;   //红外线接收器OUTPUT端接在pin 10
IRrecv irrecv(RECV_PIN);  //定义IRrecv对象来接收红外线信号
decode_results results;   //解码结果放在decode_results构造的对象results里
#define servo_pin SDA
#define  sensorPin_1  A0
#define hum 80
#define SSID           "ganlianbabadewifi"                   // cannot be longer than 32 characters!
#define PASSWORD       "sj123456789"

#define IDLE_TIMEOUT_MS  3000      // Amount of time to wait (in milliseconds) with no data 
                                   // received before closing the connection.  If you know the server
                                   // you're accessing is quick to respond, you can reduce this value.

//WEBSITE     
#define HOST_NAME   "api.heclouds.com"
#define DEVICEID   "20494245"
#define PROJECTID "104267"
#define HOST_PORT   (80)
String apiKey="hFzEVx5gx23be8mHZmC4Hvd2=tg=";
char buf[10];

#define INTERVAL_sensor 10
unsigned long sensorlastTime = millis();

float tempOLED, humiOLED, lightnessOLED;

#define INTERVAL_OLED 1000

String mCottenData;
String jsonToSend;

//3,传感器值的设置 
float sensor_tem, sensor_hum, sensor_lux;                    //传感器温度、湿度、光照   
char  sensor_tem_c[7], sensor_hum_c[7], sensor_lux_c[7] ;    //换成char数组传输
#include <SoftwareSerial.h>
SoftwareSerial mySerial(2, 3); /* RX:D3, TX:D2 */
ESP8266 wifi(mySerial);
//ESP8266 wifi(Serial1);                                      //定义一个ESP8266（wifi）的对象
unsigned long net_time1 = millis();                          //数据上传服务器时间
unsigned long sensor_time = millis();                        //传感器采样时间计时器
#define servo_pin D9

//int SensorData;                                   //用于存储传感器数据
String postString;                                //用于存储发送数据的字符串
//String jsonToSend;                                //用于存储发送的json格式参数
#define INTERVAL_LCD   20             //定义OLED刷新时间间隔  
#define PIXEL_COUNT  6
unsigned long lcd_time = millis();  
U8GLIB_SSD1306_128X64 u8g(U8G_I2C_OPT_NONE);     //设置OLED型号  

//-------字体设置，大、中、小
#define setFont_L u8g.setFont(u8g_font_7x13)
#define setFont_M u8g.setFont(u8g_font_fixed_v0r)
#define setFont_S u8g.setFont(u8g_font_fixed_v0r)
#define setFont_SS u8g.setFont(u8g_font_fub25n)
int pose=0 ;
int data;//舵机初始化
void setup()     //初始化函数  
{       
  //初始化串口波特率  
    Wire.begin();
    Serial.begin(9600);
    irrecv.enableIRIn(); // 启动红外解码   
    while(!Serial);
    pinMode(sensorPin_1, INPUT);
    pinMode(RECV_PIN, INPUT);
     myservo.attach(servo_pin);
    //ESP8266初始化
    Serial.print("setup begin\r\n");   
    Serial.print("FW Version:");
    Serial.println(wifi.getVersion().c_str());

  if (wifi.setOprToStationSoftAP()) {
    Serial.print("to station + softap ok\r\n");
  } else {
    Serial.print("to station + softap err\r\n");
  }

  if (wifi.joinAP(SSID, PASSWORD)) {      //加入无线网
    Serial.print("Join AP success\r\n");  
    Serial.print("IP: ");
    Serial.println(wifi.getLocalIP().c_str());
  } else {
    Serial.print("Join AP failure\r\n");
  }

  if (wifi.disableMUX()) {
    Serial.print("single ok\r\n");
  } else {
    Serial.print("single err\r\n");
  }

  Serial.print("setup end\r\n");
     
}
void loop()     //循环函数  
{  
  u8g.firstPage();     //从这开始下面十行是让OLED显示温湿度

    do {

        setFont_S;

        u8g.setPrintPos(4, 32);

        u8g.print("H/%: ");

        u8g.print(SHT2x.readRH());

        u8g.setPrintPos(4, 64);

        u8g.print("T/C: ");

        u8g.print(SHT2x.readT());
    }while( u8g.nextPage() );
  if (sensor_time > millis())  sensor_time = millis();  
    
  if(millis() - sensor_time > INTERVAL_SENSOR)              //传感器采样时间间隔  
  {  
    getSensorData();                                        //读串口中的传感器数据
    sensor_time = millis();
  }  
  delay(100); 
   if (irrecv.decode(&results)) {      //解码成功，收到一组红外线信号
    Serial.println(results.value, HEX);//// 输出红外线解码结果（十六进制）
    delay(15);
     if(results.value==0x1FE807F)
  {
     for(pose=0;pose<180;pose++)
    {
      myservo.write(pose);
      delay(15);
    }
  }
    irrecv.resume(); //  接收下一个值
  }
  delay(100);
   getSensorData();
  if(sensor_hum<=hum)
  {  
   for(pose=180;pose>0;pose--)
    {
      myservo.write(pose);
      delay(15);
    }
  }
  else{ 
    for(pose=0;pose<180;pose++)
    {
      myservo.write(pose);
      delay(15);}
  }
  delay(100);
  if (net_time1 > millis())  net_time1 = millis();
  
  if (millis() - net_time1 > INTERVAL_NET)                  //发送数据时间间隔
  {                
    updateSensorData();                                     //将数据上传到服务器的函数
    net_time1 = millis();
  }
}

void getSensorData(){  
    sensor_tem = SHT2x.readT() ;   
    sensor_hum = SHT2x.readRH();   
    //获取光照
    sensor_lux = analogRead(A0);    
    delay(1000);
    dtostrf(sensor_tem, 2, 1, sensor_tem_c);
    dtostrf(sensor_hum, 2, 1, sensor_hum_c);
    dtostrf(sensor_lux, 3, 1, sensor_lux_c);
}
void updateSensorData() {
  if (wifi.createTCP(HOST_NAME, HOST_PORT)) { //建立TCP连接，如果失败，不能发送该数据
    Serial.print("create tcp ok\r\n");

jsonToSend="{\"Temperature\":";
    dtostrf(sensor_tem,1,2,buf);
    jsonToSend+="\""+String(buf)+"\"";
    jsonToSend+=",\"Humidity\":";
    dtostrf(sensor_hum,1,2,buf);
    jsonToSend+="\""+String(buf)+"\"";
    jsonToSend+=",\"Light\":";
    dtostrf(sensor_lux,1,2,buf);
    jsonToSend+="\""+String(buf)+"\"";
    jsonToSend+="}";



    postString="POST /devices/";
    postString+=DEVICEID;
    postString+="/datapoints?type=3 HTTP/1.1";
    postString+="\r\n";
    postString+="api-key:";
    postString+=apiKey;
    postString+="\r\n";
    postString+="Host:api.heclouds.com\r\n";
    postString+="Connection:close\r\n";
    postString+="Content-Length:";
    postString+=jsonToSend.length();
    postString+="\r\n";
    postString+="\r\n";
    postString+=jsonToSend;
    postString+="\r\n";
    postString+="\r\n";
    postString+="\r\n";

  const char *postArray = postString.c_str();                 //将str转化为char数组
  Serial.println(postArray);
  wifi.send((const uint8_t*)postArray, strlen(postArray));    //send发送命令，参数必须是这两种格式，尤其是(const uint8_t*)
  Serial.println("send success");   
     if (wifi.releaseTCP()) {                                 //释放TCP连接
        Serial.print("release tcp ok\r\n");
        } 
     else {
        Serial.print("release tcp err\r\n");
        }
      postArray = NULL;                                       //清空数组，等待下次传输数据
  
  } else {
    Serial.print("create tcp err\r\n");
  }
  
}
