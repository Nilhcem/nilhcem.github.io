---
layout: post
title:  IoT - Home sensor data monitoring with MQTT, InfluxDB and Grafana
permalink: iot/home-monitoring-with-mqtt-influxdb-grafana
date: 2019-01-26
comments: true
---

It's winter now and the weather is pretty cold in France in early 2019.  

![pic01_snow]

I live in a small apartment with communal heating system, and so far, I never had to care about heating, except for a few _"Hmm, it seems a little cold right now!"_ moments.
And when this happens, some questions come to my mind:
- Is it really cold, or is it just me?
- How different is it from yesterday / last year?
- What's the temperature difference inside vs outside?

Measuring is important, we know that as developers, otherwise how could we get accurate answers and take the best decisions without data?  
Let's build a system to monitor the inside and outside temperature + humidity.

![pic02_grafana]
<br>


## Hardware

We will need:
- A temperature and humidity sensor
- An ESP8266, to communicate with the sensor and send data over WiFi
- A Raspberry Pi, to receive, store, and display data from the ESP8266.

<br>


## Temperature and Humidity sensor

The most famous temperature and humidity sensors in the makers community are the **DHT22** and the **BME280**. Each cost around $2.50.

![pic03_dht22_bme280]

After trying the 2 side by side for a few days, I noticed they were not giving me the exact same temperature: there was a ~2.5°C/36°F difference between the two sensors.

Since both are giving me different data, how can I know which is the most accurate one? So I called my parents for help, and they lent me a few mercury thermometers.

![pic04_mercury_thermometers]

The 2 mercury thermometers were displaying exactly the same temperature, so I assumed that they were accurate.  
The DHT22 temperature was ~1.5°C/35°F lower. The BME280 temperature was 1°C/34°F higher.

After some research, it turned out that, despite its impressive appearance, the DHT22 is actually not a very good sensor. The BME280 is better, but has its own problems too, as it suffers from **self-heating** in its default configuration _(sampling data continuously)_.

Hopefully, you can write code to change the sampling rate and make the sensor return to sleep mode when the measurement is finished.

The MCU I use is an **ESP8266** (Wemos D1 Mini), and I write Arduino code to communicate with the sensor.

![pic05_breadboard]
<br>

Instead of fetching data every second, I decided to fetch once per minute, with the sensor returning to sleep mode immediately after. The temperature it gave me then was very close to the one of the mercury thermometers.

{% highlight c %}
Adafruit_BME280 bme;

void setup() {
  bme.begin(SPI_ADDRESS);
  bme.setSampling(Adafruit_BME280::MODE_FORCED,
                Adafruit_BME280::SAMPLING_X1, // temperature
                Adafruit_BME280::SAMPLING_NONE, // pressure
                Adafruit_BME280::SAMPLING_X1, // humidity
                Adafruit_BME280::FILTER_OFF);
}

void loop() {
  bme.takeForcedMeasurement();
  humidity = bme.readHumidity();
  temperature = bme.readTemperature();
  delay(60000)
}
{% endhighlight %}
<br>


## MQTT broker

MQTT is a popular lightweight pub-sub protocol widely used in IoT.  

The Raspberry Pi will host an MQTT server (Mosquitto):
{% highlight sh %}
$ docker run -d -p 1883:1883 eclipse-mosquitto
{% endhighlight %}

And the ESP8266 will publish temperature and humidity data to the following MQTT topics:
- `home/bme280/temperature`
- `home/bme280/humidity`

To see if this works, you can use [MQTT.fx][mqttfx] to connect to your `[RaspberryPi IP]:1883` and subscribe to the temperature topic:

![pic06_mqttfx]

You can also install the [MQTT Dash][mqttdash] application on your Android device to see temperature data directly on your smartphone:

![pic07_mqttdash]

The complete Arduino implementation for the ESP8266 and the BME280 [is available here][bme280_arduino].
<br><br>


## InfluxDB

The Raspberry Pi will store humidity and temperature metrics over the time in an InfluxDB Time Series Database:

{% highlight sh %}
$ docker run -d -p 8086:8086 \
    -v $PWD:/var/lib/influxdb \
    influxdb
{% endhighlight %}

To avoid damaging your microSD card, it is a good idea to use an external storage _(hard drive / NAS / cloud)_ to store influxdb data.

Now that infuxdb is running, we need to write a small script that subscribes to MQTT events, and persists received data to InfluxDB

{% highlight python %}
def on_message(client, userdata, msg):
  sensor_data = parse_mqtt_message(msg.topic, msg.payload.decode('utf-8'))
  send_sensor_data_to_influxdb(sensor_data)

def parse_mqtt_message(topic, payload):
  match = re.match('home/([^/]+)/([^/]+)', topic)
  location = match.group(1)
  measurement = match.group(2)
  return SensorData(location, measurement, float(payload))

def send_sensor_data_to_influxdb(sensor_data):
  json_body = [
    {
      'measurement': sensor_data.measurement,
      'tags': {
        'location': sensor_data.location
      },
      'fields': {
        'value': sensor_data.value
      }
    }
  ]
  influxdb_client.write_points(json_body)
{% endhighlight %}

Complete implementation is [available here][bridge_mqtt].  
<br>


## Grafana

So far, the MCU is sending temperature and humidity data every minute to MQTT.  
When a message is published, values are automatically persisted to InfluxDB.  
Now, we need a tool to show these data over the time in a graph.

Grafana is an open-source, general purpose dashboard and graph composer.  
We will install it on the Raspberry Pi:

{% highlight sh %}
$ docker run -d -p 3000:3000 grafana/grafana
{% endhighlight %}

Now we can access it via `http://[RaspberryPi IP]:3000` and we can link it to InfluxDB:  
Menu > Configuration > Data Sources > Add

![pic08_grafana_setup]

Then we can create a new dashboard (graph) showing the BME280 temperature over the time:  
`FROM temperature WHERE location=bme280`.

![pic09_grafana_dashboard]

We can then create a similar graph for humidity data.
<br><br>


## Docker compose

To automate running containers, we can create a docker compose file.  
[See the docker-compose.yml here][docker_compose].
<br><br>


## Outside temperature

The ESP8266 with the BME280 is a good solution inside the home.  
However, I would like to also have the **outside** temperature and humidity data.

We could build a battery-powered similar device (ESP8266 + BME280), but I am too lazy to build one, so I decided to buy instead a "Xiaomi mijia temperature and humidity sensor" for ~$12.

![pic10_xiaomi_hygrothermograph]

While designed for an indoor usage, it supports temperatures from -9.9°C to 60°C. I've been using it outside for a month already and the device works really well.

![pic11_xiaomi_outside]
<br>

This device sends data using Bluetooth LE.  
You can easily reverse engineer the BLE part, and write your own code that communicates with the device via BLE and publishes data to an MQTT broker.  

[Full BLE to MQTT implementation here][mijia_mqtt]

This code in running on the Raspberry Pi 3 B+ along with all the other docker services.

![pic12_rpi_htop]

The Mijia device requires a single AAA battery. I use rechargeable batteries and it lasts more than a month.

![pic13_mijia_battery]
<br>


## Conclusion

You can find the complete source code on [github.com/Nilhcem/home-monitoring-grafana][github-repo] to build a similar project for your home.

I have been monitoring the inside and outside temperature/humidity for a month, and noticed that I'm getting more and more curious about the weather now.  
Actually, the more I have data and the more curious I am.  
Size of the influxDB after a month: 30MB.

A future iteration would be to try to do a similar project using exclusively online services, only keeping sensors in the house, getting rid of the Raspberry Pi.

![pic14_grafana]


[mqttfx]: https://mqttfx.jensd.de/
[mqttdash]: https://play.google.com/store/apps/details?id=net.routix.mqttdash
[bme280_arduino]: https://github.com/Nilhcem/home-monitoring-grafana/blob/master/03-bme280_mqtt/esp8266/esp8266.ino
[bridge_mqtt]: https://github.com/Nilhcem/home-monitoring-grafana/blob/master/02-bridge/main.py
[docker_compose]: https://github.com/Nilhcem/home-monitoring-grafana/blob/master/docker-compose.yml
[mijia_mqtt]: https://github.com/Nilhcem/home-monitoring-grafana/blob/master/04-mijia_ble_mqtt/main.py
[github-repo]: https://github.com/Nilhcem/home-monitoring-grafana

[pic01_snow]: /public/images/20190126/01_snow.jpg
[pic02_grafana]: /public/images/20190126/02_grafana.jpg
[pic03_dht22_bme280]: /public/images/20190126/03_bme280_dht22.jpg
[pic04_mercury_thermometers]: /public/images/20190126/04_mercury_thermometers.jpg
[pic05_breadboard]: /public/images/20190126/05_breadboard.jpg
[pic06_mqttfx]: /public/images/20190126/06_mqttfx.png
[pic07_mqttdash]: /public/images/20190126/07_mqttdash.png
[pic08_grafana_setup]: /public/images/20190126/08_grafana_setup.png
[pic09_grafana_dashboard]: /public/images/20190126/09_grafana_dashboard.png
[pic10_xiaomi_hygrothermograph]: /public/images/20190126/10_xiaomi_hygrothermograph.jpg
[pic11_xiaomi_outside]: /public/images/20190126/11_xiaomi_outside.jpg
[pic12_rpi_htop]:  /public/images/20190126/12_rpi_htop.png
[pic13_mijia_battery]:  /public/images/20190126/13_mijia_battery.png
[pic14_grafana]:  /public/images/20190126/14_grafana.png
