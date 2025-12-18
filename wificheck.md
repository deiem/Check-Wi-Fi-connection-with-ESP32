#include <WiFi.h>
#include <ESPping.h>
#include <SPI.h>
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

// --- Configuration ---
#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET -1
#define SCREEN_ADDRESS 0x3C
// Corrected button pin definition and name
#define BUTTON_PIN 4 

Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

// Your Wi-Fi Credentials
const char* ssid = "Wi-Fi Name"; //enter your wifi ssid
const char* password = ""; //Wi-Fi password

// Function prototypes (needed because initWiFi calls checkInternet, and connectAndCheckWiFi calls initWiFi)
void checkInternet();
void initWiFi();
void connectAndCheckWiFi();


// --- Function to Check Internet Connection ---
void checkInternet() {
    IPAddress dnsServer(8, 8, 8, 8); // Google's Public DNS Server

    Serial.print("Pinging...");

    // *** Note: If this fails, you may need to change <ESPping.h> to <ESP32Ping.h> ***
    if (Ping.ping(dnsServer)) {
        display.setTextSize(1);
        display.setCursor(0, 50);
        display.println("Internet: OK");
        display.display();
        Serial.println("Success! Internet is reachable.");
    } else {
        display.setTextSize(1);
        display.setCursor(0, 50);
        display.println("Internet: FAILED");
        display.display();
        Serial.println("Failed! Router connected, but no internet.");
    }
}

// --- Function to Connect to WiFi (Reusable) ---
void initWiFi() {
    WiFi.mode(WIFI_STA);
    WiFi.begin(ssid, password);
    
    // Display connection message
    display.clearDisplay();
    display.setTextSize(1);
    display.setTextColor(SSD1306_WHITE);
    display.setCursor(0, 0);
    display.println("Connecting to:");
    display.println(ssid);
    display.display();
    
    Serial.print("Connecting to WiFi ..");
    
    // Added Timeout Logic (20 attempts = 20 seconds)
    int attempts = 0;
    while (WiFi.status() != WL_CONNECTED && attempts < 20) { 
        Serial.print('.');
        display.print('.');
        display.display(); 
        delay(1000);
        attempts++;
    }
    
    // Check Connection Status After Timeout
    if (WiFi.status() == WL_CONNECTED) {
        // Connection successful
        display.clearDisplay();
        display.setTextSize(2); 
        display.setCursor(0, 10);
        display.println("CONNECTED!");
        display.setTextSize(1);
        display.setCursor(0, 35);
        display.print("IP: ");
        display.println(WiFi.localIP());
        
        Serial.println("\nConnected to WiFi!");
        Serial.print("IP Address: ");
        Serial.println(WiFi.localIP());
        
        // Check Internet Connection
        checkInternet(); 
    } else {
        // Connection FAILED
        display.clearDisplay();
        display.setTextSize(1);
        display.setCursor(0, 0);
        display.println("Connection FAILED!");
        display.setCursor(0, 20);
        display.println("Press button to retry.");
        display.display();

        Serial.println("\nConnection FAILED after timeout!");
    }
}

// --- Combined Function to Run the Sequence ---
void connectAndCheckWiFi() {
    Serial.println("\n--- Initiating Reconnect Sequence ---");
    // Disconnect first to ensure a fresh start
    WiFi.disconnect(true); // true to forget all credentials
    delay(500);
    
    // Run the main connection logic
    initWiFi();
}

// --- Setup Function (Runs Once) ---
void setup() {
    Serial.begin(9600);
    delay(100);

    // Initialize Button Pin with internal pull-up resistor
    // Assuming the button is wired between GPIO 4 and GND. 
    pinMode(BUTTON_PIN, INPUT_PULLUP); 

    // Initialize OLED
    if(!display.begin(SSD1306_SWITCHCAPVCC, SCREEN_ADDRESS)) { 
        Serial.println(F("SSD1306 allocation failed"));
        for(;;); 
    }
    display.display(); 
    delay(2000); 
    
    // Initial connection attempt on startup
    connectAndCheckWiFi();
}

// --- Loop Function (Runs Repeatedly) ---
void loop() {
    // Check for button press (LOW because of INPUT_PULLUP)
    if (digitalRead(BUTTON_PIN) == LOW) {
        Serial.println("Button Pressed: Restarting connection...");
        delay(50); // Debounce delay
        
        // Trigger the full sequence
        connectAndCheckWiFi(); 
        
        // Wait until the button is released to prevent multiple triggers
        while(digitalRead(BUTTON_PIN) == LOW) {
            delay(50);
        }
    }
    
    // Optional: Add a continuous check for connection loss here
    // If you uncommented the code below previously, it will auto-reconnect 
    // without a button press if the connection is lost.
    /*
    if (WiFi.status() != WL_CONNECTED) {
        Serial.println("Wi-Fi connection lost, attempting auto-reconnect...");
        connectAndCheckWiFi();
    }
    */
    
    delay(1000); // Small loop delay
}
