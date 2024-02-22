# Rural-smart-water

## Problem Descreption
Water is an essential component for maintaining good health and well-being.
However, one main challenge for sustainable development in Eswatini is bridging the
gap in water supply between rural and urban areas, ensuring reliable access to clean
water and sanitation services to all people. Lack of access to clean water and
sanitation can lead to various challenging conditions in our country, mainly health
issues. In spite of the Rural Water Supply Branch (RWSB), a fully-fledged government
department, that is involved with the design, construction, maintenance, repair and
rehabilitation of rural water supply schemes, rural areas have always faced difficulty
in accessing clean water supplies due to lack of timely maintenance of water
schemes. RWSB does provide technical support to communities in the form of
maintenance technicians to assist communities in repairing broken down
components of the water supply schemes. The section also provides training for
community members to carry out minor maintenance of their respective water
supply schemes[1]. However, the problem lies in accurately capturing information
about the water supply system and making informed decisions in real-time. This
could lead to quick responses in the case the system fails, planned and timely
maintenance ensuring a constant supply of clean water through:
● Remote monitoring
● Remote control

## The Esp32 code
This is the code that runs on the arduino esp32 board that harvest the data for the stream 
through sensor. The connected sensors are:
1. Turbidity sensor which detect the amoount of suspended particles in water.test for dissolved chemicals in the water
2. pH sensor that tests the pH of water
3. TSD  that test for dissolved chemicals in the water

This sensors upload the readings to the arduino board which then sends the data to the google firebase databese which will display on the webapp.
the board also controls the the pump, the valves that are in the water supply system.

/*
    - ESP32: https://RandomNerdTutorials.com/firebase-control-esp32-gpios/
    - ESP8266: https://RandomNerdTutorials.com/firebase-control-esp8266-nodemcu-gpios/
*/ 


#include <Arduino.h>
#if defined(ESP32)
  #include <WiFi.h>
#elif defined(ESP8266)
  #include <ESP8266WiFi.h>
#endif
#include <Firebase_ESP_Client.h>
#include <Wire.h>
#include <Adafruit_Sensor.h>
#include <DHT.h>

// Provide the token generation process info.
#include "addons/TokenHelper.h"
// Provide the RTDB payload printing info and other helper functions.
#include "addons/RTDBHelper.h"

// Insert your network credentials
#define WIFI_SSID "MTN_4G_B8EB"
#define WIFI_PASSWORD "30483891"

// Insert Firebase project API Key
#define API_KEY "AIzaSyCSF_RI5mHbmgcDEPvZHJQ6HRaazeDVros"

// Insert Authorized Username and Corresponding Password
#define USER_EMAIL "dminenhle477@gmail.com"
#define USER_PASSWORD "1234567890"

#define TdsSensorPin 27
#define VREF 3.3              // analog reference voltage(Volt) of the ADC
#define SCOUNT  30            // sum of sample point

// Insert RTDB URLefine the RTDB URL
#define DATABASE_URL "https://rural-water-project-default-rtdb.firebaseio.com/"

//DHT11 definition
#define DHTPIN 21
#define DHTTYPE DHT11

DHT dht(DHTPIN, DHTTYPE);

// Define Firebase objects
FirebaseData fbdo;
FirebaseData stream;
FirebaseAuth auth;
FirebaseConfig config;

// Variable to save USER UID
String uid;
// Variables to save database paths
String databasePath;
String tempPath;
String humPath;
String presPath;

//pH 
int pH_Value; 
float Voltage;

//turbidity
int turbidity;
              
// dht11 sensor
//Adafruit_BME280 bme; // I2C
float temperature;
float humidity;

// Timer variables (send new readings every three minutes)
unsigned long sendDataPrevMillis = 0;
unsigned long timerDelay = 180000;

// Variables to save database paths
String listenerPath = "board1/outputs/digital/";

// Declare outputs
const int output1 = 12;
const int output2 = 13;
const int output3 = 14;

int analogBuffer[SCOUNT];     // store the analog value in the array, read from ADC
int analogBufferTemp[SCOUNT];
int analogBufferIndex = 0;
int copyIndex = 0;

float averageVoltage = 0;
float tdsValue = 0;
float temperaturer = 25;       // current temperature for compensation

// median filtering algorithm
int getMedianNum(int bArray[], int iFilterLen){
  int bTab[iFilterLen];
  for (byte i = 0; i<iFilterLen; i++)
  bTab[i] = bArray[i];
  int i, j, bTemp;
  for (j = 0; j < iFilterLen - 1; j++) {
    for (i = 0; i < iFilterLen - j - 1; i++) {
      if (bTab[i] > bTab[i + 1]) {
        bTemp = bTab[i];
        bTab[i] = bTab[i + 1];
        bTab[i + 1] = bTemp;
      }
    }
  }
  if ((iFilterLen & 1) > 0){
    bTemp = bTab[(iFilterLen - 1) / 2];
  }
  else {
    bTemp = (bTab[iFilterLen / 2] + bTab[iFilterLen / 2 - 1]) / 2;
  }
  return bTemp;
}

void startDHT(){
  dht.begin();
}

void getReadings(){
  temperature = dht.readTemperature();
  humidity = dht.readHumidity();
}

// Initialize WiFi
void initWiFi() {
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
  Serial.print("Connecting to WiFi ..");
  while (WiFi.status() != WL_CONNECTED) {
    Serial.print('.');
    delay(1000);
  }
  Serial.println(WiFi.localIP());
  Serial.println();
}

// Write float values to the database
void sendFloat(String path, float value){
  if (Firebase.RTDB.setFloat(&fbdo, path.c_str(), value)){
    Serial.print("Writing value: ");
    Serial.print (value);
    Serial.print(" on the following path: ");
    Serial.println(path);
    Serial.println("PASSED");
    Serial.println("PATH: " + fbdo.dataPath());
    Serial.println("TYPE: " + fbdo.dataType());
  }
  else {
    Serial.println("FAILED");
    Serial.println("REASON: " + fbdo.errorReason());
  }
}

// Callback function that runs on database changes
void streamCallback(FirebaseStream data){
  Serial.printf("stream path, %s\nevent path, %s\ndata type, %s\nevent type, %s\n\n",
                data.streamPath().c_str(),
                data.dataPath().c_str(),
                data.dataType().c_str(),
                data.eventType().c_str());
  printResult(data); //see addons/RTDBHelper.h
  Serial.println();

  // Get the path that triggered the function
  String streamPath = String(data.dataPath());

  // if the data returned is an integer, there was a change on the GPIO state on the following path /{gpio_number}
  if (data.dataTypeEnum() == fb_esp_rtdb_data_type_integer){
    String gpio = streamPath.substring(1);
    int state = data.intData();
    Serial.print("GPIO: ");
    Serial.println(gpio);
    Serial.print("STATE: ");
    Serial.println(state);
    digitalWrite(gpio.toInt(), state);
  }

  /* When it first runs, it is triggered on the root (/) path and returns a JSON with all keys
  and values of that path. So, we can get all values from the database and updated the GPIO states*/
  if (data.dataTypeEnum() == fb_esp_rtdb_data_type_json){
    FirebaseJson json = data.to<FirebaseJson>();

    // To iterate all values in Json object
    size_t count = json.iteratorBegin();
    Serial.println("\n---------");
    for (size_t i = 0; i < count; i++){
        FirebaseJson::IteratorValue value = json.valueAt(i);
        int gpio = value.key.toInt();
        int state = value.value.toInt();
        Serial.print("STATE: ");
        Serial.println(state);
        Serial.print("GPIO:");
        Serial.println(gpio);
        digitalWrite(gpio, state);
        Serial.printf("Name: %s, Value: %s, Type: %s\n", value.key.c_str(), value.value.c_str(), value.type == FirebaseJson::JSON_OBJECT ? "object" : "array");
    }
    Serial.println();
    json.iteratorEnd(); // required for free the used memory in iteration (node data collection)
  }
  
  //This is the size of stream payload received (current and max value)
  //Max payload size is the payload size under the stream path since the stream connected
  //and read once and will not update until stream reconnection takes place.
  //This max value will be zero as no payload received in case of ESP8266 which
  //BearSSL reserved Rx buffer size is less than the actual stream payload.
  Serial.printf("Received stream payload size: %d (Max. %d)\n\n", data.payloadLength(), data.maxPayloadLength());
}

void streamTimeoutCallback(bool timeout){
  if (timeout)
    Serial.println("stream timeout, resuming...\n");
  if (!stream.httpConnected())
    Serial.printf("error code: %d, reason: %s\n\n", stream.httpCode(), stream.errorReason().c_str());
}

void setup(){
  Serial.begin(115200);
  initWiFi();
   startDHT();
   pinMode(TdsSensorPin,INPUT);
   pinMode(pH_Value, INPUT); 

  // Initialize Outputs
  pinMode(output1, OUTPUT);
  pinMode(output2, OUTPUT);
  pinMode(output3, OUTPUT);

  
  // Assign the api key (required)
  config.api_key = API_KEY;

  // Assign the user sign in credentials
  auth.user.email = USER_EMAIL;
  auth.user.password = USER_PASSWORD;

  // Assign the RTDB URL (required)
  config.database_url = DATABASE_URL;

  Firebase.reconnectWiFi(true);

  // Assign the callback function for the long running token generation task */
  config.token_status_callback = tokenStatusCallback; //see addons/TokenHelper.h

  // Assign the maximum retry of token generation
  config.max_token_generation_retry = 5;

  // Initialize the library with the Firebase authen and config
  Firebase.begin(&config, &auth);

  // Streaming (whenever data changes on a path)
  // Begin stream on a database path --> board1/outputs/digital
  if (!Firebase.RTDB.beginStream(&stream, listenerPath.c_str()))
    Serial.printf("stream begin error, %s\n\n", stream.errorReason().c_str());

  // Assign a calback function to run when it detects changes on the database
  Firebase.RTDB.setStreamCallback(&stream, streamCallback, streamTimeoutCallback);

  // Getting the user UID might take a few seconds
  Serial.println("Getting User UID");
  while ((auth.token.uid) == "") {
    Serial.print('.');
    delay(1000);
  }
  // Print user UID
  uid = auth.token.uid.c_str();
  Serial.print("User UID: ");
  Serial.println(uid);

  // Update database path
  databasePath = "/UsersData/" + uid;

  // Update database path for sensor readings
  tempPath = databasePath + "/temperature"; // --> UsersData/<user_uid>/temperature
  humPath = databasePath + "/humidity"; // --> UsersData/<user_uid>/humidity
  presPath = databasePath + "/pressure"; // --> UsersData/<user_uid>/pressure

  delay(2000);
}

void loop(){

  pH_Value = analogRead(34); 
  
  Voltage = pH_Value * (3.3 / 4095.0); 
  Serial.println(Voltage); 
  Serial.print("pH: ");
  Serial.println((pH_Value/585)*2);
  delay(500);
  
   static unsigned long analogSampleTimepoint = millis();
  if(millis()-analogSampleTimepoint > 40U){     //every 40 milliseconds,read the analog value from the ADC
    analogSampleTimepoint = millis();
    analogBuffer[analogBufferIndex] = analogRead(TdsSensorPin);    //read the analog value and store into the buffer
    analogBufferIndex++;
    if(analogBufferIndex == SCOUNT){ 
      analogBufferIndex = 0;
    }
  }   
  
  static unsigned long printTimepoint = millis();
  if(millis()-printTimepoint > 800U){
    printTimepoint = millis();
    for(copyIndex=0; copyIndex<SCOUNT; copyIndex++){
      analogBufferTemp[copyIndex] = analogBuffer[copyIndex];
      
      // read the analog value more stable by the median filtering algorithm, and convert to voltage value
      averageVoltage = getMedianNum(analogBufferTemp,SCOUNT) * (float)VREF / 4096.0;
      
      //temperature compensation formula: fFinalResult(25^C) = fFinalResult(current)/(1.0+0.02*(fTP-25.0)); 
      float compensationCoefficient = 1.0+0.02*(temperature-25.0);
      //temperature compensation
      float compensationVoltage=averageVoltage/compensationCoefficient;
      
      //convert voltage value to tds value
      tdsValue=(133.42*compensationVoltage*compensationVoltage*compensationVoltage - 255.86*compensationVoltage*compensationVoltage + 857.39*compensationVoltage)*0.5;
      
      //Serial.print("voltage:");
      //Serial.print(averageVoltage,2);
      //Serial.print("V   ");
      Serial.print("TDS Value:");
      Serial.print(tdsValue+70,0);
      Serial.println("ppm");
    }
  }

  turbidity = analogRead(33); 

  Serial.print("Turbidity: ");
  Serial.println(turbidity+3);
  delay(500);
  
   // Send new readings to database
  if (Firebase.ready() && (millis() - sendDataPrevMillis > timerDelay || sendDataPrevMillis == 0)){
    sendDataPrevMillis = millis();
    getReadings();
    sendFloat(tempPath, (pH_Value/585)*2);
    sendFloat(humPath, turbidity+3);
    sendFloat(presPath, tdsValue+70);}
    
  if (Firebase.isTokenExpired()){
    Firebase.refreshToken(&config);
    Serial.println("Refresh token");
  }
}



### The webapp html code 

This is the html code that makes the webapp for visualizing and controlling the system, it was edited with visual studio code and deployed to firebase 
to fetch and send values to the firebase database and the app, which then communicate with the arduino board that controls the actual components on the 
feild.

<!DOCTYPE html>
<!-- Complete Project Details at: https://RandomNerdTutorials.com/ -->
<html>
  <head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>ESP IoT Firebase App</title>

    <!-- update the version number as needed -->
    <script src="https://www.gstatic.com/firebasejs/8.8.1/firebase-app.js"></script>

    <!-- include only the Firebase features as you need -->
    <script src="https://www.gstatic.com/firebasejs/8.8.1/firebase-auth.js"></script>
    <script src="https://www.gstatic.com/firebasejs/8.8.1/firebase-database.js"></script>

    <script>
      // REPLACE WITH YOUR web app's Firebase configuration
      const firebaseConfig = {
        apiKey: "AIzaSyCSF_RI5mHbmgcDEPvZHJQ6HRaazeDVros",
        authDomain: "rural-water-project.firebaseapp.com",
        databaseURL: "https://rural-water-project-default-rtdb.firebaseio.com",
        projectId: "rural-water-project",
        storageBucket: "rural-water-project.appspot.com",
        messagingSenderId: "71255041757",
        appId: "1:71255041757:web:983879d5227d539e761d34"
      };

      // Initialize firebase
      firebase.initializeApp(firebaseConfig);

      // Make auth and database references
      const auth = firebase.auth();
      const db = firebase.database();

    </script>
    <link rel="stylesheet" href="https://use.fontawesome.com/releases/v5.7.2/css/all.css" integrity="sha384-fnmOCqbTlWIlj8LyTjo7mOUStjsKC4pOpQbqyi7RrhN7udi9RwhKkMHpvLbHG9Sr" crossorigin="anonymous">
    <link rel="icon" type="image/png" href="favicon.png">
    <link rel="stylesheet" type="text/css" href="style.css">
</head>

<body>
  <!--TOP BAR-->
  <div class="topnav">
    <h1>Rural Smart Water  <i class="fas fa-lightbulb"></i></h1>
  </div>

  <!--AUTHENTICATION BAR (USER DETAILS/LOGOUT BUTTON)-->
  <div id="authentication-bar" style="display: none;">
    <p><span id="authentication-status">User logged in</span>
       <span id="user-details">USEREMAIL</span>
       <a href="/" id="logout-link">[logout]</a>
    </p>
  </div>

  <!--LOGIN FORM-->
  <form id="login-form" style="display: none;">
    <div class="form-elements-container">
      <label for="input-email"><b>Email</b></label>
      <input type="text" placeholder="Enter Username" id="input-email" required>

      <label for="input-password"><b>Password</b></label>
      <input type="password" placeholder="Enter Password" id="input-password" required>

      <button type="submit" id="login-button">Login</button>
      <p id="error-message" style="color:red;"></p>
    </div>
  </form>

  <!--CONTENT (SENSOR READINGS)-->
  <div class="content-sign-in" id="content-sign-in" style="display: none;">
    <div class="card-grid">
      <!--TEMPERATURE-->
      <div class="card">
        <p><i class="fas fa-thermometer-half" style="color:#059e8a;"></i> pH Level</p>
        <p><span class="reading"><span id="temp"></span> </span></p>
      </div>
      <!--HUMIDITY-->
      <div class="card">
        <p><i class="fas fa-tint" style="color:#00add6;"></i> Turbidity</p>
        <p><span class="reading"><span id="hum"></span> NTU</span></p>
      </div>
      <!--PRESSURE-->
      <div class="card">
        <p><i class="fas fa-angle-double-down" style="color:#e1e437;"></i> Solvent Level</p>
        <p><span class="reading"><span id="pres"></span> ppm</span></p>
      </div>
    
      <!--CARD FOR GPIO 12-->
      <div class="card">
        <p class="card-title"><i class="fas fa-lightbulb"></i> PUMP</p>
        <p>
          <button class="button-on" id="btn1On">ON</button>
          <button class="button-off" id="btn1Off">OFF</button>
        </p>
        <p class="state">State:<span id="state1"></span></p>
      </div>

      <!--CARD FOR GPIO 13-->
      <div class="card">
        <p class="card-title"><i class="fas fa-lightbulb"></i> Bi Control Valve</p>
        <p>
          <button class="button-on" id="btn2On">S - F</button>
          <button class="button-off" id="btn2Off">F - T</button>
        </p>
        <p class="state">State:<span id="state2"></span></p>
      </div>

      <!--CARD FOR GPIO 14-->
      <div class="card">
        <p class="card-title"><i class="fas fa-lightbulb"></i> O/P Valve</p>
        <p>
          <button class="button-on" id="btn3On">OPEN</button>
          <button class="button-off" id="btn3Off">CLOSE</button>
        </p>
        <p class="state">State:<span id="state3"></span></p>
      </div>
    </div>
  </div>
    <script src="scripts/auth.js"></script>
    <script src="scripts/index.js"></script>
  </body>
</html>

### The java script code for authentification
this codes cotrols the responsiveness of the app during log in and log out.

document.addEventListener("DOMContentLoaded", function(){

    // listen for auth status changes
    auth.onAuthStateChanged(user => {
        if (user) {
            console.log("user logged in");
            console.log(user);
            setupUI(user);
        } else {
            console.log("user logged out");
            setupUI();
        }
    });

    // login
    const loginForm = document.querySelector('#login-form');
    loginForm.addEventListener('submit', (e) => {
        e.preventDefault();
        // get user info
        const email = loginForm['input-email'].value;
        const password = loginForm['input-password'].value;
        // log the user in
        auth.signInWithEmailAndPassword(email, password).then((cred) => {
            // close the login modal & reset form
            loginForm.reset();
            console.log(email);
        })
        .catch((error) =>{
            const errorCode = error.code;
            const errorMessage = error.message;
            document.getElementById("error-message").innerHTML = errorMessage;
            console.log(errorMessage);
        });
    });

    // logout
    const logout = document.querySelector('#logout-link');
    logout.addEventListener('click', (e) => {
    e.preventDefault();
    auth.signOut();
    });
    
});



### The java script for index html file
const loginElement = document.querySelector('#login-form');
const contentElement = document.querySelector("#content-sign-in");
const userDetailsElement = document.querySelector('#user-details');
const authBarElement = document.querySelector("#authentication-bar");

// Elements for sensor readings
const tempElement = document.getElementById("temp");
const humElement = document.getElementById("hum");
const presElement = document.getElementById("pres");

// Elements for GPIO states
const stateElement1 = document.getElementById("state1");
const stateElement2 = document.getElementById("state2");
const stateElement3 = document.getElementById("state3");

// Button Elements
const btn1On = document.getElementById('btn1On');
const btn1Off = document.getElementById('btn1Off');
const btn2On = document.getElementById('btn2On');
const btn2Off = document.getElementById('btn2Off');
const btn3On = document.getElementById('btn3On');
const btn3Off = document.getElementById('btn3Off');

// Database path for GPIO states
var dbPathOutput1 = 'board1/outputs/digital/12';
var dbPathOutput2 = 'board1/outputs/digital/13';
var dbPathOutput3 = 'board1/outputs/digital/14';

// Database references
var dbRefOutput1 = firebase.database().ref().child(dbPathOutput1);
var dbRefOutput2 = firebase.database().ref().child(dbPathOutput2);
var dbRefOutput3 = firebase.database().ref().child(dbPathOutput3);

// MANAGE LOGIN/LOGOUT UI
const setupUI = (user) => {
  if (user) {
    //toggle UI elements
    loginElement.style.display = 'none';
    contentElement.style.display = 'block';
    authBarElement.style.display ='block';
    userDetailsElement.style.display ='block';
    userDetailsElement.innerHTML = user.email;

    // get user UID to get data from database
    var uid = user.uid;
    console.log(uid);

    // Database paths (with user UID)
    var dbPathTemp = 'UsersData/' + uid.toString() + '/temperature';
    var dbPathHum = 'UsersData/' + uid.toString() + '/humidity';
    var dbPathPres = 'UsersData/' + uid.toString() + '/pressure';

    // Database references
    var dbRefTemp = firebase.database().ref().child(dbPathTemp);
    var dbRefHum = firebase.database().ref().child(dbPathHum);
    var dbRefPres = firebase.database().ref().child(dbPathPres);

    // Update page with new readings
    dbRefTemp.on('value', snap => {
      tempElement.innerText = snap.val().toFixed(2);
    });

    dbRefHum.on('value', snap => {
      humElement.innerText = snap.val().toFixed(2);
    });

    dbRefPres.on('value', snap => {
      presElement.innerText = snap.val().toFixed(2);
    });

    // if user is logged out
  //} else{
    //// toggle UI elements
    //loginElement.style.display = 'block';
    //authBarElement.style.display ='none';
    //userDetailsElement.style.display ='none';
    //contentElement.style.display = 'none';
  //}
  //Update states depending on the database value
  dbRefOutput1.on('value', snap => {
    if(snap.val()==1) {
        stateElement1.innerText="ON";
    }
    else{
        stateElement1.innerText="OFF";
    }
});
dbRefOutput2.on('value', snap => {
    if(snap.val()==1) {
        stateElement2.innerText="ON";
    }
    else{
        stateElement2.innerText="OFF";
    }
});
dbRefOutput3.on('value', snap => {
    if(snap.val()==1) {
        stateElement3.innerText="ON";
    }
    else{
        stateElement3.innerText="OFF";
    }
});

// Update database uppon button click
btn1On.onclick = () =>{
    dbRefOutput1.set(1);
}
btn1Off.onclick = () =>{
    dbRefOutput1.set(0);
}

btn2On.onclick = () =>{
    dbRefOutput2.set(1);
}
btn2Off.onclick = () =>{
    dbRefOutput2.set(0);
}

btn3On.onclick = () =>{
    dbRefOutput3.set(1);
}
btn3Off.onclick = () =>{
    dbRefOutput3.set(0);
}

// if user is logged out
} else{
// toggle UI elements
loginElement.style.display = 'block';
authBarElement.style.display ='none';
userDetailsElement.style.display ='none';
contentElement.style.display = 'none';
}
}

### The css file for app visuals

html {
    font-family: Verdana, Geneva, Tahoma, sans-serif;
    display: inline-block;
    text-align: center;
}
p {
    font-size: 1.2rem;
}
h1 {
    font-size: 1.8rem;
    color: white;
}
.topnav {
    overflow: hidden;
    background-color: #049faa;
    color: white;
    font-size: 1rem;
    padding: 10px;
}
#authentication-bar{
    background-color:mintcream;
    padding-top: 10px;
    padding-bottom: 10px;
}

#user-details{
    color: cadetblue;
}

.content {
    padding: 20px;
}

body {
    margin: 0;
}

.card {
    background-color: white;
    box-shadow: 2px 2px 12px 1px rgba(140,140,140,.5);
    padding: 5%;
}

.cards {
    max-width: 800px;
    margin: 0 auto;
    display: grid;
    grid-gap: 2rem;
    grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
}

.card-grid {
    max-width: 800px;
    margin: 0 auto;
    display: grid;
    grid-gap: 2rem;
    grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
}
.card {
    background-color: white;
    box-shadow: 2px 2px 12px 1px rgba(140,140,140,.5);
}

.card-title {
    font-size: 1.2rem;
    color: #034078
}
.state {
    color:#1282A2;
}

.reading {
    font-size: 1.4rem;
}

button {
    background-color: #049faa;
    color: white;
    padding: 14px 20px;
    margin: 8px 0;
    border: none;
    cursor: pointer;
    border-radius: 4px;
}
.button-on {
    background-color:#034078;
}
button:hover {
    opacity: 0.8;
}
.button-on:hover {
    background-color: #1282A2;
}
.button-off {
    background-color:#858585;
}
.button-off:hover {
    background-color: #252524;
}

.form-elements-container{
    padding: 16px;
    width: 250px;
    margin: 0 auto;
}

input[type=text], input[type=password] {
    width: 100%;
    padding: 12px 20px;
    margin: 8px 0;
    display: inline-block;
    border: 1px solid #ccc;
    box-sizing: border-box;
}



