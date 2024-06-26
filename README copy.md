# Развертывание инфраструктуры web-приложения, с использованием средств автоматизации
  
### Схема инфраструктуры веб-приложения
  
<img src="images/infrastructure_webapp.svg" alt="infrastructure_webapp" width=720>
  
### Описание инфраструктуры
  
Инфраструктура приложения включает в себя:
- Реверс-прокси, на основе Nginx;
- 2-а сервера обрабатывающих запросы пользователей к приложению, на основе Nodejs;
- 2-а сервера баз данных, состоящих их кластера репликации Postgesql.
  
Так же в инфраструктуру включен сервер резервного копирования Barman, на нем же организовано централизованное хранение логов.  

### Выполенение резервного копирования
  
Запуск резервного копирования barman осуществляется командоми: ``` barman switch-wal masterBD ```, ``` barman cron ```, ``` barman chek masterBD ```, ``` barman backup masterBD ```
  
### Восcтановление работы приложения после критических сбоев
  
1. Восстановление Nginx 
  
Восcтановление проксирующего сервера nginx осуществляется командой ``` ansible-playbook ./ansible/playbook.yaml -i ./ansible/inventory -t nginx ```  
Для шифрования трафика используется самоподписанный сертификат и в случаи полной потери сервера nginx потребуется создание надежной группы Диффи-Хеллмана (DH), которая используется при согласовании совершенной прямой секретности с клиентами. Это выполняется командой ``` sudo openssl dhparam -out /etc/nginx/dhparam.pem 4096 ```. Это займет некоторое время, но по окончанию процесса мы получаем группу DH в /etc/nginx/dhparam.pem, которая будет использоваться при конфигурации.
  
2. Восстановление Nodejs
  
Восстановление работоспособности nodejs осуществляется командой ``` ansible-playbook ./ansible/playbook.yaml -i ./ansible/inventory -t nodejs ```
  
3. Восстановление Postgresql
  
Восстановление работоспособности postgresql осуществляется командой ``` ansible-playbook ./ansible/playbook.yaml -i ./ansible/inventory -t postgres ```. В случаи, если проблемы были с мастер-нодой, необходимо будет восстановить базу данных. Резервное копирование осуществляется на сервер backup. Для подключения к серверу по ssh необходимо воспользоваться командой ``` vagrant ssh backup ```. В консоли сервера резревного копирования необходимо воспользоваться командами:
``` barman list-backup masterBD ``` - для вывода списка резервных копий;
``` barman recover masterBD {резервная копия из полученного ранее списка} /var/lib/postgresql/14/main/ --remote-ssh-comman "ssh postgres@192.168.255.100" ``` - для запуска восстановления последней резервной копии.
  
### Логирование
  
Сбор логов осуществляется централизованно на сервер backup. Что бы посмотреть логи необходимо подключиться к серверу по ssh, командой ``` vagrant ssh backup ```. Логи собираются в каталог /var/log/$HOSTNAME. Для просмотра логов можно воспользоваться утилитой tail.