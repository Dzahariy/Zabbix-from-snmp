# Мониторинг Zabbix-agent на маршрутизаторе Eltex ESR-200 с помощью SNMP/
## Установка клиента SNMP 

```bash 
sudo apt update
```

```bash
sudo apt install snmp snmp-mibs-downloader
```

Пакет `snmp` предоставляет набор инструментов командной строки для отправки запросов SNMP агентам. Пакет `snmp-mibs-downloader` поможет установить файлы информационной базы управления , которая отслеживает сетевые объекты, и управлять ими.

## Установка демона и утилит SNMP

С локального компьютера нужно выполнить вход на ***сервер менеджера*** с помощью вашего пользователя без прав root (например через ssh)

```bash 
sudo apt update
```

```bash
sudo apt install snmp snmp-mibs-downloader 
```

На сервере агента нужно обновить индекс пакетов:

```bash
sudo apt update
```

После этого устанавливаем демон SNMP.

```bash
sudo apt install snmpd
```

## Настройка сервера менеджера SNMP 

Затем нужно изменить один файл, чтобы гарантировать, что инструменты SNMP смогут использовать дополнительные данные MIB, установленные раннее.
На сервере менеджера нужно открыть файл `/etc/snmp/snmp.conf` . Чтобы позволить менеджеру импортировать файлы MIB, закомментируем строку `mibs :`

```
# As the snmp packages come without MIB files due to license reasons,loading 
# of MIBs is disable by default. If you added the MIBs you can reenable
# loading them by commeting out the following line.
# mibs :
```
Сохраняем и закрываем `snmp.conf`, нажав `CTRL+X` , `Y` , при использовании `nano` нажимаем ещё `ENTER`.

## Настройка сервера агента SNMP

Для начала откроем на сервере агента файл конфигурации демона с привилегиями `sudo` :

```bash
sudo nano /etc/snmp/snmpd.conf
```

Внутри этого файла вносим некоторые изменения , которые будут использоваться для начальной загрузки нашей конфигурации, чтобы была возможность управлять ею с другого сервера.
Для начала изменяем директиву `agentAddress` . С текущими настройками она разрешает подключения только с локального компьютера. Закомментируем текущую строчку и раскомментируем строку ниже, которая разрешает все подключения.

```
# Listen for connections from the local system only
# agentAddress udp:127.0.0.1:161
# Listen for connections on all interfaces (both IPv4 *and* IPv6)
agentAddress udp:161,udp6:[::1]:161
```

Далее выполним временную вставку строки `createUser`, которую добавим в конец файла.

```
...
createUser bootstrap MD5 12341234 DES
```

*bootstrap - новый временный пользователь*
*MD5 - тип аутентификации*
*12341234 - пароль*
*DES - протокол конфиденциальности*

Далее, на этом шаге, выдадим привилегии нашему новому временному пользователю bootstrap и еще не созданному пользователю demo .
В этом же файле добавим две новые строчки :

```
rwuser bootstrap priv
rwuser demo priv
```

*priv - использование шифрования*
*demo - новый (пока еще не созданный) пользователь*

После всех манипуляция с файлом, можно сохранять и закрывать его.
Для того чтобы все изменения вступили в силу, необходимо перезапустить службу snmpd на нашем ***сервере агента***:

```bash
sudo systemctl restart snmpd
```

Демон SNMP будет прослушивать подключения к порту :161. Настроим UFW для разрешения подключений с ***сервера менеджера*** к этому порту:

```bash
sudo ufw allow from 192.168.10.2 to any port 161
```

После всех этих действий можно подключиться к серверу агента с сервера менеджера для проверки подключения .

```bash
snmpget -u bootstrap -l authPriv -a MD5 -x DES -A 12341234 -X 12341234 192.168.10.1 1.3.6.1.2.1.1.1.0
```

Строка 1.3.6.1.2.1.1.1.0 - это OID, который отвечает за отображение информации о системе. Он будет возвращать вывод uname -a на удаленной системе.
Результат будет выглядеть следующим образом:
```
SNMPv2-MIB::sysDescr.0 = STRING: Eltex ESR-200 Service Router 1.8.2 build 2 (date 06/11/2019 time 18:45:04)

```

## Настройка стандартной учетной записи пользователя 

Ранее мы давали привилегии пользователю `demo`. Но самого пользователя мы еще не создали. Первого пользователя (bootstrap) мы будем использовать в качестве шаблона для создания нового пользователя demo:
```
snmpusm -u bootstrap -l authPriv -a MD5 -x DES -A 12341234 -X 12341234 agent_server_ip_address create demo bootstrap
```

После успешного выполнения команды, мы получим следующее сообщение:
```bash
User successfully created
```

Теперь у нас есть полностью рабочий пользователь с именем demo на сервере агента.

## Создание файла конфигурации клиента

Данные аутентификации для всех наших команда SNMP будут оставаться статичными для каждого запроса. Чтобы не вводить их каждый раз, мы можем создать файл конфигурации на стороне клиента, который будет содержать учетные данные, которые мы используем для подключения.

```bash
sudo nano /etc/snmp/snmp.conf
```

|  Флаг команды |                                 Описание                                |      Директива snmp.conf     |
|:-------------:|:-----------------------------------------------------------------------:|:----------------------------:|
| -u username   | Имя пользователя SNMP3 для аутентификации.                              | defSecurityName username     |
| -l authPriv   | Уровень безопасности для аутентификации.                                | defSecurityLevel authPriv    |
| -a MD5        | Протокол аутентификации для использования.                              | defAuthType MD5              | 
| -x DES        | Протокол конфиденциальности (шифрования) для использования.             | defPrivType DES              | 
| -A passphrase | Фраза-пароль аутентификации для предоставленного имени пользователя.    | defAuthPassphrase passphrase |
| -X passphrase | Фраза-пароль конфиденциальности из предоставленного имени пользователя. | defPrivPassphrase passphrase |

Используя эту информацию, мы можем создать соответствующий файл snmp.conf

```bash
defSecurityName demo
defSecurutyLevel authPriv
defAuthType MD5
defPrivType DES
def AuthPassphrase 12341234
def AuthPassphrase 12341234
```

После завершения редактирования сохраняем и закрываем файл.
Теперь вместо :

```bash
snmpget -u demo -l authPriv -a MD5 -x DES -A 12341234 -X 12341234 192.168.10.1 sysUpTime.0
```

Мы можем вводить:

```bash
snmpget 192.168.10.1 sysUpTime.0
```

## Удаление учетной записи Bootstrap
На сервере агента откроем файл `/etc/snmp/snmpd.conf` еще раз с привилегиями `sudo`.

```bash
sudo nano /etc/snmp/snmpd.conf
```

Найдем и закомментируем обе строки, которые мы ранее добавляли для ссылки на пользователя bootstrap:

```
...
#createUser bootstrap MD5 12341234 DES
#rwuser bootstrap priv
...
```

Сохраняем и закрываем файл .
Теперь перезапускаем демон SNMP:

```bash
sudo systemctl restart snmpd 
```

Это действие удалит привилегии из этого временного пользователя. А также выполнит рекомендации, согласно которым в файле `snmpd.conf` должны отсутствовать директивы `createUser`.


## Настройка ESR для работы с SNMP
Теперь займемся непосредственно маршрутизатором.
Перед тем как снимать по SNMP с устройства данные, его нужно сначала настроить. Для этого в режиме конфигурирования маршрутизатора выполним следующие действия:

```
snmp-server
snmp-server community mytestcommunity ro
snmp-server host 192.168.10.1
	community mytest community
exit
```

***Первая команда*** просто включает SNMP сервер на устройстве 
***Вторая команда*** задает `community` (сообщество), по сути уникальный идентификатор для доступа по SNMP, это может быть любая строка.
***На третьей строке*** мы указываем хост с которого разрешаем доступ к нашему сообществу, а также адрес на который будут высылаться `traps` (ловушки) при срабатывании событий.

Чтобы разрешить подключение, необходимо добавить правило в межсетевой экран:
```
object-group service SNMP
	description "SNMP"
	port-range 161
exit
```

```
object group network MONITORING_HOST
	description "My LAN"
	ip address-range 192.168.10.1
exit
```

```
security zone-pair trusted self
	rule 12
		description "Доступ SNMP"
		action permit
		match protocol udp
		match source-address MONITORING_HOST
		match destination-address any
		match source-port any
		match destination-port SNMP
		enable
	exit
exit
```

Также нужно сохранить конфигурацию

```
commit
```

```
confirm
```

Проверим статус `UFW`

```bash
sudo ufw status

Status: inactive
```
 Он отключен.

Сейчас надо добавить правило, чтобы не потерять управление над устройством 

```bash
sudo ufw allow proto tcp from 192.168.10.2 to 192.168.10.1 port 22 

Rules update
```

Включаем UFW

```bash
ufw enable

Command may disrupt existing ssh connections. Proceed with operation (y|n)? y

Firewall is active and enabled on system startup

```

Проверяем правило

```bash
sudo ufw status
Status: active 
В                          Действие    Из
-                          --------    --
192.168.10.2 22/tcp       ALLOW       192.168.10.1  

```

Добавим входящее правило по порту 162 для получения ловушек (traps):

```bash
sudo ufw allow proto tcp from 192.168.0.254 to 192.168.0.1 port 162
```

Как итог:

```bash
sudo ufw status

Status: active

В                          Действие    Из
-                          --------    --
192.168.10.2 22/tcp       ALLOW       192.168.10.1  
192.168.10.1 162/tcp      ALLOW       192.168.0.254
```

## Работа с Mib'ами 
Для начала скачиваем с официального сайта производителя архив, с mib-файлами.

Теперь создаем папку на сервере и помещаем туда наши _мибы_ :

```bash
mkdir -p /usr/local/share/snmp/mibs
```

Как и ранее, заходим в файл конфигурации /etc/snmp/snmp.conf и редактируем его

```
# As the snmp packages come without MIB files due to license reasons, loading 
# of MIBs is disabled by default. If you added the MIBs you can reenable 
# loading them by commenting out the following line. 
mibs ALL 
mibdirs +/usr/local/share/snmp/mibs
```

Теперь делаем запрос по алиасу:

```bash
snmpwalk -v2c -c ZABBIX_MERIA 192.168.0.1 memTotalReal.0

Bad operator (INTEGER): At line 73 in /usr/share/snmp/mibs/ietf/SNMPv2-PDU

UCD-SNMP-MIB::memTotalReal.0 = INTEGER: 3859328 kB
```

Выходит ошибка:
```bash
Bad operator (INTEGER): At line 73 in /usr/share/snmp/mibs/ietf/SNMPv2-PDU
```

В целом это поправимо. Нам нужно взять исправление [отсюда](https://pastebin.com/raw/p3QyuXzZ) и заменить содержимое файла: 

```
usr/share/snmp/mibs/ietf/SNMPv2-PDU
```

Теперь ошибок быть не должно.


## OID 
В ESR-200 не так много OID'ов для снятия данных, Полный список можно получить с помощью команды:

```bash
snmpwalk -v 2c -c mytestcommunity   192.168.10.1
```

Выходит относительно не много, учитывая что некоторые повторяются, а некоторые просто бесполезны. Некоторые из значение приведены в таблице ниже.

| Наименование                | Описание (пример)                                                                                                                                                              | 
|-----------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| sysName.0                   | Имя хоста (GW-MAIN)                                                                                                                                                            |
| ifDescr.x(1-8)              | Имя интерфейса (gigabitethernet 1/0/1)                                                                                                                                         |
| ifType.x(1-8)               | Тип интерфейса ( ethernetCsmacd(6) propVirtual(53) )                                                                                                                           |
| ifMtu.x(1-8)                | MTU (1500)                                                                                                                                                                     |
| ifSpeed.x(1-8)              | Скорость в байтах (1000000000)                                                                                                                                                 |
| ifPhysAddress.x(1-8)        | MAC-адрес                                                                                                                                                                      |
| ifAdminStatus.x(1-8)        | Статус порта, он может быть включен (1), выключен (2) или в режиме теста (3) Порт будет в статусе выключен/down (2) если в конфигурации для интерфейса прописано «no shutdown» |
| ifOperStatus.x(1-8)         | Рабочий статус порта, показывает подключен ли кабель и работоспособен ли порт для передачи данных                                                                              |
| ifInOctets.x(1-8)           | Количество полученных октетов                                                                                                                                                  |
| ifInDiscards.x(1-8)         | Количество входящих отброшенных пакетов                                                                                                                                        |
| ifInErrors.x(1-8)           | Количество входящих пакетов с ошибкой                                                                                                                                          |
| ifOutOctets.x(1-8)          | Количество отправленных октетов                                                                                                                                                |
| ifOutDiscards.x(1-8)        | Количество исходящих отброшенных пакетов                                                                                                                                       |
| ifOutErrors.x(1-8)          | Количество исходящих пакетов с ошибкой                                                                                                                                         |
| ifAlias.x(1-8)              | Описание интерфейса                                                                                                                                                            |
| entPhysicalSerialNum.680000 | Серийный номер устройства                                                                                                                                                      |
| laLoad.1                    | Нагрузка на CPU за последнюю минуту                                                                                                                                            |
| laLoad.2                    | Нагрузка на CPU за последние 5 минут                                                                                                                                           |
| laLoad.3                    | Нагрузка на CPU за последние 15 минут                                                                                                                                          |

*Запись вида `ifOutOctets.x(1-8)` обозначает что вместо `x` нужно поставить цифру, номер интерфейса.


## Установка пакетов Zabbix

На [официальном сайте](https://www.zabbix.com/ru/download?zabbix=6.4&os_distribution=ubuntu&os_version=22.04&components=server_frontend_agent&db=mysql&ws=apache) скачиваем и устанавливаем пакеты zabbix нужной версии, с нужными параметрами.

```bash
wget https://repo.zabbix.com/zabbix/6.4/ubuntu/pool/main/z/zabbix-release/zabbix-release_6.4-1+ubuntu22.04_all.deb

dpkg -i zabbix-release_6.4-1+ubuntu22.04_all.deb

apt update
```

В данном случае будет установка версии zabbix 6.4, Дистрибутив Ubuntu, Версия ОС 22.04(Jammy), Компонент Server, Frontend, Agent,  С базой данных MySQL и веб сервером Apache.

```bash
sudo apt install zabbix-server-mysql zabbix-frontend-php zabbix-apache-conf zabbix-sql-scripts zabbix-agent
```

Создадим базу данных. Выполним на хосте, где будет распологаться база данных, следующие команды:
```bash
mysql -uroot -p
password 
mysql> create database zabbix character set utf8mb4 collate utf8mb4_bin;  
mysql> create user zabbix@localhost identified by 'password';  
mysql> grant all privileges on zabbix.* to zabbix@localhost;  
mysql> set global log_bin_trust_function_creators = 1;  
mysql> quit;
```

На хосте zabbix сервера импортируем начальную схему и данные. Нам предложат ввести недавно созданный пароль.

```bash
zcat /usr/share/zabbix-sql-scripts/mysql/server.sql.gz | mysql --default-character-set=utf8mb4 -uzabbix -p zabbix
```

Выключаем опцию log_bin_trust_function_creators после импорта базы данных

```bash
mysql -uroot -p  
password  
mysql> set global log_bin_trust_function_creators = 0;  
mysql> quit;
```

Настроим базу данных для сервера Zabbix, отредактировав файл /etc/zabbix/zabbix_server.conf

```bash
DBPassword=password
```

Теперь запускаем процессы Zabbix сервера и агента

```bash
systemctl restart zabbix-server zabbix-agent apache2
systemctl enable zabbix-server zabbix-agent apache2
```
Если устанавливать так, как сказано в инструкции на официальном сайте, то проблем и ошибок быть не должно.


## Вход и настройка

После того, как все установили , входим в веб интерфейс.
Вводим в адресную строку браузера http://host/zabbix (вместо host наш ip-адрес. Мы переходим на страницу мастера установки веб интерфейса.
Выбираем язык. На следующей странице проверяем что соблюдены все требования программного обеспечения. Далее настраиваем соединение с базой данных ( которую мы ставили вместе с Zabbix), она должна быть уже создана. Вводим все необходимые данные. На странице настроек вводим имя сервера , часовой пояс, тему оформления интерфейса. Далее проверяем результат настроек.

![Приветствие](https://www.zabbix.com/documentation/current/assets/en/manual/quickstart/login.png)


После настроек, перед нами появится экран приветствия. Вводим имя пользователя :
Admin, а пароль: zabbix. Мы войдем под Супер-Администратором.

### Добавление пользователей
Для просмотра всех пользователей и информации о них перейдите в `Администрирование _→_ Пользователи` 
Для того чтобы создать пользователя, нужно нажать соответствующую кнопку -`Создать пользователя` . По умолчанию новый пользователь не имеет прав доступа к узлам сети. Для того чтобы это исправить, нужно выдать ему соответствующие права. Для этого нажмем на имя группы в колонке `Группы`. В этом диалоге перейдем на вкладку `Права`. Далее выбираем группу пользователей `Linux servers` для того чтобы был доступ на чтение этой группы. Нажимаем `Выбрать` и назначаем права доступа. В диалоге свойств группы пользователей, нажимаем `Обновить` .

### Создание узла сети
Изначально существует один предустановленный узел сети, называемый `Zabbix server` . Для добавления нового узла сети нужно перейти на вкладку `Настройка → Узлы сети` и нажать на кнопку `Создать`, после нажатия появится диалог настройки узла сети.
Обязательный поля для ввода отмечены красными звездочками. Т.е как минимум нужно вводить `Имя узла сети` , `Группы` и `Интерфейсы: IP-адрес`, остальные опции можно оставить по умолчанию. Нажимаем кнопку `Добавить` , новый узел должен быть виден в списке узлов сети

## SNMP - интерфейс в Zabbix
Не уходя со вкладки `Узлы сети` заходим в раздел настройки узла. Ниже раздела `Интерфейсы` выберем SNMP и развернем его свойства.
В поле SNMP введем адрес маршрутизатора . Версия SNMP - выбираем вторую (SNMPv2). В SNMP community вводим то сообщество, которое мы создавали во время настройки ESR для работы с SNMP. 
Общая картина должна выглядеть следующим образом:
![Pasted image 20230614094049](https://github.com/Dzahariy/Zabbix-from-snmp/assets/77004612/636ed8de-9dae-42b8-b177-455442906ad8)

Нажимаем `Обновить`. Если все прошло удачно, то в строчке с именем маршрутизатора загорится зеленая кнопка :

### Шаблоны
Открываем раздел `Настройки`, выбираем `Шаблоны` нажимаем `Импорт` . Откроется окно, в котором нужно выбрать файл который мы будем импортировать. Файл должен быть с расширением **.yaml** .
Пример шаблона можно взять по [этой ссылке](https://eltexcm.ru/stati-dlya-tovarov/primer-shablona-zabbix-5.2-esr-21.html). Переходим на сайт, копируем весь шаблон, вставляем в текстовый документ и называем `tample.yaml`

### Создание нового шаблона 
На странице `Шаблоны` нажимаем `Создать`.  Заполняем поля:
**Имя шаблона**- Eltex ESR

**Группы**- Templates/Network devices

**Описание** – введите какой-нибудь текст

Нажмем `Добавить` 

### Привязка шаблона

После создания и импорта  шаблона, его нужно привязать к хосту, для того чтобы Zabbix смог начать загружать данные с устройства по протоколу SNMP. 
Откроем окно хоста нашего устройства. В поле `Шаблоны` начнем вводить нужный нам шаблон, выберем Eltex ESR-21 (тот, который импортировали) из списка и нажмем `Обновить`.

![Pasted image 20230614104257](https://github.com/Dzahariy/Zabbix-from-snmp/assets/77004612/11591574-d48a-4452-81e5-1335c1e97eaa)


## Элементы мониторинга
Существует 2 способа добавления элементов - ручной и с помощью обнаружения.
### Ручное добавление элементов
Открываем созданный шаблон и выбираем пункт - `элементы данных` . Нажимаем на кнопку `Создать элемент данных`
Начинаем заполнять:
**Имя** – HostName

**Тип** – SNMP agent

**Ключ** - hostname_snmpv2

**Тип информации** - Text

**SNMP OID** - .1.3.6.1.2.1.1.5.0

**Интервал обновления** – 1d 

**Описание** – Имя устройства

_Поле `Ключ` должно быть уникальным для всех шаблонов и элементов других устройств_
Нажимаем `Добавить` . В списке элементов появится новый
В течении минуты Zabbix получит новое значение для вновь созданного элемента и полученные с устройства данные отобразятся в списке
Можно также добавить следующие параметры:

|                                                        |                    |
|--------------------------------------------------------|--------------------|
| Описание устройства, с версией прошивки                | .1.3.6.1.2.1.1.1.0 |
| Время работы с последнего выключения                   | .1.3.6.1.2.1.1.3.0 |
| Контактные данные системного администратора устройства | .1.3.6.1.2.1.1.4.0 |
| Расположение устройства                                | .1.3.6.1.2.1.1.6.0 |


## Автоматическое добавление элементов к хосту 
Для автоматизации некоторых процессов, существует раздел `Обнаружение`, с помощью предварительно заданных правил можно сразу получить список всех интерфейсов и создать для них соответствующие элементы и триггеры.
Для этого откроем `Настройка → Шаблоны` и нажмем на `Обнаружение` . Нажмем на кнопку `Создать правило обнаружения` . Заполняем форму:
**Имя** - IF-MIB Interface

**Тип** – SNMP Agent

**Ключ** - ifmib_snmpv2

**SNMP OID**

```bash

discovery[{#IFNAME},1.3.6.1.2.1.31.1.1.1.1,  {#IFALIAS},1.3.6.1.2.1.31.1.1.1.18, {#IFDESCR},1.3.6.1.2.1.2.2.1.2, {#IFTYPE},1.3.6.1.2.1.2.2.1.3, {#IFADMINSTATUS},1.3.6.1.2.1.2.2.1.7, {#IFOPERSTATUS},1.3.6.1.2.1.2.2.1.8]

```

Интервал обновления - 1m (потом можно изменить, т.к смысла в частом обнаружении нет)
Нажимаем `Добавить`. Будет добавлено новое правило обнаружения. Но само по себе правило не добавит элементы интерфейса к нашему хосту, нам нужно с помощью отдельных правил показывать Zabbix как создавать элементы.

Откроем окно с правилами обнаружений нашего шаблона и кликнем на ссылке `Прототипы элементов данных`
Нажмем на кнопку `Создать прототип элементов данных`. Заполняем форму:

**Имя** - Входящий трафик на {#IFDESCR}

**Тип** – SNMP agent

**Ключ** - if_in_nps_snmpv2_[{#IFDESCR}]

**Тип информации** – Numeric (unsigned)

**SNMP OID** - .1.3.6.1.2.1.2.2.1.10.{#SNMPINDEX}

**Единицы измерения** - bps

**Интервал обновления** – 1m

**Период хранения истории** – 90d

Переходим на вкладку `Предобработка`. Нажимаем `Добавить` выберем `Изменение в секунду`. Нажимаем `Добавить` выбираем  `Пользовательский множитель`  и в поле вводим 8. Нажимаем `Обновить`
После некоторого количества времени все интерфейсы появятся в списке элементов нашего маршрутизатора. Зайдем в `Мониторинг → Последние данные`. В фильтрах выберем наше устройство . В загруженном списке будут отображаться данные об объеме данных передаваемых через все интерфейсы устройства (возможно придется подождать пока не будут загружены первые данные).

