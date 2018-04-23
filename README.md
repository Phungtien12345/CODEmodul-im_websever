#include <SoftwareSerial.h>
#include <TimerOne.h>
SoftwareSerial ZigBee(2, 3);
SoftwareSerial mySerial(4, 5); // RX, TX
//RECIEVE DATA
uint8_t t_receive,hum_receive,ground_hum_receive,light_receive;
uint8_t interrupt_count=0,statement_1=0,pre_statement_1=0,statement_2=0,pre_statement_2=0;
uint8_t code=0;//PHAN BIET CÁC NODE
char buffer_recieve[80],Buffer_Send[80];
int pos_Send=0,pos_Recieve=0,SEND_SETPOINT_AUTO_1=-1,SEND_SETPOINT_AUTO_2=-1,SEND_STATEMENT_1=-1,SEND_STATEMENT_2=-1;
uint8_t sp_temp_1,sp_temp_2,sp_hum_1,sp_hum_2,sp_light_1,sp_light_2;
uint8_t pre_t1,pre_h1,pre_l1,pre_t2,pre_h2,pre_l2;
void setup() {
  mySerial.begin(9600);
  Serial.begin(9600);
  ZigBee.begin(9600);
   Timer1.initialize(1000000);
   Serial.println("InitGSM....");
    //initGSM();
    Serial.println("InitHTTP....");
    delay(500);
    initHTTP();
    delay(2000);
     Serial.println("READY..!");  
     Timer1.attachInterrupt(Ngat_Data);
}
void Ngat_Data()
{
  interrupt_count++;
        if(interrupt_count==7)
           {
            ZigBee.write("g");
            Serial.println("Yêu cau node 1");
            while(ZigBee.available())
              {
                Receive_data_function();
                break;
              }
          }
         if(interrupt_count==14)
           {
             ZigBee.write("s");
             Serial.println("Yêu cau node 2");
             while(ZigBee.available())
                 {
                    Receive_data_function();
                    break;
                 }
              interrupt_count==0;
           }
         if(code==1)////Recive Data from Node 1
         {
            Serial.println("NODE 1");
            Serial.println(t_receive);
            Serial.println(hum_receive);
            Serial.println(ground_hum_receive);
            Serial.println(light_receive);
            SendtoWeb(0 ,t_receive,hum_receive,ground_hum_receive,light_receive);  
         }
         if(code==2)///Recive data from node 2
         {
            Serial.println("NODE 2");
            Serial.println(t_receive);
            Serial.println(hum_receive);
            Serial.println(ground_hum_receive);
            Serial.println(light_receive);  
            SendtoWeb(1,t_receive,hum_receive,ground_hum_receive,light_receive); 
         }
    code=0;
}
void initHTTP()
{
 mySerial.println("AT+SAPBR=3,1,\"CONTYPE\",\"GPRS\""); //Setup ket noi
 toSerial();
 delay(2000);
 mySerial.println("AT+SAPBR=0,1"); //Closed connect
 toSerial();
 delay(2000);
 mySerial.println("AT+SAPBR=1,1"); //Open connect with baund rate 4800
 toSerial();
 delay(1000);
 mySerial.println("AT+SAPBR=2,1"); //Display IP address
 toSerial();
 delay(2000);
 mySerial.println("AT+HTTPINIT"); //INIT cho HTTP
 toSerial();
delay(3000);
}
/////////////////////////////////////VONG LAP LOOP:
void loop() {
   Recieve_from_Web();
   ///GUI LENH DEN NODE_1
     if((SEND_STATEMENT_1==1)|(SEND_STATEMENT_2==1))
        {
         ZigBee.write("a");
          Send_Statement(statement_1,statement_2);
           Serial.println(statement_1);
           Serial.println(statement_2);
        }
  //Gui sp DEN CAC NODE
    delay(500);
    if(SEND_SETPOINT_AUTO_1==1)
        {
         ZigBee.write("o");
          Send_Setpoint(sp_temp_1,sp_hum_1,sp_light_1);
           Serial.println(sp_temp_1);
           Serial.println(sp_hum_1);
           Serial.println(sp_light_1);
           Serial.println("Da gui SP_2");
        }
   delay(500);
   if(SEND_SETPOINT_AUTO_2==1)
        {
         ZigBee.write("p");
         Send_Setpoint(sp_temp_2,sp_hum_2,sp_light_2);
           Serial.println(sp_temp_2);
           Serial.println(sp_hum_2);
           Serial.println(sp_light_2);
           Serial.println("Da gui SP_2");
        }
}
void toSerial()
{
  while(mySerial.available()!=0)
  {
  Serial.write(mySerial.read());
  }
}
void toSerial_Read()
{
  while(mySerial.available()!=0)
  {
    if ( pos_Send >= sizeof(Buffer_Send) )
    ResetBuffer_Send(); 
    Buffer_Send[pos_Send++]=mySerial.read();
  }
  Serial.println(Buffer_Send);
  ResetBuffer_Send();
}
void toSerial_Recieve()
{
  while(mySerial.available()!=0)
  {
    if ( pos_Recieve >= sizeof(buffer_recieve) )
    ResetBuffer_Recieve(); 
    buffer_recieve[pos_Recieve++]=mySerial.read();
  }
  Serial.println(buffer_recieve);
  GET_STATEMENT();
  Serial.println(statement_1);
  Serial.println(statement_2);
  GET_SETPOINT();
  Serial.println( sp_temp_1);
  Serial.println( sp_hum_1);
   Serial.println( sp_light_1);
  Serial.println( sp_temp_2);
  Serial.println( sp_hum_2);
   Serial.println( sp_light_2);
  ResetBuffer_Recieve();
}
void GET_STATEMENT()
{
  uint8_t temp_light_1,temp_fan_1,temp_pump_1;
  uint8_t temp_light_2,temp_fan_2,temp_pump_2;
  if(buffer_recieve[35]=='1')  temp_light_1=0x01; else temp_light_1=0x00;//LIGHT_1
  if(buffer_recieve[36]=='1')  temp_fan_1=0x02;  else temp_fan_1=0x00;//FAN_1
  if(buffer_recieve[37]=='1')  temp_pump_1=0x04; else temp_pump_1=0x00;//PUMP_1
  if(buffer_recieve[34]=='0')  statement_1=0x00; else statement_1=temp_light_1|temp_fan_1|temp_pump_1;
  if(buffer_recieve[40]=='1')  temp_light_2=0x01; else temp_light_2=0x00;//LIGHT_2
  if(buffer_recieve[41]=='1')  temp_fan_2=0x02;  else temp_fan_2=0x00;//FAN_2
  if(buffer_recieve[42]=='1')  temp_pump_2=0x04; else temp_pump_2=0x00;//PUMP_2
  if(buffer_recieve[39]=='0')  statement_2=0x00; else statement_2=temp_light_2|temp_fan_2|temp_pump_2; 
  if(pre_statement_1==statement_1)
      SEND_STATEMENT_1=-1; else SEND_STATEMENT_1=1;
  if(pre_statement_2==statement_2)
      SEND_STATEMENT_2=-1; else SEND_STATEMENT_2=1;
  pre_statement_1=statement_1;
  pre_statement_2=statement_2;
}
void GET_SETPOINT()
{ 
   sp_temp_1=(buffer_recieve[44]-48)*10+(buffer_recieve[45]-48);
   sp_hum_1=(buffer_recieve[46]-48)*10+(buffer_recieve[47]-48);
   sp_light_1=(buffer_recieve[48]-48)*10+(buffer_recieve[49]-48);
   if((pre_t1== sp_temp_1)&&(pre_h1==sp_hum_1)&&(pre_l1==sp_light_1)) SEND_SETPOINT_AUTO_1=-1;else  SEND_SETPOINT_AUTO_1=1;
   sp_temp_2=(buffer_recieve[51]-48)*10+(buffer_recieve[52]-48);
   sp_hum_2=(buffer_recieve[53]-48)*10+(buffer_recieve[54]-48);
   sp_light_2=(buffer_recieve[55]-48)*10+(buffer_recieve[56]-48);
  if((pre_t2== sp_temp_2)&&(pre_h2==sp_hum_2)&&(pre_l2==sp_light_2)) SEND_SETPOINT_AUTO_2=-1;else  SEND_SETPOINT_AUTO_2=1;
  pre_t1=sp_temp_1;pre_h1=sp_hum_1;pre_l1=sp_light_1;
  pre_t2=sp_temp_2;pre_h2= sp_hum_2;pre_l2=sp_light_2;
}
void ResetBuffer_Send() {
  memset(Buffer_Send, 0, sizeof(Buffer_Send));
  pos_Send = 0;
}
void ResetBuffer_Recieve() {
  memset(buffer_recieve, 0, sizeof(buffer_recieve));
  pos_Recieve = 0;
}
void SendtoWeb(uint8_t a,uint8_t b,uint8_t c,uint8_t d,uint8_t e)
{
//Gui lenh ban dau sau do them nhiet do///  DA GUI XONG.
  mySerial.print("AT+HTTPPARA=URL,smartfarmh.info/add.php?id="+(String)a+"&temp="+(String)b+"&hum="+(String)c+"&light="+(String)d+"&ph="+(String)e+"\r\n");
  delay(100);
  toSerial();
  delay(1000);
  mySerial.println("AT+HTTPACTION=0");
  delay(200);
  toSerial();
  delay(3000);
  toSerial();
  mySerial.println("AT+HTTPREAD=0,60");
  delay(500);
  toSerial_Read();
}
void Recieve_from_Web()/////NHAN DU LIEU TU WEBSERVER
{
///URL gui yeu cau nhan data///Da NHAN XONG
  mySerial.print("AT+HTTPPARA=URL,smartfarmh.info/get_state.php\r\n");
  delay(100);
  toSerial();
  delay(1000);
  mySerial.println("AT+HTTPACTION=0");
  delay(200);
  toSerial();
  delay(3000);
  toSerial();
  mySerial.println("AT+HTTPREAD=0,23");
  delay(500);
  toSerial_Recieve();
}
void Receive_data_function()
{
  uint8_t kt_goi_tin=5*sizeof(t_receive);
  if(ZigBee.available() >= kt_goi_tin)
  {
     code=ZigBee.read();
     delay(50);
    t_receive=ZigBee.read();
     delay(50);
    hum_receive=ZigBee.read();
     delay(50);
    ground_hum_receive=ZigBee.read();
     delay(50);
    light_receive=ZigBee.read();
     delay(50);
  }
}
//////////////////////////////
void Send_Statement(uint8_t a,uint8_t b)
{
  delay(50);
  ZigBee.write(a);
  delay(50);
  ZigBee.write(b);
   delay(50);
   Serial.println("Da gui lenh xong");
}
void Send_Setpoint(uint8_t a,uint8_t b,uint8_t c)
{
  delay(50);
  ZigBee.write(a);
  delay(50);
  ZigBee.write(b);
   delay(50);
  ZigBee.write(c);
  delay(50);
  Serial.println("Da Setpoint xong");
}
