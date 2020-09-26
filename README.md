# 습도측정 악기 케이스&보관장소

## 프로젝트 목표
- 악기의 철저한 습도 관리를 위한 하드웨어 시스템 구축
- 센서로 측정된 정보 TCP통신을 이용하여 정보 송수신
- 데이터베이스를 활용하여 시기별 습도 관리요령 습득

## 필요성
악기의 대부분은 나무&동물가죽으로 이루어져 있고, 이러한 재질들은 습도에 민감하기 때문에 철저한 관리가 필요하다. 실제로 습도의 미흡한 관리로 악기의 문제가 발생하여 수리를 하거나 새로운 악기로 교체하는 경우가 빈번히 일어나지만 연주자가 항시 습도를 체크하며 악기를 관리하기에는 큰 어려움이 있다. 이러한 문제를 해결하기 위해 악기 케이스&보관장소에 습도 센서로 값을 측정한 후 습도의 정보와 위험정보 등을 연주자에게 송신하므로 연주자가 보다 쉽고 체계적으로 습도를 관리 할 수 있도록 한다.   

## 순서도
![순서도](https://user-images.githubusercontent.com/70748105/94340222-03b20180-003b-11eb-9855-371cf733a086.PNG)

## 구조도
![구조도](https://user-images.githubusercontent.com/70748105/94340218-fbf25d00-003a-11eb-8a1a-c0a1be3fd832.PNG)

## 개발도구
- Arduino MEGA 2560
- 온습도 센서 DHT11
- LCD(16x2,0x27)
- 3색 LED
- VM_WARE 개발환경
- Ubuntu 20.04.4 LTS
- gcc ver 9.3.0
- Arduino sketch IDE
- MYSQL ver 8.0.21

## 기능설명
- 케이스&보관장소에서 아두이노를 활용하여 습도를 측정하고,측정값을 LCD에 표시하고 범위를 판단하여 LCD에 문구 출력과 함께 LED 점등
- 측정과 동시에 서버프로그램으로 정보를 전송  
◎서버프로그램
- 정보를 전달 받으면 습도의 범위를 판단하여 측정값과 현재 상태를 클라이언트 프로그램에 전송(TCP통신)
- 전송과 동시에 데이터베이스에 측정값 저장(MYSQL)
- 서버 프로그램 창에는 실시간 시간과 함께 측정값 출력  
◎클라이언트 프로그램
- 서버에게 위험 정보를 전달 받으면 상황을 인지하고 즉각적으로 대처 

## 시연영상
![아두 시연영상](https://user-images.githubusercontent.com/70748105/94340595-167a0580-003e-11eb-8a42-64fd5d7c12f9.gif)
![프로그램 시연영상](https://user-images.githubusercontent.com/70748105/94340599-1a0d8c80-003e-11eb-99f8-9c99d57b8ca4.gif)



## 기대효과
- 철저한 습도관리로 연주자의 연주력 향상 및 수리&유지보수 측면에서의 경제적 절약효과 기대
- 데이터베이스에 저장된 값들을 분석하여 시기별 습도 관리요령 습득기대
- 습도 관리가 어려운 분야에서도 높은 활용도에 대한 기대


## 프로그램별 소스코드
- 아두이노 소스코드
```c
#include <Arduino.h>

#include "DHT.h"

#include<LiquidCrystal_I2C.h>
#include <Wire.h>
LiquidCrystal_I2C lcd = LiquidCrystal_I2C(0x27, 16, 2);   

const uint8_t TEMPER_SENSOR_PIN = A0;
DHT dht = DHT(TEMPER_SENSOR_PIN, 11);

const uint8_t RED_PIN = 2U;
const uint8_t GREEN_PIN = 3U;
const uint8_t BLUE_PIN = 4U;


void setup() {
  Serial.begin(9600UL);
  dht.begin();

  pinMode(TEMPER_SENSOR_PIN, INPUT);
  pinMode(RED_PIN, OUTPUT);
  pinMode(GREEN_PIN, OUTPUT);
  pinMode(BLUE_PIN, OUTPUT);



  lcd.init();
  lcd.backlight();
  lcd.home();





}


void loop() {

  float temper_data = dht.readTemperature();
  float F_data = dht. convertCtoF(temper_data);
  float humit_data = dht.readHumidity();
  float heat_data = dht.computeHeatIndex(temper_data, humit_data);

  Serial.println((int)humit_data);



  lcd.setCursor(0, 0);
  lcd.print("T:");
  lcd.setCursor(2, 0);
  lcd.print(temper_data);
  lcd.setCursor(7, 0);
  lcd.print(" H:");
  lcd.setCursor(10, 0);
  lcd.print(humit_data);





  if (humit_data > 68)
  {
    RGB_action(255, 0, 0);
    lcd.setCursor(0, 1);
    lcd.print("DANGER(WET) ");

  }
  else if (humit_data > 60)
  {
    RGB_action(255, 0, 255);
    lcd.setCursor(0, 1);
    lcd.print("WARNING     ");

  }

  else if (humit_data <50)
  {
    RGB_action(255, 0, 0);
    lcd.setCursor(0, 1);
    lcd.print("DANGER(DRY)  ");

  }

  else
  {
    RGB_action(0, 255, 0);
    lcd.setCursor(0, 1);
    lcd.print("STABLE     ");
  }





  delay(1000UL);

}

void RGB_action(int RED_LIGHT, int GREEN_LIGHT, int BLUE_LIGHT)
{
  analogWrite(RED_PIN, RED_LIGHT);
  analogWrite(GREEN_PIN, GREEN_LIGHT);
  analogWrite(BLUE_PIN, BLUE_LIGHT);
}
```

- 서버프로그램 소스코드
```c
#include<stdio.h>
#include<stdlib.h>
#include<string.h>
#include<unistd.h>
#include<arpa/inet.h>
#include<sys/socket.h>
#include<sys/types.h>
#include<sys/stat.h>
#include<fcntl.h>
#include<termios.h>
#include<time.h>
#include<mysql/mysql.h>
#include <errno.h>

#define BAUDRATE        B9600
#define MODEMDEVICE     "/dev/ttyACM0"
#define _POSIX_SOURCE   1       // POSIX compliant source

#define FALSE   0
#define TRUE    1
int closeDB(MYSQL *mysql);
int writeDB(MYSQL *mysql,int humid);
int initDB(MYSQL * mysql, const char * host, const char * id, const char * pw, const char * db);
void error_handling(char *message);



int main(int argc,char* argv[])
{
        int serv_sock,clnt_sock;
        char buf[255];
        int hum;

	time_t rawTime;
        struct tm* pTimeInfo;

	
	MYSQL mysql;

	
	if(initDB(&mysql, "localhost","root","bigdata","DHT")<0)
	{
	printf("(!) initDB failed\n");
	return -1;
	}
	else
	printf("(i) initDB successd!\n"); 


	


        int fd = open(MODEMDEVICE, O_RDWR | O_NOCTTY );
        if (fd <0) { perror(MODEMDEVICE); exit(-1); }

        struct termios tio;
        tcgetattr(fd,&tio);
        tio.c_cflag |= BAUDRATE;
        tio.c_cflag |= CS8;
        tio.c_lflag = ICANON;
        tcsetattr(fd,TCSANOW,&tio);

        struct sockaddr_in serv_adr, clnt_adr;
        socklen_t clnt_adr_size;

        char message1[]="DANGER(WET)";
        char message2[]="WARNING";
        char message3[]="DANGER(DRY)";
        char message4[]="STABLE";
        char L_MSG[30];

        if(argc!=2) {
	printf("Using : %s <port>\n",argv[0]);
        exit(1);
        }


        serv_sock=socket(PF_INET,SOCK_STREAM,0);
        if(serv_sock==-1)
        error_handling("socket() error");

        memset(&serv_adr,0,sizeof(serv_adr));
        serv_adr.sin_family=AF_INET;
        serv_adr.sin_addr.s_addr=htonl(INADDR_ANY);
        serv_adr.sin_port=htons(atoi(argv[1]));

        if(bind(serv_sock,(struct sockaddr*)&serv_adr,sizeof(serv_adr))==-1)
                error_handling("bind () error");

        if(listen(serv_sock,1)==-1)
        error_handling("listen() error");

        clnt_adr_size=sizeof(clnt_adr);


         clnt_sock=accept(serv_sock,(struct sockaddr *)&clnt_adr,&clnt_adr_size);
                  if(clnt_sock==-1)
                 error_handling("accept() error");



        while(1)
        {
		rawTime = time(NULL);
		pTimeInfo = localtime(&rawTime);


   		int year = pTimeInfo->tm_year + 1900;
 	 	int month = pTimeInfo->tm_mon + 1;
    		int day = pTimeInfo->tm_mday;
		int hour = pTimeInfo->tm_hour;
    		int min = pTimeInfo->tm_min;
    		int sec = pTimeInfo->tm_sec;

                int res = read(fd,buf,255);
                hum = atoi(buf);

		if(res<=0)
			break;

		if(res>2)
		writeDB(&mysql,hum);


                if(hum>68 && res>2)
                {       sprintf(L_MSG,"%s \nhumi  : %d",message1,hum);
                        write(clnt_sock,L_MSG,sizeof(L_MSG));
                        printf("timeInfo : %d/%d/%d %d:%d:%d\n", year, month, day, hour, min, sec);
                        printf("HUMI : %d\n\n",hum);
			
                }
                else if(hum>60 && res>2)
                {
                        sprintf(L_MSG,"%s \nhumi  : %d",message2,hum);
                        write(clnt_sock,L_MSG,sizeof(L_MSG));
                        printf("timeInfo : %d/%d/%d %d:%d:%d\n", year, month, day, hour, min, sec);
                        printf("HUMI : %d\n\n",hum);
                }

		else if(hum<50 && res>2)
                {
                        sprintf(L_MSG,"%s \nhumi  : %d",message3,hum);
                        write(clnt_sock,L_MSG,sizeof(L_MSG));
                        printf("timeInfo : %d/%d/%d %d:%d:%d\n", year, month, day, hour, min, sec);
                        printf("HUMI : %d\n\n",hum);
                }

                else if(res>2)
                 {
                        sprintf(L_MSG,"%s \nhumi  : %d",message4,hum);
                        write(clnt_sock,L_MSG,sizeof(L_MSG));
                        printf("timeInfo : %d/%d/%d %d:%d:%d\n", year, month, day, hour, min, sec);
                        printf("HUMI : %d\n\n",hum);
                }
                memset(L_MSG,0,sizeof(L_MSG));
	

        }
        close(clnt_sock);
        close(serv_sock);
        close(fd);

	if(closeDB(&mysql)<0) 
	{
	printf("(!) clseDB failed\n"); 
	return -1;
	}
	else
	printf("(i) closeDB success\n");
        return 0;


}



void error_handling(char *message)
{
   fputs(message,stderr);
   fputc('\n',stderr);
   exit(1);
}

int writeDB(MYSQL * mysql, int humid)
{
	char strQuery[255]="";
	sprintf(strQuery, "INSERT INTO DHT_DATA(HUMI, DATE) values(%d, now())", humid);
	int res = mysql_query(mysql, strQuery); 
	if (!res) 
	{
	}
	else 
	{
		fprintf(stderr, "(!) insert error %d : %s\n", mysql_errno(mysql),
		mysql_error(mysql));
	return -1;
	}
	return 0;
}

int closeDB(MYSQL * mysql)
{
	mysql_close(mysql); 
	return 1;
}


int initDB(MYSQL * mysql, const char * host, const char * id, const char * pw, const char * db)
{
	printf("(i) initDB called, host=%s, id=%s, pw=%s, db=%s\n", host, id, pw, db);
	mysql_init(mysql); 
	if(mysql_real_connect(mysql, host, id, pw, db,0,NULL,0)) 
	{	
		printf("(i) mysql_real_connect success\n");
		return 0; 
	}
	printf("(!) mysql_real_connect failed\n");
	return -1; 
}
```
- 서버 프로그램 소스코드
```c
#include<stdio.h>
#include<stdlib.h>
#include<string.h>
#include<unistd.h>
#include<arpa/inet.h>
#include<sys/socket.h>

void error_handling(char *message);

int main(int argc, char *argv[])
{
	int sock;
	char message[30];
	struct sockaddr_in serv_addr;
	int str_len;

	if(argc!=3) {
        printf("Using : %s <port>\n",argv[0]);
        exit;
         }

	sock=socket(PF_INET,SOCK_STREAM,0);
        if(sock==-1)
        error_handling("socket() error");

        memset(&serv_addr,0,sizeof(serv_addr));
        serv_addr.sin_family=AF_INET;
        serv_addr.sin_addr.s_addr=inet_addr(argv[1]);
        serv_addr.sin_port=htons(atoi(argv[2]));

	if(connect(sock, (struct sockaddr*)&serv_addr,sizeof(serv_addr))==-1)
        error_handling("bind() error");



	while(1)
	{
		str_len=read(sock,message,sizeof(message)-1);
		if(str_len==-1)
			error_handling("read() error!");
		
                
  
		if(str_len>3)
			printf("state : %s\n\n",message);
			
		
		
	}

	close(sock);
	return 0;

}

void error_handling(char *message)
{
   fputs(message,stderr);
   fputc('\n',stderr);
   exit(1);
}
```
