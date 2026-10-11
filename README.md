## Создаём виртуальные машины
Создаём каталог, в котором будут храниться настройки виртуальной машины. В каталоге создаём файл с именем Vagrantfile, добавляем в него следующее содержимое:
```shell
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/jammy64"

  config.vm.provider :virtualbox do |v|
    v.memory = 1024
    v.cpus = 2
  end

  boxes = [
    { name: "web", ip: "192.168.56.10" },
    { name: "log", ip: "192.168.56.15" }
  ]

  boxes.each do |opts|
    config.vm.define opts[:name] do |srv|
      srv.vm.hostname = opts[:name]
      srv.vm.network "private_network", ip: opts[:ip]
    end
  end
end
```

Результатом выполнения команды `vagrant up` станут 2 созданные виртуальные машины.

## Настройка виртуальной машины web
Для установки добавим следующий блок кода в наш конфиг:
```shell
      # Провижининг конкретно для web
      if opts[:name] == "web"
        srv.vm.provision "shell", inline: <<-SHELL
          apt-get update
          apt-get install -y nginx
          systemctl enable nginx
          systemctl start nginx
          echo "Hello from WEB (192.168.56.10)" > /var/www/html/index.html
          cp /vagrant/configs/nginx.conf /etc/nginx/nginx.conf
          systemctl restart nginx
        SHELL
      end
```

Здесь мы запускаем команды на `web` по очереди:
- Обновление информации о пакетах
- Установка `nginx`
- Добавление в автозагрузку `nginx`
- Запуск `nginx`
- Прописываем строчку `Hello from WEB (192.168.56.10)` в файл `/var/www/html/index.html` с помощью команды `echo`
- Копируем конфиг `nginx` из локальной папки на хостовой машине в нужную нам папку.

```ad-info
Важное уточнение!
При создании виртуальной машины `vagrant` по умолчанию монтирует папку с проектом в корневую директорию виртуальной машины. Отсюда появляется возможность создать папку я `./configs` в папке с проектом и просто копировать оттуда наш измененный конфиг `nginc.conf`
```

- Перезапускаем nginx.

**Измененная часть конфига `nginx`:**
```shell

        #access_log /var/log/nginx/access.log;
        access_log syslog:server=192.168.56.15:514,tag=nginx_access,severity=info combined;
        error_log /var/log/nginx/error.log;
        error_log syslog:server=192.168.56.15:514,tag=nginx_error;
```
Тут мы настраиваем пересылку логов на сервер `log`.
`tag` нужен для того, чтобы логи записывались в разные файлы
По умолчанию, error-логи отправляют логи, которые имеют severity: error, crit, alert и emerg. Если требуется хранить или пересылать логи с другим severity, то это также можно указать в настройках nginx.
```ad-info
Обратим внимание на то что `error.log` мы пишем и локально и транслируем на удаленный сервер.
```

## Настройка центрального сервера сбора логов.
Все настройки Rsyslog хранятся в файле /etc/rsyslog.conf
Для того, чтобы наш сервер мог принимать логи, нам необходимо внести следующие изменения в
файл:
```shell
      # Провижининг конкретно для log
      if opts[:name] == "log"
        srv.vm.provision "shell", inline: <<-'SHELL'
          set -e
       # Включаем UDP-модуль (Ubuntu 22.04: без пробелов)
          sed -i 's/^#module(load="imudp")/module(load="imudp")/' /etc/rsyslog.conf
          sed -i 's/^#input(type="imudp" port="514")/input(type="imudp" port="514")/' /etc/rsyslog.conf
          sed -i 's/^#module(load="imtcp")/module(load="imtcp")/' /etc/rsyslog.conf
          sed -i 's/^#input(type="imtcp" port="514")/input(type="imtcp" port="514")/' /etc/rsyslog.conf
          echo "\$template RemoteLogs,\"/var/log/rsyslog/%HOSTNAME%/%PROGRAMNAME%.log\"" >> /etc/rsyslog.conf
          echo "*.* ?RemoteLogs" >> /etc/rsyslog.conf
          echo "& ~" >> /etc/rsyslog.conf
       # Перезапуск rsyslog
          systemctl restart rsyslog
        SHELL
      end
```

Что тут происходит:
- С помошью `sed` расскоментируем строки с конфигом где настраиваются порты tcp и udp для приема логов.
- С помошью `echo` в конец файла `/etc/rsyslog.conf` добавляем правила приёма сообщений от хостов:
Данные параметры будут отправлять в папку `/var/log/rsyslog` логи, которые будут приходить от других
серверов. Например, Access-логи nginx от сервера `web`, будут идти в файл
`/var/log/rsyslog/web/nginx_access.log`
- Далее перезапускаем службу rsyslog.

## Готовый vagrantfile

```shell
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/jammy64"

  config.vm.provider :virtualbox do |v|
    v.memory = 1024
    v.cpus = 2
  end

  boxes = [
    { name: "web", ip: "192.168.56.10" },
    { name: "log", ip: "192.168.56.15" }
  ]

  boxes.each do |opts|
    config.vm.define opts[:name] do |srv|
      srv.vm.hostname = opts[:name]
      srv.vm.network "private_network", ip: opts[:ip]

      # Провижининг конкретно для web
      if opts[:name] == "web"
        srv.vm.provision "shell", inline: <<-SHELL
          apt-get update
          apt-get install -y nginx
          systemctl enable nginx
          systemctl start nginx
          echo "Hello from WEB (192.168.56.10)" > /var/www/html/index.html
          cp /vagrant/configs/nginx.conf /etc/nginx/nginx.conf
          systemctl restart nginx
        SHELL
      end
      # Провижининг конкретно для log
      if opts[:name] == "log"
        srv.vm.provision "shell", inline: <<-'SHELL'
          set -e
       # Включаем UDP-модуль (Ubuntu 22.04: без пробелов)
          sed -i 's/^#module(load="imudp")/module(load="imudp")/' /etc/rsyslog.conf
          sed -i 's/^#input(type="imudp" port="514")/input(type="imudp" port="514")/' /etc/rsyslog.conf
          sed -i 's/^#module(load="imtcp")/module(load="imtcp")/' /etc/rsyslog.conf
          sed -i 's/^#input(type="imtcp" port="514")/input(type="imtcp" port="514")/' /etc/rsyslog.conf
          echo "\$template RemoteLogs,\"/var/log/rsyslog/%HOSTNAME%/%PROGRAMNAME%.log\"" >> /etc/rsyslog.conf
          echo "*.* ?RemoteLogs" >> /etc/rsyslog.conf
          echo "& ~" >> /etc/rsyslog.conf
       # Перезапуск rsyslog
          systemctl restart rsyslog
        SHELL
      end
    end
  end
end
```

## Результат
Попробуем несколько раз зайти по адресу http://192.168.56.10
Далее заходим на log-сервер и смотрим информацию об nginx:
● cat /var/log/rsyslog/web/nginx_access.log
● cat /var/log/rsyslog/web/nginx_error.log
Поскольку наше приложение работает без ошибок, файл nginx_error.log не будет создан. Чтобы
сгенерировать ошибку, можно переместить файл веб-страницы, который открывает nginx -
mv /var/www/html/index.nginx-debian.html /var/www/
После этого мы получим 403 ошибку.
Видим, что логи отправляются корректно.
