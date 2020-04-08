Title: Making a Digital Power Meter - Part 2: API
Date: 2019-02-02 15:00
Category: Smart Meter
Tags: ESP8266

In [Part 1]({filename}/smartmeter.md), I explained the general idea for my smart meter and how I assembled the hardware as well as the microcontroller code. In this part, I will explain the little API that I built to record the ticks and to display it using either a website or another microcontroller with a small OLED display.

I decided to build a [RESTful API](https://de.wikipedia.org/wiki/Representational_State_Transfer) using the [Flask microframework](http://flask.pocoo.org/) with [Connexion](http://flask.pocoo.org/). I chose Flask mainly because I have already worked with it at university in the past, but don't worry, it is very easy to get started with it. Have a look at the [Connexion helloworld example](https://github.com/zalando/connexion/tree/master/examples/openapi3/helloworld), you can start from there and adapt it. You can also use any other API framework, and as long as it's Python, you should be able to reuse most of my code if you want to.

Our API only needs to be able to do three things for us:

1. Provide an endpoint to which the ESP8266 can report disc-spins (ticks)
2. Save information on the ticks in some kind of permanent data storage (how many ticks happened and when did each tick happen)
2. Provide a way to look at the recorded information, ideally in an easy way that even my (less geeky) roommates can use

Let's start with our first objective. Connexion allows us to define our routes in a yaml file:
```yaml
swagger: '2.0'
info:
  title: Very simple energy API
  version: "0.1"
consumes:
  - application/json
produces:
  - application/json
paths:
  /tick:
    get:
      summary: Signal one tick (13.3Wh)
      operationId: api.tick
      responses:
        200:
          description: Successful tick
```
Let's ignore everything above the 'paths' directive for now and look at the first path (or route) that we defined. The first line defines the URL under which we will find this API method later. Our microcontroller will be able to call this method by sending a request to `http://192.168.0.38/tick` (Obviously, this address will be different for you, depending on where you decide to run your API). In addition to the yaml file defining the structure of our API, we also create a python file in which we define the methods that should be executed when a certain api endpoint is called. The method that ought to be executed when we call our /tick route is already specified in our yaml file in the `operationID` attribute. Let's create a python file called api.py with the following content:
```python
import connexion
from flask import render_template, make_response
import sqlite3
import time
from datetime import datetime, time as dttime
import json

conn = sqlite3.connect('energy.db')
c = conn.cursor()
c.execute('CREATE TABLE IF NOT EXISTS energy(timestamp INTEGER)')
c.execute('CREATE TABLE IF NOT EXISTS connection_loss(timestamp INTEGER, duration INTEGER, ticks_missed INTEGER)')
conn.commit()


def tick():
    ts = time.time()
    print("13Wh consumed! Timestamp is: {}. Time readable: {}.".format(ts, datetime.utcfromtimestamp(ts).strftime(
        '%H:%M:%S %d-%m-%Y')))

    c.execute("INSERT INTO energy VALUES ({})".format(ts))
    conn.commit()


app = connexion.App(__name__)
app.add_api('api.yaml')

# Set the WSGI application callable to allow using uWSGI:
# uwsgi --http :8080 -w app
application = app.app


def main():
    # Run our standalone gevent server
    app.debug = True
    app.run(port=8080, server='gevent')


if __name__ == '__main__':
    main()
```
The tick method is very simple: All it does is add a unix timestamp to the `energy.db` sqlite database file. This simple method is already enough to start recording data on our energy consumption! We can run the api in debug mode by executing `python3 api.py` in a shell.

So now that we've started saving data to the database, it's time to work on a way to view this information. I still had a couple of ESPs lying around, as well as a tiny OLED display that I've been planning to try out for a while, so I decided to use them and make a little display in our kitchen that shows the most important info. To do that, I created an additional API route:
```yaml
  /lcdinfo:
    get:
      summary: Get today's energy consumption to show on an lcd screen
      operationId: api.get_lcd_info
      responses:
        200:
          description: Successful response
```
Next, we create the corresponding method in our api.py file:
```python
def get_lcd_info():
    midnight = datetime.combine(datetime.today(), dttime.min)
    c.execute("SELECT Count(*) FROM energy WHERE timestamp > {}".format(midnight.timestamp()))
    ticks_today = c.fetchone()
    daily_power = float('%.2f' % (ticks_today[0] * 13.33333))

    c.execute("SELECT * FROM energy ORDER BY energy._ROWID_ DESC LIMIT 2")
    last_2_timestamps = c.fetchall()

    time_diff_secs = last_2_timestamps[0][0] - last_2_timestamps[1][0]

    # Time it took to consume 13.3Wh extrapolated to hourly energy use
    current_power = int(13.33333 / time_diff_secs * 3600)

    result = {"daily_power": daily_power, "current_power": current_power}

    return result
```
This simple method will calculate our total energy consumption for the current day as well as our current energy consumption. The result is a dictionary, which will automatically be serialized into JSON by the connexion framework. We can then query this API method with a second ESP and display this information on the OLED display. Since I know the price per kWh, I also calculate the cost on the ESP.

Finally, I also wanted a way to check the energy consumption when I was not at home or at least not in the kitchen. Therefore, I created a third API endpoint, which, when called, returns an html website with the same data as the OLED display, and a nice plot showing our daily energy consumption. Flask offers the very useful method `render_template()`, which allows us to write [jinja templates](http://jinja.pocoo.org/) and fill them with data using small snippets of python code. We can use that to provide our energy data, which we can then plot on the client side (in the browser) using javascript. This is the code that selects the data from the database and returns the populated html page:
```python
def render_html():
    c.execute("SELECT Count(*) FROM energy")
    total_power_in_db = c.fetchone()
    total_power = float('%.2f' % (total_power_in_db[0] * 13.33333 + 25503550))

    midnight = datetime.combine(datetime.today(), dttime.min)
    c.execute("SELECT Count(*) FROM energy WHERE timestamp > {}".format(midnight.timestamp()))
    ticks_today = c.fetchone()
    daily_power = float('%.2f' % (ticks_today[0] * 13.33333))

    # TODO: Implement average daily power here for each day

    c.execute("SELECT * FROM energy ORDER BY energy._ROWID_ DESC LIMIT 2")
    last_2_timestamps = c.fetchall()

    time_diff_secs = last_2_timestamps[0][0] - last_2_timestamps[1][0]

    # Time it took to consume 13.3Wh extrapolated to hourly energy use
    current_power = float('%.2f' % (13.33333 / time_diff_secs * 3600))

    c.execute("SELECT timestamp FROM energy WHERE timestamp > {}".format(midnight.timestamp()))
    rows = c.fetchall()

    data_points = []

    c.execute("SELECT * FROM energy WHERE timestamp > {}".format(midnight.timestamp()))

    data_rows = c.fetchall()

    for i in range(1, len(data_rows)):
        seconds = data_rows[i][0] - data_rows[i-1][0]
        watt = int(13.33333 / seconds * 3600)
        timestamp_avg = (data_rows[i][0] + data_rows[i-1][0]) / 2
        data_points.append([timestamp_avg, watt])

    # print(data_points)
    
    headers = {'Content-Type': 'text/html'}

    return make_response(render_template('index.html', current_power=current_power, total_power=total_power,
                                         daily_power=daily_power, data_points=json.dumps(data_points)), 200,
                         headers)
```
You can find [index.html in the GitHub repository](https://github.com/LasseMoench/smartmeter/blob/master/API/templates/index.html). I decided to use [google charts](https://developers.google.com/chart/) for plotting the data, however, you could also plot it using python libraries like matplotlib and include the graphs in the website. Congratulations, you can now have a close look on your energy consumption!
