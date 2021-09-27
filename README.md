#include <ESP8266WiFi.h>
#include "FirebaseArduino.h" // 파이어베이스 라이브러리

#define FIREBASE_HOST "파이어베이스 주소"
#define FIREBASE_AUTH "비밀번호"
#define WIFI_SSID "와이파이 이름"
#define WIFI_PASSWORD "비밀번호"


#include "HX711.h" //HX711로드셀 라이브러리 불러오기
#define calibration_factor -7050.0 // 로드셀 초기값을 설정해줍니다. 이렇게 해주어야 처음에 작동시에 0점을 맞추는 것이라고 생각하시면 됩니다.
#define DOUT  3 //엠프 데이터 아웃 핀 
#define CLK  2  //엠프 클락  
HX711 scale(DOUT, CLK); //엠프 핀 선언 
int weight;
int flag = 1;
int irPin1 = 7, irPin2 = 6; //ir핀 핀번
boolean state = true;
int Time = 0;
boolean reset = false;
boolean pakage = false;
int pakageCount = 0;
int preWeight = 0;
int checkPoint = 0;
boolean comeOut = false;
boolean comeIn = false;
boolean checkIr1 = false;
boolean checkIr2 = false;
int buttonClick = 0;



void setup() {
  Serial.begin(115200);

  // connect to wifi.
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
  Serial.print("connecting");
  while (WiFi.status() != WL_CONNECTED) {
    Serial.print(".");
    delay(500);
  }
  Serial.println();
  Serial.print("connected: ");
  Serial.println(WiFi.localIP());
  
  Firebase.begin(FIREBASE_HOST, FIREBASE_AUTH);
  Firebase.reconnectWiFi(true);

  Serial.println("Weight detection start");  
  scale.set_scale(calibration_factor);  
  scale.tare();
  pinMode(irPin2, INPUT);
  pinMode(irPin1, INPUT);
}

void loop() {
  Serial.println(); 
  weight = abs(scale.get_units()*0.045359237); //로드셀로 받은 값을 weight라는 변수에 저
  Serial.print(weight); //
  delay(500);



//택배기사가 1번 센서를 지난다.
  if(flag == 0 && digitalRead(irPin1) && state) {
    flag = 1;
    Time = millis() / 1000;
    checkPoint = 0;
  }


// 택배기사가 택배함에 택배를 올려 놓는다.
  if(flag == 1 && changeWeight(weight, preWeight) && state) {
    flag = 2;
    pakageCount++;
    reset = false;
    state = false;
    preWeight = weight;
    Serial.print("pakageCount: ");
    Serial.println(pakageCount);

    if(checkPoint == 0) {
      if(weight > 0) {
        Serial.print("arrival");
        checkPoint = 1;
      }
    }
  }

// 택배기사가 다시 센서 1번을 지나서 나간다.
  if(flag == 2 && !state && digitalRead(irPin1)) {
    state = true;
    delay(1000);
    flag = 0;
    pakage = true;

    //이때 푸쉬알림
  }




///////// 1번 센서를 지나고 20초후에 초기화된다.///////////////
  if(Time == 20) {
  reset = true;
  }

  switch(reset) {
    case true:
      flag = 0;
      state = true;
      pakage = false;
      Time = 0;
      break;
  }
/////////////////////////////////////////////////////////



  
  

//1-1 집에 있는 내가 택배함에 택배를 갖고 집으로 들어간다. ////////////////////////
  if(digitalRead(irPin2)) {
    checkIr2 = true;
  }
  if(checkIr2 && pakage == true && !changeWeight(weight, preWeight)) {
    flag = 0;
    pakage = false;
    pakageCount = 0;
    Serial.print("GoodTake");
    Firebase.setString("getPakage: ","GoodTake");//파어베이스에 데이터 전송 
       if (Firebase.failed()) { //firebase 오류처리 
           Serial.print("setting /message failed:");
           Serial.println(Firebase.error());  
           return;
          }//이값은 파이어베이스로 보내서 안드로이드에서 받는다.
  }
}


////////////////////////////////////////////////////////////////






//1-3. 밖에 있는 내가 택배함에 택배를 갖고 집으로 들어간다./////////////////////

  if(digitalRead(irPin1)) {
    checkIr1 = true;
  }
  if(checkIr1 && digitalRead(irPin2)) {
    comeIn = true;
  }
  
//본인수령 
  if(checkIr1 && pakage == true && !changeWeight(weight, preWeight)) {
    flag = 0;
    pakage = false;
    pakageCount = 0;
    Serial.print("GoodTake")
    Firebase.setString("getPakage: ","GoodTake");//파어베이스에 데이터 전송 
       if (Firebase.failed()) { //firebase 오류처리 
           Serial.print("setting /message failed:");
           Serial.println(Firebase.error());  
           return;
          }
  }



/////////////////////////////////////////////////////////////////////////

//1-4. ★밖에 있는 내가 택배함에 택배를 갖고 밖으로나간다★///////////////////////////////////////////////////////////////////////
  if(checkIr1 && digitalRead(irPin1)){
    comeOut = true;
  }
  
  //도난의심 알림
  if(comeOut && pakage && !changeWeight(weight, preWeight)) {
    flag = 0;
    pakage = false;
    pakageCount = 0;
    checkIr1 = 0;
    comeOut = false;
    Serial.print("SuspectedTheft")
    Firebase.setString("getPakage: ","SuspectedTheft");//파어베이스에 데이터 전송 
       if (Firebase.failed()) { //firebase 오류처리 
           Serial.print("setting /message failed:");
           Serial.println(Firebase.error());  
           return;
          }
  }
}





//택배가 2번 이상 왔을 때 판별 해주는 함수
int changeWeight(int weight, int preWeight) {
  boolean difference = false;
  if(weight - preWeight > 0){
   difference = true;
     Serial.print("증가");

     
  }
  else if (weight - preWeight <= 0){
   difference = false;
  }
  
  return difference;
}
