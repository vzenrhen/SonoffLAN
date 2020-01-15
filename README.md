# Sonoff

Компонент для работы с устройствами **eWeLink** по локальной сети. Устройства 
должны быть обновлены на прошивку 3й версии. В локальной сети должен 
поддерживаться **Multicast**.

Основные моменты компонента: 

- работает с оригинальной прошивкой Sonoff, нет необходимости перепрошивать 
  устройства
- работает по локальной сети, нет тормозов китайских серверов
- можно получить список устройств с серверов eWeLink, либо настроить его 
  вручную (список сохраняется локально и может больше не запрашиваться)
- мгновенное получение нового состояния устройств по Multicast (привет 
  Yeelight)
- есть возможность менять тип устройства (например свет или вентилятор), для 
  удобной интеграции в голосовые ассистенты
- есть возможность объединить несколько каналов в один источник света и 
  управлять яркостью

## Примеры конфигов

Минимальный конфиг:

```yaml
sonoff:
  username: mymail@gmail.com
  password: mypassword
```

Расширенный конфиг:

```yaml
sonoff:
  username: mymail@gmail.com
  password: mypassword
  reload: once # загружает конфиг только 1 раз, для обновления необходимо
               # удалить файл `.sonoff.json` и перезапустить HA
  devices:
    1000abcdefg:
      device_class: light
```

Устройства можно задать вручную, без подключения к китайским серверам. Но в 
этом случае нужно знать `devicekey` для каждого устройства.

```yaml
sonoff:
  devices:
    1000abcdefg:
      devicekey: f9765c85-463a-4623-9cbe-8d59266cb2e4
```

Примеры использования `device_class`:

```yaml
sonoff:
  username: mymail@gmail.com
  password: mypassword
  reload: once
  devices:
    1000abcde0: # коридор свет
      device_class: light
    1000abcde1: # детская свет (двойной выключатель, одна люстра)
      device_class:
      - device_class: light
        channels: [1, 2]
    1000abcde2: # туалет свет и вытяжка (двойной выключатель)
      device_class: [light, fan]
    1000abcde3: # спальня свет и подсветка (двойной выключатель)
      device_class: [light, light]
    1000abcde4: # зал три зоны света Sonoff 4CH
      device_class:
      - light # зона 1 (канал 1)
      - light # зона 2 (канал 2)
      - device_class: light # зона 3 (каналы 3 и 4)
        channels: [3, 4]
```

## Параметры:

- **reload** - *optional*  
  `always` - загружать список устройств при каждом старте HA  
  `once` - загрузить список устройств единожды
- **device_class** - *optional*, переопределяет тип устройства (по умолчанию 
  все устройства **sonoff** отображаются как `switch`). Может быть строкой 
  или массивом строк (для многоканальных выключателей). Поддерживает типы:
  `light`, `fan`, `switch`.

## Работа с китайскими серверами

Если в настройках указать `username` и `password` (опционально) - при первом
запуске HA скачает список устройств **eWeLink** с китайских серверов и сохранит
в файле `/config/.sonoff.json` (скрытый файл).

Другие запросы к серверам компонент не делает.

При каждом старте HA этот файл будет обновляться.

Если в настройках указать `reload: once` - список загрузится только один раз. И 
при старте ХА список устройств будет загружаться из локального файла. В этом 
случае, когда у вас появятся новые устройства **eWeLink** - вручную удалите 
файл и перезагрузите ХА.

Список устройств будет загружаться из локального файла даже если убрать 
`username` и `password` из настроек.

## Получение devicekey вручную

При желании ключ устройства можно получить таким способом.

1. Перевести устройство в режим настройки (*на выключателе это долгое 
удерживание одной из кнопок*)
2. Подключиться к Wi-Fi сети `ITEAD-10000`, пароль `12345678`
3. Открыть в браузере `http://10.10.7.1/device`
4. Скопировать полученные `deviceid` и `apikey` (это и есть `devicekey`)
5. Подключиться к своей Wi-Fi сети и настроить Sonoff через приложение eWeLink

## Описание протокола

- Для обнаружения устройств Sonoff используется **Zeroconf** 
(сервис `_ewelink._tcp.local.`)
- В сообщении **Zeroconf** устройство передаёт своё текущее состояние и 
настройки
- Изменение состояния устройства так же передаётся по **Zeroconf** 
(сервис `eWeLink_1000abcdef._ewelink._tcp.local.`)
- Устройство управляется через POST-запросы вида 
`http://{ip}:8081/zeroconf/{command}` с JSON в теле запроса
- В 3й версии прошивки при отключенном режиме DIY - сообщения и управляющие 
комманды шифруются алгоритмом AES 128, где в качестве ключа используется 
`devicekey` 

**Пример:**

```
POST http://192.168.1.175:8081/zeroconf/switches
{
    "sequence": "1570626382", 
    "deviceid": "1000abcdef", 
    "selfApikey": "123", 
    "data": "MpiTz2jyRiIIaEB4z1nv/ZUaJuToGv8N5SY+G/5tDjQ3f+FGZ/2L0vajqzwbcjIS", 
    "encrypt": true, 
    "iv": "SI3QEXgpvuaHM3hL/1f3eg=="
}
```

В `data` закодирована комманда:

```json
{"switches": [{"outlet": 0, "switch": "on"}]}
```

## Полезные ссылки

- https://github.com/mattsaxon/sonoff-lan-mode-homeassistant
- https://blog.ipsumdomus.com/sonoff-switch-complete-hack-without-firmware-upgrade-1b2d6632c01