![[Pasted image 20230616135731.png]]


## Настройка телеграма 

Первое что надо сделать - это в поиске телеграма найти отца всех ботов ( @BotFather). Отправляем ему сообщение `/newbot` и следуем его дальнейшим инструкциям. После введения имени и юзернейма, бот вышлет API, который будет в дальнейшем использоваться в настройке рассылки в Zabbix. 
![1](https://git.zabbix.com/projects/ZBX/repos/zabbix/browse/templates/media/telegram/images/1.png?raw=true)
Для того чтобы получать уведомления от бота , нужно узнать свой ID, для этого в поиске ищем IDBot (@myidbot). Пишем ему `/start` , а после `/getid` . ID получен.
![2](https://git.zabbix.com/projects/ZBX/repos/zabbix/browse/templates/media/telegram/images/3.png?raw=true)
Если нужно, чтобы бот отправлял данные в группу, то сначала нужно создать группу, добавить туда IDBot (@myidbot) и в группе написать `/getgroupid@myidbot` (в дальнейшем, не забыть про минус перед id). После этого добавляем нашего созданного бота в группу.
![3](https://git.zabbix.com/projects/ZBX/repos/zabbix/browse/templates/media/telegram/images/9.png?raw=true)
Все данные в телеграме получены.

## Настройка Zabbix
В разделе `Администрирование → Способы оповещения`  импортировать шаблон телеграм-оповещения(если его изначально нет), для того чтобы появился такой способ рассылки. Настраиваем `Способ оповещения` . 
Имя ставим любое. 
Тип - Webhook. 
Параметры:

|               |                                        |
|---------------|----------------------------------------|
| **Message**   | {ALERT.MESSAGE}                        |
| **ParseMode** | Markdown                               |
| **Subject**   | {ALERT.SUBJECT}                        |
| **To**        | ID, который мы получили у IDBot        |
| **Token**     | Токен, который мы получили у BotFather |

![Pasted image 20230616145554](https://github.com/Dzahariy/Zabbix-from-snmp/assets/77004612/ea45c897-18a4-4acc-8563-0adcd0556e16)


Проверим способ оповещения , используя идентификатор чата или идентификатор группы, в зависимости от того, какие требования.

![Pasted image 20230616145453](https://github.com/Dzahariy/Zabbix-from-snmp/assets/77004612/20258bfa-2062-4200-af50-5debfde854a9)


Обязательно, в личном чате с ботом, написать `/start` , иначе выйдет ошибка на подобии:

```
PROBLEM zabbix agent is not available
```

## Настройка пользователей 

В настройках пользователя необходимо перейти на вкладку `Оповещения`  и добавить оповещение через Телеграм.
![Pasted image 20230616150447](https://github.com/Dzahariy/Zabbix-from-snmp/assets/77004612/0f197bda-4684-4df7-b1e1-ff4f48374960)


***все настройки ситуативны за исключением идентификатора***

![Pasted image 20230616150559](https://github.com/Dzahariy/Zabbix-from-snmp/assets/77004612/49936bee-7ed6-48be-a7b8-dc78b0d7b7c4)


Если ранее не работали какие либо оповещения, то необходимо активировать их через `Настройка → Действия → Действия триггеров` 
![Pasted image 20230616151125](https://github.com/Dzahariy/Zabbix-from-snmp/assets/77004612/db1f01af-1825-49f3-9fe9-fba50f4ccda2)

И при необходимости настроить отправку сообщений (кому, когда и т.п)
