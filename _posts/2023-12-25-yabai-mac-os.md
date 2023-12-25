---
layout: post
title: "MacOS: tiling window manager"
description: "Тайловый менеджер окон в MacOS"
category: blog
blog: true
tags: ["yabai", "macos", "tiling manager", "vimlife", "hotkeys"]
lang: ru
---

Тайловые менержеры окон (tiling window managers) не так уж широко известны. Сам я слышал о них когда то давно, еще на линуксе,
но тогда не было желания их более подробно изучать. И я не видел чтобы кто-то из моих друзей или коллег их использовал.

Я как фанат vim и tmux, где твое пространство делится на окна между которыми ты легко переключаешься хоткеями, хотел бы такого же
взаимодействия с системой не только в терминале, но и с обычными окнами управляться без мышки.

Вот как tiling WM выглядит:

![yabai tiling wm](/assets/blog/yabai/how_it_looks.gif)

Изначально tiling WMs появились в линуксе, это например [i3](https://i3wm.org) и [bspwm](https://github.com/baskerville/bspwm).
MacOS как известно базируется на UNIX и в целом было логичным что для MacOS тоже возможно сделать tiling WM.

[yabai](https://github.com/koekeishiya/yabai) появился в 2019 году. По сути это порт линуксового [bspwm](https://github.com/baskerville/bspwm)
который использует [binary space partitioning](https://en.wikipedia.org/wiki/Binary_space_partitioning) для разделения рабочего стола на окна.
Оба этих WMs имеют cервисы для хоткеев, [skhd](https://github.com/koekeishiya/skhd) для yabai и [sxhkd](https://github.com/baskerville/sxhkd) для bswpm.
Эти сервисы позволяют перехватывать нажатия и посылать команды tiling WM.

Я пробовал yabai в 2021 году, но тогда не взлетело. Во-первых работал он нестабильно, во-вторых сложно было привыкнуть и некоторые вещи я не знал как реализовать,
это также как сложно начать пользоваться vim когда ты им никогда не пользовался. Но в 2023 я снова где-то наткнулся на упоминание yabai и решил попробовать еще раз.
На данный момент прошло около месяца, полет нормальный.


## Установка Yabai

На странице [wiki в github](https://github.com/koekeishiya/yabai/wiki#yabai) в целом все отлично описано. Из особенностей: вероятно придется отключать SIP в MacOS
чтобы пользоваться всеми возможностями, например созданием рабочих пространств.
Все конфигурирование осуществляется через конфиг `~/.yabairc`

Вот пример:

```
yabai -m config                                 \
    mouse_follows_focus          off            \
    focus_follows_mouse          off            \
    window_origin_display        default        \
    window_placement             second_child   \
    window_zoom_persist          on             \
    window_shadow                on             \
    window_animation_duration    0.0            \
    window_animation_frame_rate  120            \
    window_opacity_duration      0.0            \
    active_window_opacity        1.0            \
    normal_window_opacity        0.90           \
    window_opacity               off            \
    insert_feedback_color        0xffd75f5f     \
    split_ratio                  0.50           \
    split_type                   auto           \
    auto_balance                 off            \
    top_padding                  12             \
    bottom_padding               12             \
    left_padding                 12             \
    right_padding                12             \
    window_gap                   06             \
    layout                       bsp            \
    mouse_modifier               fn             \
    mouse_action1                move           \
    mouse_action2                resize         \
    mouse_drop_action            swap

echo "yabai configuration loaded.."
```

Здесь вызывается консольная команда в которую передаются параметры. В принципе названия говорят сами за себя.

Сейчас в yabai нет встроенного статус-бара. Есть несколько вариантов, например [spacebar](https://github.com/cmacrae/spacebar),
[SketchyBar](https://github.com/FelixKratz/SketchyBar), я в данный момент использую [simple-bar](https://github.com/Jean-Tinland/simple-bar).
В целом для статус-бара можно уже самому писать плагинчики или команды которые он можем запускать, например я добавил вывод количества непрочитанных сообщений в мессенджерах.

Еще можно упомянуть об одной фиче которую убрали из yabai, это подсвечивание активного окна. Но кажется крайне полезной фичей.
Для этого использую [JankyBorders](https://github.com/FelixKratz/JankyBorders). Кстати, этот же чувак сделал SketchyBar и SketchyVim - это чтобы любое поле в MacOS было как vim редактор :)

### Предсоздание рабочих столов

У меня это так:

```
yabai -m space 1 --label code
rename_space 1 code

yabai -m space --create
yabai -m space 2 --label web
rename_space 2 web

...
```

То есть после загрузки компа загружается yabai и создаются рабочие столы. Сейчас у меня их 9 штук.

Дальше при запуске приложений они автоматом отправляются на нужные рабочие столы, это делается благодаря:

```
# window rules
yabai -m rule --add app="^iTerm2$" space=code
yabai -m rule --add app="^Mail$" space=mail
yabai -m rule --add app="^Safari$" space=web
yabai -m rule --add app=".*Chrome.*" space=web
yabai -m rule --add app="^Arc$" space=web
...
```

Для некоторых приложений отключен менеджмент окон, например для настроек - они откроются как обычное окно:

```
yabai -m rule --add label="System Settings" app="^System Settings$" title=".*" manage=off
yabai -m rule --add label="App Store" app="^App Store$" manage=off
```

Остальное можно посмотреть в моем [конфиге .yabairc](https://github.com/accessd/dotfiles/blob/master/files/yabairc)

## Хоткеи

Половина успеха это конечно хоткеи. В yabai как уже было сказано чаще используется skhd. Рассмотрим конфиг `~/.skhdrc`.
Наиболее используемые фичи по частоте использования:

### Переключение окон

![focusing windows](/assets/blog/yabai/focus_windows.gif)

```
# focus window
alt - x : yabai -m window --focus recent
alt - h : yabai -m window --focus west
alt - j : yabai -m window --focus south
alt - k : yabai -m window --focus north
alt - l : yabai -m window --focus east
alt - z : yabai -m window --focus stack.prev
alt - c : yabai -m window --focus stack.next
```

Чаще я пользуюсь hjkl переключением, т.е. выбираю в одном из четырех направлений.

### Переключение рабочих столов

```
alt - 1 : yabai -m space --focus  1 || skhd -k "ctrl + alt + cmd - 1"
alt - 2 : yabai -m space --focus  2 || skhd -k "ctrl + alt + cmd - 2"
alt - 3 : yabai -m space --focus  3 || skhd -k "ctrl + alt + cmd - 3"
alt - 4 : yabai -m space --focus  4 || skhd -k "ctrl + alt + cmd - 4"
и так далее
```

### Развернуть окно на максимум

Это часто использую, разворачиваю и сворачиваю окно на весь стол этим хоткеем.
На первой гифке есть.

```
# toggle window fullscreen zoom
alt - f : yabai -m window --toggle zoom-fullscreen
```

### Замена окон местами и их перемещение в рамках одного рабочего стола

В первой гифке это было продемонстрированно.

```
# swap window
shift + alt - x : yabai -m window --swap recent
shift + alt - h : yabai -m window --swap west
shift + alt - j : yabai -m window --swap south
shift + alt - k : yabai -m window --swap north
shift + alt - l : yabai -m window --swap east
```

```
# move window
shift + cmd - h : yabai -m window --warp west
shift + cmd - j : yabai -m window --warp south
shift + cmd - k : yabai -m window --warp north
shift + cmd - l : yabai -m window --warp east
```

У меня это alt + номер стола, так мне удобнее. Вообще в MacOS по дефолту столы перключаются через ctrl + номер, можно и так сделать.


### Отправить окно на другой рабочий стол

Не просто отправить, а еще и переключиться на него.

```
shift + cmd - 1 : yabai -m window --space  1 && yabai -m space --focus 1
shift + cmd - 2 : yabai -m window --space  2 && yabai -m space --focus 2
shift + cmd - 3 : yabai -m window --space  3 && yabai -m space --focus 3
и так далее
```

Тоже использую чтобы окна перемещать между столами, например окно браузера отправить на стол с терминалом. Отображено на первой гифке.


### Изменить размер окна

Тут понятно, уже с помощью awsd меняем размер окна.


```
# increase window size
shift + alt - a : yabai -m window --resize left:-50:0
shift + alt - s : yabai -m window --resize bottom:0:50
shift + alt - w : yabai -m window --resize top:0:-50
shift + alt - d : yabai -m window --resize right:50:0

# decrease window size
shift + cmd - a : yabai -m window --resize left:50:0
shift + cmd - s : yabai -m window --resize bottom:0:-50
shift + cmd - w : yabai -m window --resize top:0:50
shift + cmd - d : yabai -m window --resize right:-50:0
```


### Floating окна

![floating window](/assets/blog/yabai/floating_window.gif)

Эта штука полезна для окон которые имеют фиксированные размеры, оно становится вне сетки и его можно тоже перемещать отдельными хоткеями.

```
# float / unfloat window and restore position
alt - t : yabai -m window --toggle float && yabai -m window --grid 4:4:1:1:2:2

# move window
shift + ctrl - a : yabai -m window --move rel:-20:0
shift + ctrl - s : yabai -m window --move rel:0:20
shift + ctrl - w : yabai -m window --move rel:0:-20
shift + ctrl - d : yabai -m window --move rel:20:0
```

### Вертеть окнами

![rotate tree](/assets/blog/yabai/rotate_tree.gif)

Можно вертеть по кругу или по осям x/y.

```
# rotate tree
alt - r : yabai -m space --rotate 90

# mirror tree y-axis
alt - y : yabai -m space --mirror y-axis

# mirror tree x-axis
alt - x : yabai -m space --mirror x-axis
```

Весь конфиг [здесь](https://github.com/accessd/dotfiles/blob/master/files/skhdrc)


## Заключение

На первый взгляд yabai кажется сложным, но стоит немного привыкнуть к этому подходу и жить ставновится лучше.
Если вы любите управляться без мышки, то вам точно стоит попробовать. Можно запускать yabai через brew services при необходимости и пробовать.

В планах еще написать про жизнь в MacOS без мышки, а конкрентно про SpaceLauncher, Raycast и другие штуки ✌️
