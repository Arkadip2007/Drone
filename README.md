# Drone

Here's the complete code integrating MPU6050 for auto-balance functionality with your drone, along with the existing features for controlling motors and LEDs via a web interface:

```cpp
#include <ESP8266WiFi.h>
#include <Wire.h>
#include <MPU6050.h>

// Access point credentials
const char* ssid = "DroneAP";
const char* password = "12345678";

// Motor pins
const int motor1Pin = D1;
const int motor2Pin = D2;
const int motor3Pin = D3;
const int motor4Pin = D4;

// LED pin
const int ledPin = D5;

// MPU6050 object
MPU6050 mpu;

// Variables to control motor speeds
int motor1Speed = 1000;
int motor2Speed = 1000;
int motor3Speed = 1000;
int motor4Speed = 1000;

// Desired angles for stabilization (adjust as needed)
float desiredPitch = 0.0;  // Degrees
float desiredRoll = 0.0;   // Degrees

// PID constants (adjust as needed)
float Kp_pitch = 5.0;
float Kp_roll = 5.0;

WiFiServer server(80);

void setup() {
  // Initialize serial communication
  Serial.begin(115200);

  // Initialize motor pins
  pinMode(motor1Pin, OUTPUT);
  pinMode(motor2Pin, OUTPUT);
  pinMode(motor3Pin, OUTPUT);
  pinMode(motor4Pin, OUTPUT);

  // Initialize LED pin
  pinMode(ledPin, OUTPUT);
  digitalWrite(ledPin, LOW);  // Turn off LED initially

  // Initialize I2C communication for MPU6050
  Wire.begin();
  mpu.initialize();

  // Check MPU6050 connection
  if (!mpu.testConnection()) {
    Serial.println("MPU6050 connection failed");
    while (1);
  }

  // Calibrate MPU6050 if needed
  mpu.setXAccelOffset(-2396);
  mpu.setYAccelOffset(-412);
  mpu.setZAccelOffset(1454);
  mpu.setXGyroOffset(44);
  mpu.setYGyroOffset(-19);
  mpu.setZGyroOffset(-9);

  // Set up the ESP8266 as an access point
  WiFi.softAP(ssid, password);
  Serial.println("Access point created");

  // Start the server
  server.begin();
  Serial.println("Server started");

  // Print the IP address
  Serial.println(WiFi.softAPIP());
}

void loop() {
  // Listen for incoming clients
  WiFiClient client = server.available();
  if (client) {
    Serial.println("New Client.");
    String currentLine = "";

    while (client.connected()) {
      if (client.available()) {
        char c = client.read();
        Serial.write(c);
        if (c == '\n') {
          if (currentLine.length() == 0) {
            client.println("HTTP/1.1 200 OK");
            client.println("Content-type:text/html");
            client.println();

            // HTML content for control interface
            client.println("<html><body><h1>Drone Control</h1>");
            client.println("<div id='joystickContainer' style='width:200px;height:200px;background-color:lightgray;position:relative;'>");
            client.println("<div id='joystick' style='width:50px;height:50px;background-color:gray;border-radius:50%;position:absolute;top:75px;left:75px;'></div>");
            client.println("</div>");
            client.println("<br>");
            client.println("<label for='speed'>Speed:</label>");
            client.println("<input type='range' id='speed' name='speed' min='1000' max='2000' oninput='updateSpeed(this.value)'>");
            client.println("<br>");
            client.println("<button onclick='toggleLED()'>Toggle LED</button>");
            client.println("<script>");
            client.println("var joystick = document.getElementById('joystick');");
            client.println("var joystickContainer = document.getElementById('joystickContainer');");
            client.println("var containerRect = joystickContainer.getBoundingClientRect();");
            client.println("var centerX = containerRect.width / 2 - joystick.offsetWidth / 2;");
            client.println("var centerY = containerRect.height / 2 - joystick.offsetHeight / 2;");
            client.println("joystick.style.left = centerX + 'px';");
            client.println("joystick.style.top = centerY + 'px';");

            client.println("joystick.addEventListener('touchmove', function(e) {");
            client.println("  var touch = e.touches[0];");
            client.println("  var touchX = touch.clientX - containerRect.left - joystick.offsetWidth / 2;");
            client.println("  var touchY = touch.clientY - containerRect.top - joystick.offsetHeight / 2;");
            client.println("  touchX = Math.max(0, Math.min(touchX, containerRect.width - joystick.offsetWidth));");
            client.println("  touchY = Math.max(0, Math.min(touchY, containerRect.height - joystick.offsetHeight));");
            client.println("  joystick.style.left = touchX + 'px';");
            client.println("  joystick.style.top = touchY + 'px';");

            client.println("  var commandX = (touchX - centerX) / centerX;");
            client.println("  var commandY = (touchY - centerY) / centerY;");
            client.println("  sendCommand(commandX, commandY);");
            client.println("}, false);");

            client.println("joystick.addEventListener('touchend', function() {");
            client.println("  joystick.style.left = centerX + 'px';");
            client.println("  joystick.style.top = centerY + 'px';");
            client.println("  sendCommand(0, 0);");
            client.println("}, false);");

            client.println("function sendCommand(x, y) {");
            client.println("  var xhr = new XMLHttpRequest();");
            client.println("  xhr.open('GET', '/MOVE?x=' + x + '&y=' + y, true);");
            client.println("  xhr.send();");
            client.println("}");
            client.println("function updateSpeed(value) {");
            client.println("  var xhr = new XMLHttpRequest();");
            client.println("  xhr.open('GET', '/SPEED?value=' + value, true);");
            client.println("  xhr.send();");
            client.println("}");
            client.println("function toggleLED() {");
            client.println("  var xhr = new XMLHttpRequest();");
            client.println("  xhr.open('GET', '/TOGGLE_LED', true);");
            client.println("  xhr.send();");
            client.println("}");
            client.println("</script>");
            client.println("</body></html>");

            break;
          } else {
            currentLine = "";
          }
        } else if (c != '\r') {
          currentLine += c;
        }

        // Handle different commands
        if (currentLine.startsWith("GET /MOVE")) {
          int xIndex = currentLine.indexOf("x=");
          int yIndex = currentLine.indexOf("&y=");
          if (xIndex != -1 && yIndex != -1) {
            String xValue = currentLine.substring(xIndex + 2, yIndex);
            String yValue = currentLine.substring(yIndex + 3);
            float x = xValue.toFloat();
            float y = yValue.toFloat();
            controlDrone(x, y);
          }
        } else if (currentLine.startsWith("GET /SPEED")) {
          int idx = currentLine.indexOf("value=");
          if (idx != -1) {
            String speedValue = currentLine.substring(idx + 6);
            int speed = speedValue.toInt();
            updateSpeed(speed);
          }
        } else if (currentLine.startsWith("GET /TOGGLE_LED")) {
          toggleLED();
        }
      }
    }
    client.stop();
    Serial.println("Client Disconnected.");
  }

  // Stabilize drone using MPU6050 data
  stabilizeDrone();
}

void controlDrone(float x, float y) {
  // Adjust motor speeds based on joystick input

  // Forward/Backward
  motor1Speed = 1500 + (y * 500);  // Motor 1 (front-left)
  motor2Speed = 1500 + (y * 500);  // Motor 2 (front-right)
  motor3Speed = 1500 - (y * 500);  // Motor 3 (rear-left)
  motor4Speed = 1500 - (y * 500);  // Motor 4 (rear-right)

  // Left/Right
  motor1Speed += x * 500;  // Motor 1 (front-left)
  motor2Speed -= x * 500;  // Motor 2 (front-right)
  motor3Speed -= x * 500;  // Motor 3 (rear-left)
  motor4Speed += x * 500;  // Motor 4 (rear-right)

  // Constrain speeds to safe values
  motor1Speed = constrain(motor1Speed, 1000, 2000);
  motor2Speed = constrain(motor2Speed, 1000, 2000);
  motor3Speed = constrain(motor3Speed, 1000, 2000);
  motor4Speed = constrain(motor4Speed, 1000, 2000);

  // Apply motor speeds
  analogWrite(motor1Pin, motor1Speed);
  analogWrite(motor2Pin, motor2Speed);
  analogWrite(motor3Pin, motor3Speed);
  analogWrite(motor4Pin, motor4Speed);
}

void updateSpeed(int

 speed) {
  motor1Speed = speed;
  motor2Speed = speed;
  motor3Speed = speed;
  motor4Speed = speed;

  // Apply motor speeds
  analogWrite(motor1Pin, motor1Speed);
  analogWrite(motor2Pin, motor2Speed);
  analogWrite(motor3Pin, motor3Speed);
  analogWrite(motor4Pin, motor4Speed);
}

void toggleLED() {
  static bool ledState = false;
  ledState = !ledState;
  digitalWrite(ledPin, ledState ? HIGH : LOW);
}

void stabilizeDrone() {
  // Read accelerometer and gyroscope data
  int16_t ax, ay, az, gx, gy, gz;
  mpu.getMotion6(&ax, &ay, &az, &gx, &gy, &gz);

  // Calculate pitch and roll angles (using accelerometer and gyroscope data)
  float pitch = atan2(-ax, sqrt(ay * ay + az * az)) * RAD_TO_DEG;
  float roll = atan2(ay, az) * RAD_TO_DEG;

  // Use PID control or simple proportional control to adjust motor speeds
  float pitchCorrection = Kp_pitch * (desiredPitch - pitch);
  float rollCorrection = Kp_roll * (desiredRoll - roll);

  // Adjust motor speeds based on corrections
  motor1Speed = 1500 + pitchCorrection - rollCorrection;  // Front-left
  motor2Speed = 1500 + pitchCorrection + rollCorrection;  // Front-right
  motor3Speed = 1500 - pitchCorrection - rollCorrection;  // Rear-left
  motor4Speed = 1500 - pitchCorrection + rollCorrection;  // Rear-right

  // Constrain speeds to safe values
  motor1Speed = constrain(motor1Speed, 1000, 2000);
  motor2Speed = constrain(motor2Speed, 1000, 2000);
  motor3Speed = constrain(motor3Speed, 1000, 2000);
  motor4Speed = constrain(motor4Speed, 1000, 2000);

  // Apply motor speeds
  analogWrite(motor1Pin, motor1Speed);
  analogWrite(motor2Pin, motor2Speed);
  analogWrite(motor3Pin, motor3Speed);
  analogWrite(motor4Pin, motor4Speed);
}
```

### Explanation:
1. **MPU6050 Integration**: The MPU6050 is initialized in the `setup()` function to read accelerometer and gyroscope data.
2. **Stabilization Functionality**: The `stabilizeDrone()` function calculates the pitch and roll angles using MPU6050 data and adjusts motor speeds (`motor1Speed` to `motor4Speed`) accordingly to maintain balance.
3. **Client Handling**: The `loop()` function handles client requests for controlling the drone via a web interface. Commands for adjusting motor speeds (`/MOVE`), setting speed (`/SPEED`), and toggling LED (`/TOGGLE_LED`) are processed.
4. **Motor Control**: The `controlDrone()` and `updateSpeed()` functions adjust motor speeds based on joystick inputs and direct speed updates, respectively.
5. **Web Interface**: Provides a basic HTML interface with a joystick for directional control, speed adjustment slider, and a button to toggle an LED.

### Note:
- **Adjustments**: You may need to adjust `Kp_pitch` and `Kp_roll` values to fine-tune the stabilization performance based on your drone's characteristics.
- **Safety**: Test the drone in a safe environment, ensuring proper stabilization before attempting flight.
- **Web Interface**: Customize the HTML and JavaScript for better user interaction or additional functionalities as needed.

Ensure your hardware setup (connections and power supply) is robust and suitable for your drone's operation. This code provides a foundational setup for integrating auto-balance functionality using MPU6050 with your drone controlled via a web interface. Adjustments and further development may be necessary based on specific drone design and performance requirements.

Here's a comprehensive guide with connections and a diagram for integrating the MPU6050, motors, and LEDs with your ESP8266-based drone. This setup assumes you have basic knowledge of electronics and soldering.

### Components Needed:
- ESP8266 development board (e.g., NodeMCU)
- MPU6050 accelerometer and gyroscope module
- Four brushless DC motors (for quadcopter configuration)
- Four electronic speed controllers (ESCs) compatible with your motors
- Power distribution board
- LiPo battery suitable for your motors and ESCs
- LEDs (optional, for indicating drone orientation)

### Connections:

#### ESP8266 (NodeMCU) to MPU6050:
- **SCL (D1)** pin of NodeMCU to **SCL** pin of MPU6050
- **SDA (D2)** pin of NodeMCU to **SDA** pin of MPU6050
- **VCC** and **GND** of MPU6050 to 3.3V and GND of NodeMCU, respectively

#### ESP8266 (NodeMCU) to Motors and ESCs:
- **Motor 1 Pin (D1)** of NodeMCU to signal pin of ESC 1
- **Motor 2 Pin (D2)** of NodeMCU to signal pin of ESC 2
- **Motor 3 Pin (D3)** of NodeMCU to signal pin of ESC 3
- **Motor 4 Pin (D4)** of NodeMCU to signal pin of ESC 4
- **GND** of NodeMCU to **GND** of ESCs
- **LiPo battery** positive to ESCs positive (through power distribution board)
- **LiPo battery** negative to ESCs negative (through power distribution board)

#### LEDs (Optional):
- **LED Pin (D5)** of NodeMCU to anode of the LED (through current limiting resistor, e.g., 220 ohms)
- **Cathode** of the LED to **GND**

### Diagram:
```
  MPU6050
  +-----------------+
  |                 |
  |  SDA      D2    |<-----> D2 (SDA) ESP8266 (NodeMCU)
  |  SCL      D1    |<-----> D1 (SCL) ESP8266 (NodeMCU)
  |  VCC      3.3V  |<-----> 3.3V ESP8266 (NodeMCU)
  |  GND      GND   |<-----> GND ESP8266 (NodeMCU)
  |                 |
  +-----------------+
          |
          |
          V

   ESP8266 (NodeMCU)
  +-----------------+
  |                 |
  |  D1             |-----> Signal ESC 1
  |  D2             |-----> Signal ESC 2
  |  D3             |-----> Signal ESC 3
  |  D4             |-----> Signal ESC 4
  |  D5             |-----> Anode LED (through resistor)
  |  GND            |<-----> GND
  |                 |
  +-----------------+

   ESCs and Motors
  +-----------------+
  |                 |
  |  Signal ESC 1   |-----> Motor 1
  |  Signal ESC 2   |-----> Motor 2
  |  Signal ESC 3   |-----> Motor 3
  |  Signal ESC 4   |-----> Motor 4
  |  GND            |<-----> GND
  |  LiPo +         |<-----> LiPo + (via PDB)
  |  LiPo -         |<-----> LiPo - (via PDB)
  |                 |
  +-----------------+

   LEDs (Optional)
  +-----------------+
  |                 |
  |  Anode          |-----> D5 (through resistor)
  |  Cathode        |<-----> GND
  |                 |
  +-----------------+
```

### Notes:
- **Power Distribution**: Use a power distribution board to connect the LiPo battery to each ESC. Ensure the power distribution board can handle the current requirements of your motors.
- **Current Protection**: Use appropriate current-limiting resistors for LEDs to prevent excessive current draw from the NodeMCU pins.
- **Safety**: Double-check all connections before powering up your drone. Ensure the LiPo battery is securely connected and observe proper safety precautions when working with LiPo batteries.

This setup provides a basic framework for integrating the MPU6050 for stabilization, controlling motors via the NodeMCU, and optionally controlling LEDs for visual indication. Adjustments may be necessary based on specific drone design and component specifications. Always test in a safe environment and ensure proper functionality before attempting flight.

Determining the front of a drone is crucial for proper orientation during operation and control. Here's a basic diagram illustrating how the front of a typical quadcopter drone is identified based on its physical design and component layout:

### Quadcopter Drone Front Diagram:

```
       Motor 1 (Front-Left)
      (       )
     [         ]
      \       /
       \     /
        \   /
         \ /
          O
         / \
        /   \
       /     \
      [         ]
       (       )
       Motor 2 (Front-Right)

     ===================
           Front
     ===================

       Motor 3 (Rear-Left)
      (       )
     [         ]
      \       /
       \     /
        \   /
         \ /
          O
         / \
        /   \
       /     \
      [         ]
       (       )
       Motor 4 (Rear-Right)
```

### Explanation:
- **Motors**: In a typical quadcopter configuration, there are four motors arranged in an "X" pattern when viewed from above. Motors are usually numbered in a clockwise or counterclockwise sequence starting from the front-left.
- **Front Orientation**: The front of the drone is generally where Motor 1 (front-left) and Motor 2 (front-right) are located. This configuration allows the drone to move forward when these motors increase thrust.
- **Center Point**: The center point between Motor 1 and Motor 2 represents the front of the drone. This is where control inputs are typically directed to move the drone forward.
- **LED Indicators**: Many drones have LED indicators located near the front arms or motor pods. These LEDs often blink or change color to indicate the orientation of the drone, helping pilots maintain control.

### Practical Considerations:
- **Flight Controller Alignment**: Ensure your flight controller (such as the ESP8266 in your setup) is aligned with the physical front of the drone. This alignment ensures that control inputs correspond correctly to the drone's movement relative to the pilot's perspective.
- **Control Inputs**: When piloting the drone, pushing the joystick or control stick forward should move the drone in the direction of the identified front. This intuitive setup aids in controlling the drone effectively.

By referencing this diagram and understanding the physical layout of your drone, you can easily identify and mark the front to ensure consistent and accurate operation during flights.
