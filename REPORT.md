# Отчёт по домашней работе

> Заполняйте отчёт по ходу выполнения задания. Для каждой найденной проблемы скопируйте секцию ниже и опишите не только внесённое изменение, но и ход диагностики.

## Проблема 1

### Что было обнаружено

Проверил состояние сервиса, но сервис не запускался и выводил status=203/EXEC
systemd пыталось запустить приложение через ExecStart=/opt/linux-devops-homework/app.py
### Как проводилась диагностика

Посмотрел фактический ExecStart 
Проверил наличие файла и получил No such file or directory
Посмотрел содержимое каталога приложения
Проверил unit-файл и в нем был указан тот же неверный путь к app.py
```bash
# команды диагностики

systemctl show homework-app.service -p ExecStart
ls -l /opt/linux-devops-homework/app.py
ls -la /opt/linux-devops-homework/
find /opt/linux-devops-homework -maxdepth 2 -type f
cat systemd/homework-app.service
```
### В чём была причина

В ExecStart был указан несуществующий файл и из-за этого не получалось выполнить команду запуска и выходила ошибка.


### Что было изменено

В файле systemd/homework-app.service изменил ExecStart=/opt/linux-devops-homework/app.py на ExecStart=/opt/linux-devops-homework/app/server.py
Подготовил каталог и файл

```bash
# изменения и/или команды

sudo mkdir -p /opt/linux-devops-homework/app
sudo cp /opt/linux-devops-homework/server.py /opt/linux-devops-homework/app/server.py
sudo chmod 0755 /opt/linux-devops-homework/app/server.py
sudo cp systemd/homework-app.service /etc/systemd/system/homework-app.service
sudo systemctl daemon-reload
sudo systemctl restart homework-app.service
```
### Почему было выбрано это решение

Ошибка показывала, что systemd не может выполнить команду. Проверка подтвердила, что app.py по указанному пути отсутствует. Поэтому нужно было исправить путь запуска


### Как проверялся результат

После использования следующих команд проблема не возникала


```bash
# команды проверки

systemctl show homework-app.service -p ExecStart
systemctl status homework-app.service
```
---


## Проблема 2
После исправления ExecStart приложение всё ещё не могло нормально запуститься.
Я попробовал запустить его вручную от пользователя homework и получил ошибку PermissionError: [Errno 13] Permission denied:
'/var/lib/linux-devops-homework/startup.log'
### Что было обнаружено

### Как проводилась диагностика
Проверил пользователя, результат показал, что группа существует
Проверил каталог данных и результат был: owner=root group=root permissions=700

```bash
# команды диагностики
id homework
ls -ld /var/lib/linux-devops-homework
```
### В чём была причина
Каталог принадлежал root:root и имел права 700 при этом сервис запускается от пользователя homework
Права 700 дают доступ только владельцу каталога, то есть root. Поэтому пользователь homework не мог создать startup.log.

### Что было изменено
Изменил владельца каталога и его права

```bash
# изменения и/или команды
sudo chown homework:homework /var/lib/linux-devops-homework
sudo chmod 0750 /var/lib/linux-devops-homework
```

### Почему было выбрано это решение?
Сервис должен работать от отдельного пользователя homework, поэтому правильным решением было дать именно этому пользователю необходимые права на каталог данных.
### Как проверялся результат
stat -c 'owner=%U group=%G permissions=%a path=%n' /var/lib/linux-devops-homework
вывод: owner=homework group=homework permissions=750
после этого снова запустил приложение и ошибка исчезлаsudo -u homework /opt/linux-devops-homework/server.py


## Проблема 3

### Что было обнаружено
После успешного запуска сервиса я проверил TCP-сокет
Получил 127.0.0.1:8080
То есть приложение слушало порт 8080 только на loopback-интерфейсе.

### Как проводилась диагностика

Проверил конфигурацию в репозитории
Также посмотрел конфигурацию, которая реально используется системой
В обоих файлах было
[server]
host = 127.0.0.1
port = 8080
```bash
# команды диагностики
cat config/app.conf
cat /etc/linux-devops-homework/app.conf
```


### В чём была причина
В конфигурации приложения параметр host был установлен в 127.0.0.1
Адрес 127.0.0.1 является loopback-адресом, поэтому сервис принимал подключения только с самой виртуальной машины

### Что было изменено


```bash
# изменения и/или команды
изменил
host = 127.0.0.1
на
host = 0.0.0.0
применил конфигурацию sudo cp config/app.conf /etc/linux-devops-homework/app.conf
sudo systemctl restart homework-app.service
```

### Почему было выбрано это решение?
0.0.0.0 означает прослушивание на всех IPv4-интерфейсах системы
По условию задания сервис должен быть доступен не только через localhost, поэтому изменение адреса с 127.0.0.1 на 0.0.0.0 решает именно эту проблему, не меняя порт и другие настройки приложения.
### Как проверялся результат
Повторно проверил сокет
```bash
# команды проверки
sudo ss -ltnp | grep 8080
curl -i http://127.0.0.1:8080/health
```

---



## Дополнительные наблюдения




## Итоговый ход диагностики
Сначала сервис не запускался из-за неправильного ExecStart: systemd пытался запустить несуществующий app.py. После исправления появилась ошибка прав доступа к /var/lib/linux-devops-homework, поэтому я сменил владельца на homework:homework и установил права 750.
Далее через ss обнаружил, что сервис слушает только 127.0.0.1:8080. В конфиге изменил адрес на 0.0.0.0 и перезапустил сервис. После этого проверил работу через systemctl, ss, curl, ps и /proc. Сервис успешно запустился, а /health стал возвращать 200 OK.


## Итоговая проверка

Вставьте полный вывод успешного запуска:

```bash
Linux DevOps Homework Checker

[PASS] systemd ExecStart points to the application
[PASS] service runs as homework:homework
[PASS] state directory ownership and permissions are correct
[PASS] service is enabled
[PASS] service is active
[PASS] running process has the expected UID
[PASS] TCP/8080 listens on 0.0.0.0
[PASS] GET /health returns the expected response
[PASS] fixed systemd unit is saved in the repository
[PASS] fixed application configuration is saved in the repository

Result: 10 passed, 0 failed
```

```text
# вывод check.sh
```
