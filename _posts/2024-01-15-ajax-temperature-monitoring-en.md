---
layout: post
title: "Non-Standard Temperature Monitoring in an Apartment"
description: "All you need is Apple Shortcuts + Go + InfluxDB + Grafana"
category: blog
blog: true
tags: ["golang", "influxdb", "grafana", "monitoring"]
lang: en
---

Due to communal accidents, I would like to monitor the temperature in an apartment where I currently do not live.
In my apartment, several sensors from [Ajax.Systems](https://ajax.systems) are installed, such as leak detectors, smoke detectors, and others.
These sensors can capture data about the ambient temperature.
Most likely, they have something like a thermistor installed - a type of resistor whose resistance is highly sensitive to temperature.

It seems that you can even use existing means to monitor the temperature, but I do not have the ability to update the software of the system and sensors.
The system also does not have an API. Therefore, we will try to collect temperature information from the iOS application.

Here's what the screen looks like from which we need to collect information:

![ajax rooms](/assets/blog/temperature_monitoring/ajax_rooms.png)

## Apple Shortcuts

At first, I thought I needed to jailbreak the iPhone and come up with something by writing an application that would collect data, but then I remembered that Apple introduced a feature called Shortcuts for automating various scenarios a few years ago, like sending a message once a day or integrating with home assistant, for example, to play music when you come home.
I had never used shortcuts before, and the abundance of possibilities really amazed me.

So, in the available actions when creating your shortcut, there are the actions we need:

- Open app - open a specific application
- Take screenshot - take a screenshot
- Extract text from ... - get text from an image, in our case from a screenshot. iOS does this using its neural networks, very convenient :)
- Get contents of URL - we need this to send a POST request with the text obtained in the previous step

I created the following shortcut:

![shortcut1](/assets/blog/temperature_monitoring/shortcut1.png)
![shortcut2](/assets/blog/temperature_monitoring/shortcut2.png)

Next, the shortcut should run periodically. I tried to do this using the Repeat action, where steps can be run a certain number of times,
run, wait for a minute, for example, and run again. But the problem is that every 10 screenshots, the system asks for permission to take a screenshot.

Therefore, let's turn to another feature in Shortcuts - Automation. Among the Automation options is executing a shortcut at a specific time of day.
We will run our shortcut once an hour, for this, we will have to add 24 tasks for each hour. It is important when creating a task to note that it should run immediately without confirmation.


![automation1](/assets/blog/temperature_monitoring/automation1.png)
![automation2](/assets/blog/temperature_monitoring/automation2.png)

All this will run on my old iPhone, which will be connected to the charger and will always remain unlocked (this setting exists, do not lock the phone automatically).

## Go

The text that we send to our server from our shortcut looks like this:

```
18:08
三つ
854
Tower
Disarmed
Hall
18°C
Kitchen
18°C
Bathroom
20°C
Toilet
20°C
+ Add Room
Devices
Rooms
Notifications
Control
```

Let's write a small script in Go that will accept a POST request at the `/temp` endpoint.
The endpoint accepts JSON with this text and parses it; eventually, our text is in `input.Data`. Then, we need to extract the temperature and the corresponding room from the text:

```
lines := strings.Split(input.Data, "\n")
re := regexp.MustCompile(`^(\d+)°C$`)
roomTemps := make(map[string]interface{})

for i := 0; i < len(lines); i++ {
	matches := re.FindStringSubmatch(lines[i])
	if matches != nil {
		v, _ := strconv.Atoi(matches[1])
		roomTemps[lines[i-1]] = v
	}
}
```

As a result, we have a map `roomTemps` of the form: `map[Bathroom:20 Hall:18 Kitchen:18 Toilet:20]`. To send these readings to InfluxDB, we use the influxdb-client-go client.

```
point := write.NewPoint("temp", tags, params, time.Now())

if err := writeAPI.WritePoint(context.Background(), point); err != nil {
	log.Fatal(err)
}

```

All source code can be found here: [https://github.com/accessd/ajax-temp-monitor](https://github.com/accessd/ajax-temp-monitor)

## InfluxDB

I also run InfluxDB, like the Go application, through Docker.
All you need to do after starting is configure it and generate an auth token.

```
docker-compose exec influxdb influx setup
docker-compose exec influxdb influx auth create \
  --org org \
  --all-access
```

## Grafana

Perhaps the best visualization is to use Grafana. It can also be installed on your own server, but I will use Grafana Cloud.
All I need to do is create a new Source with my InfluxDB:

![grafana source](/assets/blog/temperature_monitoring/grafana_source.png)

As the password, we specify the auth token we generated earlier.

And add a new dashboard with the charts I need. I added 2 charts. One shows the current temperature in each room as a gauge chart and
the second is simply with curves.

![grafana dash](/assets/blog/temperature_monitoring/grafana_dashboard.png)

I also added an alert to Grafana. If the temperature drops to 16 degrees, I will receive an alert in Telegram. Hopefully, it won't come :)
