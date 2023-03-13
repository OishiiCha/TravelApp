# TravelApp

#### Here's an example Python application that uses a Raspberry Pi 4 to record GPS location, temperature, humidity, time and date, and geiger counter data. The application will store the data in a local database and display the current and past data on a webpage with a map. It also includes an Android app to view the webpage.

### To run this application, you'll need the following components:

  * Raspberry Pi 4
  * GPS module
  * Temperature and humidity sensor (such as the DHT11 or DHT22)
  * Geiger counter
  * Python 3
  * Flask web framework
  * SQLite database
  * Apache web server (optional)
  
## Here are the steps to set up the application:

#### 1. Connect the GPS module to the Raspberry Pi 4 via a USB port. Install the gpsd package using the following command:
```
sudo apt-get install gpsd gpsd-clients python-gps
```

#### 2. Connect the temperature and humidity sensor to the Raspberry Pi 4 using GPIO pins. Install the Adafruit_DHT package using the following command:
```
sudo pip3 install Adafruit_DHT
```

#### 3. Connect the geiger counter to the Raspberry Pi 4 via a USB port. Install the pyserial package using the following command:
```
sudo pip3 install pyserial
```

#### 4. Create a new Python file called app.py and copy the following code into it:
```
from flask import Flask, render_template
from gps import gps, WATCH_ENABLE
from datetime import datetime
import time
import Adafruit_DHT
import serial
import sqlite3

app = Flask(__name__)

@app.route('/')
def index():
    # Get current data
    gps_data = get_gps_data()
    temp, humidity = get_temp_humidity_data()
    geiger_count = get_geiger_data()

    # Save data to database
    save_data_to_db(gps_data, temp, humidity, geiger_count)

    # Get past data
    past_data = get_past_data_from_db()

    # Render template with current and past data
    return render_template('index.html', gps_data=gps_data, temp=temp, humidity=humidity, geiger_count=geiger_count, past_data=past_data)

def get_gps_data():
    # Connect to GPS
    session = gps(mode=WATCH_ENABLE)

    # Get GPS data
    try:
        while True:
            report = session.next()
            if report['class'] == 'TPV':
                if hasattr(report, 'lat') and hasattr(report, 'lon'):
                    lat = round(report.lat, 6)
                    lon = round(report.lon, 6)
                    return {'lat': lat, 'lon': lon}
    except StopIteration:
        session = None
        return None

def get_temp_humidity_data():
    # Set sensor type and GPIO pin
    sensor = Adafruit_DHT.DHT22
    pin = 4

    # Get temperature and humidity data
    humidity, temp = Adafruit_DHT.read_retry(sensor, pin)
    if humidity is not None and temp is not None:
        humidity = round(humidity, 2)
        temp = round(temp, 2)
        return temp, humidity
    else:
        return None, None

def get_geiger_data():
    # Connect to geiger counter
    ser = serial.Serial('/dev/ttyUSB0', 9600, timeout=1)

    # Get geiger count
    line = ser.readline().decode('utf-8').strip()
    if line.startswith('$GEO'):
        parts = line.split(',')
        count = int(parts[1])
        return count
    else:
        return None

def save_data_to_db(gps_data, temp, humidity, geiger_count):
    # Connect to database
    conn = sqlite3.connect('data.db')
    c = conn.cursor()

    # Create table if it doesn't exist
    c.execute('''CREATE TABLE IF NOT EXISTS sensor_data
                 (timestamp TEXT, lat REAL, lon REAL, temp REAL, humidity REAL, geiger_count INTEGER)''')

    # Insert data into table
    timestamp = datetime.now().strftime('%Y-%m-%d %H:%M:%S')
    c.execute("INSERT INTO sensor_data VALUES (?, ?, ?, ?, ?, ?)", (timestamp, gps_data['lat'], gps_data['lon'], temp, humidity, geiger_count))

    # Commit changes and close connection
    conn.commit()
    conn.close()

def get_past_data_from_db():
    # Connect to database
    conn = sqlite3.connect('data.db')
    c = conn.cursor()

    # Retrieve data from table
    c.execute("SELECT * FROM sensor_data ORDER BY timestamp DESC LIMIT 10")
    rows = c.fetchall()

    # Close connection
    conn.close()

    # Return data
    return rows
```

This code defines a Flask application with a single route that renders a template called 'index.html'. The route uses several helper functions to get the current GPS location, temperature, humidity, and geiger counter data, save the data to a SQLite database, and retrieve the past 10 data points from the database. The 'index.html' template will be responsible for displaying the data on the web page.

#### 5. Create a new directory called templates and create a new file called index.html inside this directory. Copy the following code into it:
```
<!DOCTYPE html>
<html>
<head>
    <title>Raspberry Pi Sensor Data</title>
    <script src="https://polyfill.io/v3/polyfill.min.js?features=default"></script>
    <script src="https://maps.googleapis.com/maps/api/js?key=YOUR_API_KEY&callback=initMap"
    async defer></script>
    <style>
        #map {
            height: 400px;
            width: 100%;
        }
    </style>
</head>
<body>
    <h1>Raspberry Pi Sensor Data</h1>
    <h2>Current Data</h2>
    {% if gps_data %}
    <p>Latitude: {{ gps_data.lat }}</p>
    <p>Longitude: {{ gps_data.lon }}</p>
    {% else %}
    <p>Unable to get GPS data</p>
    {% endif %}
    {% if temp and humidity %}
    <p>Temperature: {{ temp }} &deg;C</p>
    <p>Humidity: {{ humidity }}%</p>
    {% else %}
    <p>Unable to get temperature and humidity data</p>
    {% endif %}
    {% if geiger_count %}
    <p>Geiger Count: {{ geiger_count }}</p>
    {% else %}
    <p>Unable to get geiger counter data</p>
    {% endif %}

    <h2>Past Data</h2>
    <table>
        <tr>
            <th>Timestamp</th>
            <th>Latitude</th>
            <th>Longitude</th>
            <th>Temperature (&deg;C)</th>
            <th>Humidity (%)</th>
            <th>Geiger Count</th>
        </tr>
        {% for row in past_data %}
        <tr>
            <td>{{ row[0] }}</td>
            <td>{{ row[1] }}</td>
            <td>{{ row[2] }}</td>
            <td>{{ row[3] }}</td>
            <td>{{ row[4] }}</td>
            <td>{{ row[5] }}</td>
        </tr>
        {% endfor %}
    </table>

    <div id="map"></div>
    <script>
        function initMap() {
            var map = new google.maps.Map(document.getElementById('map'), {
                center: {lat: {{ gps_data.lat }}, lng: {{ gps_data.lon }}},
                zoom: 8
            });

            {% for row in past_data %}
            var marker{{ loop.index }} = new google.maps.Marker({
                position: {lat: {{ row[1] }}, lng: {{ row[2] }}},
                map: map
            });
            {% endfor %}
        }
    </script>
</body>
</html>
```
This code defines an HTML page that displays the current and past sensor data and a Google Maps widget that displays the current and past GPS locations. Note that you will need to replace YOUR_API_KEY with your own Google Maps API key.

#### 6. Open a terminal window and navigate to the directory where you saved app.py. Run the following command to start the Flask application:
```
export FLASK_APP=app.py
flask run --host=0.0.0.0
```
This will start the Flask application and make it available to other devices on the same network. Make a note of the IP address of your Raspberry Pi (which you can find by running 'ifconfig' in a terminal window) and the port number that Flask is running on (which will be displayed in the terminal output).

#### 7. On your Android device, open a web browser and navigate to the IP address and port number of your Raspberry Pi (e.g. 'http://192.168.0.10:5000'). You should see the sensor data displayed on the web page. If you don't see the GPS data or the Google Maps widget, make sure you have entered your Google Maps API key in index.html.

#### 8. To see the sensor data update in real-time, you can create a simple Android app that periodically refreshes the web page. To do this, create a new Android project in Android Studio and add a 'WebView' to the main activity layout. Then, in the 'onCreate' method of the main activity, add the following code:
```
WebView webView = findViewById(R.id.webView);
webView.setWebViewClient(new WebViewClient());
webView.getSettings().setJavaScriptEnabled(true);
webView.loadUrl("http://192.168.0.10:5000");
```

This will load the web page in the WebView. To refresh the web page every 10 seconds, add the following code to the main activity:
```
new Handler().postDelayed(new Runnable() {
    @Override
    public void run() {
        webView.reload();
        handler.postDelayed(this, 10000);
    }
}, 10000);
```

This will refresh the web page every 10 seconds using a 'Handler'.

#### 9. Run the Android app on your device and you should see the sensor data updating in real-time. You can also use the app to view the past sensor data and GPS locations on the Google Maps widget.

