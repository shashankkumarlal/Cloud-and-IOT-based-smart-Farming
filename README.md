#USE IDE THAT SUPPORTS PYTHON I HAVE USED THONNY
import network
import time
import machine
from machine import Pin, ADC
import urequests  # To send HTTP requests to Firebase
import ujson

# WiFi credentials
WIFI_SSID = "wifi"
WIFI_PASSWORD = "12345678"

"""Firebase (easier for beginners)
Step 1: Create a Firebase Project
Go to the Firebase Console.
Click on Add Project in the top-right corner.
Follow the prompts to create a new project:
Name the project (e.g., "SmartFarmingProject").
Select your country/region and accept the terms.
Disable Google Analytics (if you don’t want to use it).
Once the project is created, click on Continue.
Step 2: Set Up Firebase Realtime Database
In the Firebase Console, on the left-hand menu, click Realtime Database under the "Build" section.
Click on Create Database.
Choose Start in test mode for development (be sure to update security rules for production).
Set the location for your database (choose the one closest to you).
Click Enable.
Step 3: Get Firebase Credentials
Go to Project Settings by clicking the gear icon next to Project Overview.
Under Service Accounts, click Firebase Admin SDK.
Click Generate New Private Key.
Save the JSON file. This will contain the credentials you need to interact with Firebase from your Arduino (or other platforms).
"""

# Firebase credentials
FIREBASE_HOST = "https:......."

"""Get Firebase Credentials
In the Firebase Console, go to Project Settings by clicking the gear icon next to Project Overview.
Under Service Accounts, click Firebase Admin SDK.
Click Generate New Private Key.
Download the JSON file containing your credentials."""

FIREBASE_AUTH = "........"

# Pin definitions
MOISTURE_SENSOR_PIN = 0  # Soil Moisture Sensor Pin (A0)
TRIG_PIN = 14            # Ultrasonic Sensor Trigger Pin (D5)
ECHO_PIN = 12            # Ultrasonic Sensor Echo Pin (D6)
IR_SENSOR_PIN = 13       # IR Sensor Pin (D7)

# Connect to Wi-Fi
def connect_wifi():
    wlan = network.WLAN(network.STA_IF)
    wlan.active(True)
    if not wlan.isconnected():
        print('Connecting to WiFi...')
        wlan.connect(WIFI_SSID, WIFI_PASSWORD)
        while not wlan.isconnected():
            time.sleep(1)
    print('Connected to WiFi, IP:', wlan.ifconfig()[0])

# Initialize sensors
moisture_sensor = ADC(MOISTURE_SENSOR_PIN)  # Soil Moisture sensor
ir_sensor = Pin(IR_SENSOR_PIN, Pin.IN)  # IR sensor
trig_pin = Pin(TRIG_PIN, Pin.OUT)  # Ultrasonic Trigger Pin
echo_pin = Pin(ECHO_PIN, Pin.IN)  # Ultrasonic Echo Pin

# Read the soil moisture sensor
def read_soil_moisture():
    moisture_level = moisture_sensor.read()
    print(f"Soil Moisture Level: {moisture_level}")  # Debug print
    if moisture_level == 1024:
        print("Warning: Maximum moisture level detected. Soil may be too wet.")
    return moisture_level

# Read the ultrasonic sensor
def read_ultrasonic_distance():
    trig_pin.value(0)
    time.sleep_us(2)
    trig_pin.value(1)
    time.sleep_us(10)
    trig_pin.value(0)
    pulse_duration = machine.time_pulse_us(echo_pin, 1)
    if pulse_duration == -1:
        print("Invalid pulse duration, check ultrasonic sensor connection.")
        return 0  # Invalid distance
    else:
        distance = pulse_duration * 0.0344 / 2  # Convert to cm
        print(f"Ultrasonic Distance: {distance} cm")  # Debug print
        return distance

# Read the IR sensor
def read_ir_sensor():
    ir_value = ir_sensor.value()  # Returns 0 if no object, 1 if object detected
    print(f"IR Sensor Status: {ir_value}")  # Debug print
    return ir_value

# Upload data to Firebase
def upload_to_firebase(moisture, distance, ir_status):
    url = FIREBASE_HOST + "/sensorData.json?auth=" + FIREBASE_AUTH
    data = {
        "moisture": moisture,
        "distance": distance,
        "irStatus": ir_status
    }
    headers = {'Content-Type': 'application/json'}
    try:
        response = urequests.post(url, data=ujson.dumps(data), headers=headers)
        print("Data uploaded:", response.status_code)
        response.close()
    except Exception as e:
        print(f"Error uploading data to Firebase: {e}")

# Test sensors before starting main loop
def test_sensors():
    print("Testing soil moisture sensor...")
    moisture = read_soil_moisture()
    print("Testing ultrasonic sensor...")
    distance = read_ultrasonic_distance()
    print("Testing IR sensor...")
    ir_status = read_ir_sensor()
    return moisture, distance, ir_status

# Main logic
def main():
    connect_wifi
    # Test sensors
    moisture, distance, ir_status = test_sensors()

    while True:
        # Read sensor data
        moisture = read_soil_moisture()
        distance = read_ultrasonic_distance()
        ir_status = read_ir_sensor()

        # Debug print to see sensor data
        print(f"Moisture: {moisture}, Distance: {distance}, IR Status: {ir_status}")

        # Upload data to Firebase if changed
        upload_to_firebase(moisture, distance, ir_status)

        # Control irrigation logic based on sensor data
        if moisture < 400 and ir_status == 1:  # Dry soil and no object detected
            print("Soil is dry and no object detected, activating water pump...")
        elif ir_status == 0:  # Object detected
            print("Object detected, irrigation blocked.")
        else:  # Soil is moist enough
            print("Soil is moist enough, pump is off.")

        time.sleep(10)  # Wait for 10 seconds before the next reading

# Run the program
main()
