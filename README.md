# Smart-Cargo-Theft-detection
IoT-Based Smart Cargo Theft Detection and Prevention System Using ESP32
#include <WiFi.h>
#include <WebServer.h>
#include <Wire.h>
#include <Adafruit_MPU6050.h>
#include <Adafruit_Sensor.h>
#include <DHT.h>

// ================= WIFI =================
const char* ssid = "vivot4x";
const char* password = "12345678";

// ================= PINS =================
#define DHTPIN 4
#define DHTTYPE DHT11

#define MQ2_PIN 34
#define MQ5_PIN 35

#define TRIG_PIN 5
#define ECHO_PIN 18

#define BUZZER_PIN 23

// ================= OBJECTS =================
DHT dht(DHTPIN, DHTTYPE);
Adafruit_MPU6050 mpu;
WebServer server(80);

// ================= VARIABLES =================
float temperature = 0;
float humidity = 0;

int mq2Value = 0;
int mq5Value = 0;

float distance = 0;

float acceleration = 0;
float shockValue = 0;

int safetyScore = 100;

String safetyStatus = "SAFE";
String shockStatus = "NORMAL";
String gasStatus = "NORMAL";
String cargoStatus = "STABLE";
String openingStatus = "CLOSED";

// Reference distance for cargo/door
float referenceDistance = 0;

// ================= THRESHOLDS =================

// Adjust these according to your prototype
float TEMP_WARNING = 35.0;
float TEMP_HIGH = 45.0;

int MQ2_WARNING = 1800;
int MQ2_HIGH = 2800;

int MQ5_WARNING = 1800;
int MQ5_HIGH = 2800;

float SHOCK_WARNING = 2.5;
float SHOCK_HIGH = 4.0;

float MOVEMENT_THRESHOLD = 8.0;
float OPEN_THRESHOLD = 15.0;

// ================= HTML DASHBOARD =================

const char dashboard[] PROGMEM = R"rawliteral(

<!DOCTYPE html>
<html>

<head>

<meta name="viewport" content="width=device-width, initial-scale=1">

<title>Smart Cargo Safety Monitor</title>

<style>

body{
    font-family:Arial;
    background:#101820;
    color:white;
    margin:0;
    padding:15px;
}

.header{
    text-align:center;
    padding:10px;
}

h1{
    margin-bottom:5px;
}

.scoreBox{
    text-align:center;
    background:#1d2a35;
    padding:20px;
    border-radius:20px;
    margin-bottom:15px;
}

.score{
    font-size:55px;
    font-weight:bold;
}

.status{
    font-size:25px;
    font-weight:bold;
}

.grid{
    display:grid;
    grid-template-columns:repeat(auto-fit,minmax(150px,1fr));
    gap:12px;
}

.card{
    background:#1d2a35;
    padding:18px;
    border-radius:15px;
    text-align:center;
}

.value{
    font-size:27px;
    font-weight:bold;
    margin-top:8px;
}

.label{
    color:#aab5bd;
}

.safe{
    color:#00e676;
}

.warning{
    color:#ffca28;
}

.danger{
    color:#ff5252;
}

.normal{
    color:#00e676;
}

</style>

</head>

<body>

<div class="header">

<h1>🚛 Smart Cargo Safety Monitor</h1>

<p>ESP32 IoT Cargo Protection System</p>

</div>

<div class="scoreBox">

<div class="label">CARGO SAFETY SCORE</div>

<div id="score" class="score">--</div>

<div id="status" class="status">Loading...</div>

</div>

<div class="grid">

<div class="card">
<div class="label">🌡 Temperature</div>
<div id="temp" class="value">--</div>
</div>

<div class="card">
<div class="label">💧 Humidity</div>
<div id="humidity" class="value">--</div>
</div>

<div class="card">
<div class="label">💨 MQ-2 Gas/Smoke</div>
<div id="mq2" class="value">--</div>
</div>

<div class="card">
<div class="label">🛢 MQ-5 Gas</div>
<div id="mq5" class="value">--</div>
</div>

<div class="card">
<div class="label">📳 Shock</div>
<div id="shock" class="value">--</div>
</div>

<div class="card">
<div class="label">📦 Cargo Movement</div>
<div id="cargo" class="value">--</div>
</div>

<div class="card">
<div class="label">🚪 Cargo Opening</div>
<div id="opening" class="value">--</div>
</div>

<div class="card">
<div class="label">📏 Distance</div>
<div id="distance" class="value">--</div>
</div>

</div>

<script>

function updateDashboard(){

fetch('/data')

.then(response => response.json())

.then(data => {

document.getElementById("score").innerHTML =
data.score + " / 100";

document.getElementById("status").innerHTML =
data.status;

document.getElementById("temp").innerHTML =
data.temperature + " °C";

document.getElementById("humidity").innerHTML =
data.humidity + " %";

document.getElementById("mq2").innerHTML =
data.mq2;

document.getElementById("mq5").innerHTML =
data.mq5;

document.getElementById("shock").innerHTML =
data.shockStatus;

document.getElementById("cargo").innerHTML =
data.cargoStatus;

document.getElementById("opening").innerHTML =
data.openingStatus;

document.getElementById("distance").innerHTML =
data.distance + " cm";

let status =
document.getElementById("status");

status.className="status";

if(data.status=="SAFE"){
    status.classList.add("safe");
}

else if(data.status=="WARNING"){
    status.classList.add("warning");
}

else{
    status.classList.add("danger");
}

});

}

setInterval(updateDashboard,1000);

updateDashboard();

</script>

</body>

</html>

)rawliteral";

// ================= ULTRASONIC =================

float readDistance(){

    digitalWrite(TRIG_PIN, LOW);
    delayMicroseconds(2);

    digitalWrite(TRIG_PIN, HIGH);
    delayMicroseconds(10);

    digitalWrite(TRIG_PIN, LOW);

    long duration =
    pulseIn(ECHO_PIN, HIGH, 30000);

    if(duration == 0)
        return -1;

    float d =
    duration * 0.0343 / 2;

    return d;
}

// ================= SHOCK =================

void readShock(){

    sensors_event_t a,g,temp;

    mpu.getEvent(&a,&g,&temp);

    acceleration =
    sqrt(
        a.acceleration.x * a.acceleration.x +
        a.acceleration.y * a.acceleration.y +
        a.acceleration.z * a.acceleration.z
    );

    // Convert m/s² to G
    shockValue =
    acceleration / 9.81;

    if(shockValue >= SHOCK_HIGH){

        shockStatus = "HIGH SHOCK";

    }

    else if(shockValue >= SHOCK_WARNING){

        shockStatus = "WARNING";

    }

    else{

        shockStatus = "NORMAL";

    }
}

// ================= SAFETY SCORE =================

void calculateSafetyScore(){

    safetyScore = 100;

    // Temperature
    if(temperature >= TEMP_HIGH)
        safetyScore -= 20;

    else if(temperature >= TEMP_WARNING)
        safetyScore -= 10;


    // Humidity
    if(humidity >= 80)
        safetyScore -= 10;

    else if(humidity >= 70)
        safetyScore -= 5;


    // MQ2
    if(mq2Value >= MQ2_HIGH)
        safetyScore -= 25;

    else if(mq2Value >= MQ2_WARNING)
        safetyScore -= 15;


    // MQ5
    if(mq5Value >= MQ5_HIGH)
        safetyScore -= 20;

    else if(mq5Value >= MQ5_WARNING)
        safetyScore -= 10;


    // Shock
    if(shockValue >= SHOCK_HIGH)
        safetyScore -= 20;

    else if(shockValue >= SHOCK_WARNING)
        safetyScore -= 10;


    // Cargo movement
    if(referenceDistance > 0 &&
       abs(distance - referenceDistance)
       >= MOVEMENT_THRESHOLD){

        safetyScore -= 15;

        cargoStatus = "MOVEMENT";

    }

    else{

        cargoStatus = "STABLE";

    }


    // Opening
    if(referenceDistance > 0 &&
       abs(distance - referenceDistance)
       >= OPEN_THRESHOLD){

        safetyScore -= 25;

        openingStatus = "OPEN / ALERT";

    }

    else{

        openingStatus = "CLOSED";

    }


    if(safetyScore < 0)
        safetyScore = 0;


    // Final status

    if(safetyScore >= 90){

        safetyStatus = "SAFE";

    }

    else if(safetyScore >= 60){

        safetyStatus = "WARNING";

    }

    else{

        safetyStatus = "HIGH RISK";

    }


    // Gas status

    if(mq2Value >= MQ2_HIGH ||
       mq5Value >= MQ5_HIGH){

        gasStatus = "HIGH GAS";

    }

    else if(mq2Value >= MQ2_WARNING ||
            mq5Value >= MQ5_WARNING){

        gasStatus = "WARNING";

    }

    else{

        gasStatus = "NORMAL";

    }


    // Buzzer

    if(safetyStatus == "HIGH RISK" ||
       openingStatus == "OPEN / ALERT" ||
       shockStatus == "HIGH SHOCK"){

        digitalWrite(BUZZER_PIN,HIGH);

    }

    else{

        digitalWrite(BUZZER_PIN,LOW);

    }

}

// ================= JSON DATA =================

void sendData(){

    String json = "{";

    json += "\"temperature\":" +
            String(temperature,1) + ",";

    json += "\"humidity\":" +
            String(humidity,1) + ",";

    json += "\"mq2\":" +
            String(mq2Value) + ",";

    json += "\"mq5\":" +
            String(mq5Value) + ",";

    json += "\"shock\":" +
            String(shockValue,2) + ",";

    json += "\"distance\":" +
            String(distance,1) + ",";

    json += "\"score\":" +
            String(safetyScore) + ",";

    json += "\"status\":\"" +
            safetyStatus + "\",";

    json += "\"shockStatus\":\"" +
            shockStatus + "\",";

    json += "\"cargoStatus\":\"" +
            cargoStatus + "\",";

    json += "\"openingStatus\":\"" +
            openingStatus + "\"";

    json += "}";

    server.send(
        200,
        "application/json",
        json
    );
}

// ================= SETUP =================

void setup(){

    Serial.begin(115200);

    pinMode(TRIG_PIN,OUTPUT);
    pinMode(ECHO_PIN,INPUT);

    pinMode(BUZZER_PIN,OUTPUT);

    digitalWrite(BUZZER_PIN,LOW);

    // DHT
    dht.begin();

    // MPU6050
    Wire.begin(21,22);

    if(!mpu.begin()){

        Serial.println("MPU6050 not detected!");

        while(1){
            delay(100);
        }

    }

    Serial.println("MPU6050 OK");

    // WiFi

    WiFi.begin(
        ssid,
        password
    );

    Serial.print("Connecting to WiFi");

    while(WiFi.status() != WL_CONNECTED){

        delay(500);

        Serial.print(".");

    }

    Serial.println();

    Serial.println("WiFi connected!");

    Serial.print("Dashboard IP: ");

    Serial.println(
        WiFi.localIP()
    );


    // Initial ultrasonic reading

    delay(1000);

    referenceDistance =
    readDistance();

    Serial.print(
        "Reference distance: "
    );

    Serial.println(
        referenceDistance
    );


    // Web server

    server.on("/", [](){

        server.send_P(
            200,
            "text/html",
            dashboard
        );

    });

    server.on("/data", sendData);

    server.begin();

    Serial.println(
        "Web server started"
    );

}

// ================= LOOP =================

void loop(){

    server.handleClient();


    // DHT

    float t =
    dht.readTemperature();

    float h =
    dht.readHumidity();

    if(!isnan(t))
        temperature = t;

    if(!isnan(h))
        humidity = h;


    // Gas sensors

    mq2Value =
    analogRead(MQ2_PIN);

    mq5Value =
    analogRead(MQ5_PIN);


    // Ultrasonic

    float newDistance =
    readDistance();

    if(newDistance > 0)
        distance = newDistance;


    // MPU6050

    readShock();


    // Calculate score

    calculateSafetyScore();


    // Serial monitor

    Serial.println(
        "---------------------------"
    );

    Serial.print("Temperature: ");
    Serial.println(temperature);

    Serial.print("Humidity: ");
    Serial.println(humidity);

    Serial.print("MQ2: ");
    Serial.println(mq2Value);

    Serial.print("MQ5: ");
    Serial.println(mq5Value);

    Serial.print("Shock: ");
    Serial.println(shockValue);

    Serial.print("Distance: ");
    Serial.println(distance);

    Serial.print("Safety Score: ");
    Serial.println(safetyScore);

    Serial.print("Status: ");
    Serial.println(safetyStatus);

    delay(1000);

}
