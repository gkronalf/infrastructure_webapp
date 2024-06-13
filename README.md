# Развертывание инфраструктуры web-приложения, с использованием средств автоматизации
  
### Схема инфраструктуры веб-приложения
  
<img src="images/infrastructure_webapp.svg" alt="infrastructure_webapp" width=720>
  
### Описание инфраструктуры
  
Инфраструктура приложения включает в себя:
- Реверс-прокси, на основе Nginx;
- 2-а сервера обрабатывающих запросы пользователей к приложению, на основе Nodejs;
- 2-а сервера баз данных, состоящих их кластера репликации Postgesql.
  
Так же в инфраструктуру включен сервер резервного копирования Barman, на нем же организовано централизованное хранение логов.  

### Выполнение резервного копирования
  
Запуск резервного копирования barman осуществляется командоми: ``` barman switch-wal masterBD ```, ``` barman cron ```, ``` barman check masterBD ```, ``` barman backup masterBD ```
  
### Восcтановление работы приложения после критических сбоев
  
1. Восстановление Nginx 
  
Восcтановление проксирующего сервера nginx осуществляется командой ``` ansible-playbook ./ansible/playbook.yaml -i ./ansible/inventory -t nginx ```  
Для шифрования трафика используется самоподписанный сертификат и в случаи полной потери сервера nginx потребуется создание надежной группы Диффи-Хеллмана (DH), которая используется при согласовании совершенной прямой секретности с клиентами. Это выполняется командой ``` sudo openssl dhparam -out /etc/nginx/dhparam.pem 4096 ```. Это займет некоторое время, но по окончанию процесса мы получаем группу DH в /etc/nginx/dhparam.pem, которая будет использоваться при конфигурации.
  
2. Восстановление Nodejs
  
Восстановление работоспособности nodejs осуществляется командой ``` ansible-playbook ./ansible/playbook.yaml -i ./ansible/inventory -t nodejs ```
  
3. Восстановление Postgresql
  
Для восстановления работы приложения, при потери мастер-сервера СУБД, не обходимо зайти в консоль подчиненного сервера в кластере. ``` vagrant ssh slaveBD ``` и выполнить команду ``` /usr/lib/postgresql/14/bin/pg_ctl promote -D /etc/postgresql/14/main/ ``` - эта команда переключит роль мастер-сервера на текущий сервер. 
Далее необходимо изменить IP адрес сервера СУБД в файле конфигурации ``` /home/vagrant/nginx_server_project/webserver.js ```,  строку ``` host: '192.168.255.100', ``` нужно заменить на  ``` host: '192.168.255.100', ```, на серверах backEnd01, backEnd02 и выполнить перезапуск службы nodejs на этих серверах.  

  
Восстановление работоспособности кластера postgresql осуществляется командой ``` ansible-playbook ./ansible/playbook.yaml -i ./ansible/inventory -t postgres ```. В случаи, если проблемы были с мастер-нодой, необходимо будет предварительно восстановить базу данных.  
Выполнение данного восстановления потребует приостановку работы приложения и предварительного восстановления базы данных.  
  
Резервное копирование осуществляется на сервер backup. Для подключения к серверу по ssh необходимо воспользоваться командой ``` vagrant ssh backup ```.  
  
После ввода нового сервера 
В консоли сервера резервного копирования необходимо воспользоваться командами:  
``` barman list-backup masterBD ``` - для вывода списка резервных копий;
``` barman recover masterBD {резервная копия из полученного ранее списка} /var/lib/postgresql/14/main/ --remote-ssh-comman "ssh postgres@192.168.255.100" ``` - для запуска восстановления последней резервной копии.  
После завершения восстановления данных требуется перезапуск postgresql командой: ``` sudo systemctl restart postgresql ```, которая выполняется в консоле сервера masterBD.  
  
### Логирование
  
Сбор логов осуществляется централизованно на сервер backup. Что бы посмотреть логи необходимо подключиться к серверу по ssh, командой ``` vagrant ssh backup ```. Логи собираются в каталог /var/log/$HOSTNAME. Для просмотра логов можно воспользоваться утилитой tail.