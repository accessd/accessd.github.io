---
layout: post
title: "Нестандартный мониторинг температуры в квартире"
description: "All you need is Apple Shortcuts + Go + InfluxDB + Grafana"
category: blog
blog: true
tags: ["golang", "influxdb", "grafana", "monitoring"]
lang: ru
---

В связи с коммунальными авариями хотелось бы мониторить температуру в квартире в которой сейчас не живу.
В моей квартире установлено несколько датчиков из [Ajax.Systems](https://ajax.systems), типа датчиков протечки, дыма и прочее.
Данные датчики могут получать данные о температуре окружающей среды.
Скорее всего там установлено что-то типа термистора - это тип резисторов, сопротивление которых очень чувствительно к температуре.

Вроде как можно даже существующими средствами сделать мониторинг температуры, но возможности обновить ПО системы и датчиков у меня нет возможности.
Какого то API у системы также нет.  Поэтому попробуем собирать информацию о температуре из iOS приложения.

Как выглядит экран с которого нам надо собрать информацию:

![ajax rooms](/assets/blog/temperature_monitoring/ajax_rooms.png)

## Apple Shortcuts

Сперва я подумал что надо делать jailbreak айфона и там чего то придумывать с написанием приложения которое будет собирать данные, но потом вспомнил что у apple несколько лет назад
появилась такая штука как Shortcuts для автоматизации разных сценариев, типа раз в день отправлять сообщение или для интеграции с home assistant, например включить музыку когда пришел домой.
Я shortcuts до этого не пользовался и обилие возможностей прямо удивило.

Так вот, в возможных действиях при создании своего шортката есть то что нам понадобится:

- Open app - открыть определенное приложение
- Take screenshot - сделать скриншот экрана
- Extract text from ... - получить текст из изображения, в нашем случае из скриншота. Это iOS делает своими нейронками, удобно:)
- Get contents of URL - нам оно нужно для того чтобы POST запросом отправить текст полученный на предыдущем шаге

У меня получился такой shortcut:

![shortcut1](/assets/blog/temperature_monitoring/shortcut1.png)
![shortcut2](/assets/blog/temperature_monitoring/shortcut2.png)

Далее, shortcut должен запускаться с какой-то периодичностью. Я попробовал сделать это с помощью действия Repeat, когда шаги можно запускать какое-то количество раз,
запускаем, делаем wait например на минуту и снова запускаем. Но проблема в том что каждые 10 скриншотов система спрашивает разрешение на создание скриншота.

Поэтому обратимся к другой фиче в Shortcuts - это Automation. Среди возможностей Automation есть выполнение шортката в определенное время дня.
Будем запускать наш шорткат раз в час, для этого придется добавить 24 задачки на каждый час. Важно при создании задачки отметить чтобы она запускалась сразу, без подтверждения.

![automation1](/assets/blog/temperature_monitoring/automation1.png)
![automation2](/assets/blog/temperature_monitoring/automation2.png)

Все это будет крутится на моем старом айфоне который будет подключен к зарядке и который постоянно будет в разблокированном состоянии(это в настройках есть, не блокировать телефон автоматически).

## Go

Текст который мы передаем на наш сервер из нашего шортката выглядит в итоге так:

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

Напишем небольшой скрипт на go который будет принимать POST запрос на эндпоинте `/temp`.
Эндпоинт принимает json с данным текстом и парсит его, в итоге наш текст лежит в `input.Data`. Затем из текста надо достать температуру и к какой комнате она относится:

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

В итоге имеем map `roomTemps` такого вида: `map[Bathroom:20 Hall:18 Kitchen:18 Toilet:20]`. Чтобы отправить эти показатели в InfluxDB используем клиент influxdb-client-go.

```
point := write.NewPoint("temp", tags, params, time.Now())

if err := writeAPI.WritePoint(context.Background(), point); err != nil {
	log.Fatal(err)
}

```

Все исходники можно найти здесь: [https://github.com/accessd/ajax-temp-monitor](https://github.com/accessd/ajax-temp-monitor)

## InfluxDB

InfluxDB у меня поднимается также как и приложение go, через докер.
Все что надо сделать после запуска это сконфигурить его и сгенерить auth token.

```
docker-compose exec influxdb influx setup
docker-compose exec influxdb influx auth create \
  --org org \
  --all-access
```

## Grafana

Пожалуй лучшей визуализации не придумаешь чем использовать графану. Ее тоже можно поставить на собственный сервер, но я воспользуюсь Grafana Cloud.
Все что мне надо сделать это создать новый Source с моим InfluxDB:

![grafana source](/assets/blog/temperature_monitoring/grafana_source.png)

В качестве пароля указываем сгенеренный нами ранее auth token.

И добавить новый дэшборд с нужными мне графиками. Я добавил 2 графика. Один показывает текущую температуру в каждом помещении в виде gauge chart и
второй просто с кривыми.

![grafana dash](/assets/blog/temperature_monitoring/grafana_dashboard.png)

Также я добавил в графану alert. Если температура опустится до 16 градусов, то мне в телеграм придет алярм. Надеюсь не придет :)
