#  Домашнее задание  
Размещаем свой RPM в своем репозитории

* Описание/Пошаговая инструкция выполнения домашнего задания:
* создать свой RPM (можно взять свое приложение, либо собрать к примеру апач с определенными опциями);
* создать свой репо и разместить там свой RPM;
реализовать это все либо в вагранте, либо развернуть у себя через nginx и дать ссылку на репо.
* реализовать дополнительно пакет через docker

## Создадим свой RPM пакет
*Для примера возьмем пакет NGINX и соберем его с поддержкой openssl.*

*Первым делом устанавливаем пакеты:*

``sudo su``

``cd /root ``

``[root@rpmlab ~]# yum install -y redhat-lsb-core wget rpmdevtools rpm-build createrepo yum-utils nano gcc wget lynx ``

``[root@rpmlab ~]# yum update``


*Загрузим SRPM пакет NGINX для дальнейшей работы над ним:*

``[root@rpmlab ~]# wget https://nginx.org/packages/centos/7/SRPMS/nginx-1.14.1-1.el7_4.ngx.src.rpm``

*При установке такого пакета в домашней директории создается древо каталогов для
сборки:*

``[root@rpmlab ~]# rpm -i nginx-1.14.1-1.el7_4.ngx.src.rpm``

*Также нужно скачать и разархивировать последние исходники для openssl - они
потребуются при сборке*

``[root@rpmlab ~]# wget https://www.openssl.org/source/openssl-1.1.1o.tar.gz --no-check-certificate``
>     --2022-06-01 05:19:07--  https://www.openssl.org/source/openssl-1.1.1o.tar.gz
>     Resolving www.openssl.org (www.openssl.org)... 184.51.226.32, 2a02:26f0:41:681::c1e, 2a02:26f0:41:688::c1e
>     Connecting to www.openssl.org (www.openssl.org)|184.51.226.32|:443... connected.
>     WARNING: cannot verify www.openssl.org's certificate, issued by ‘/C=US/O=Let's Encrypt/CN=R3’:
>     Issued certificate has expired.
>     HTTP request sent, awaiting response... 200 OK
>     Length: 9856386 (9.4M) [application/x-gzip]
>     Saving to: ‘openssl-1.1.1o.tar.gz’
>     100%[================================================================================>] 9,856,386   1.55MB/s   in 5.3s
>     2022-06-01 05:19:12 (1.78 MB/s) - ‘openssl-1.1.1o.tar.gz’ saved [9856386/9856386]

``[root@rpmlab ~]# tar -xvf openssl-1.1.1o.tar.gz``

*Заранее поставим все зависимости чтобы в процессе сборки не было ошибок*

``[root@rpmlab ~]# yum-builddep rpmbuild/SPECS/nginx.spec -y``

*Поправим сам spec файл чтобы NGINX собирался с необходимыми нам
опциями:*

``[root@rpmlab ~]# nano rpmbuild/SPECS/nginx.spec``

%build

./configure %{BASE_CONFIGURE_ARGS} \

    --with-cc-opt="%{WITH_CC_OPT}" \
    --with-ld-opt="%{WITH_LD_OPT}" \
    --with-openssl=/root/openssl-1.1.1o 
    #--with-debug


*Теперь можно приступить к сборке RPM пакета:*

``[root@rpmlab ~]# rpmbuild -bb rpmbuild/SPECS/nginx.spec``

>     Wrote: /root/rpmbuild/RPMS/x86_64/nginx-1.14.1-1.el7_4.ngx.x86_64.rpm
>     Wrote: /root/rpmbuild/RPMS/x86_64/nginx-debuginfo-1.14.1-1.el7_4.ngx.x86_64.rpm
>     Executing(%clean): /bin/sh -e /var/tmp/rpm-tmp.TnxPBH
>     + umask 022
>     + cd /root/rpmbuild/BUILD
>     + cd nginx-1.14.1
>     + /usr/bin/rm -rf /root/rpmbuild/BUILDROOT/nginx-1.14.1-1.el7_4.ngx.x86_64
>     + exit 0
>     [root@rpmlab ~]#

*Убедимся что пакеты создались:*

``[root@rpmlab ~]# ll rpmbuild/RPMS/x86_64/``

>     total 4396
>     -rw-r--r--. 1 root root 2007420 Jun  1 06:00 nginx-1.14.1-1.el7_4.ngx.x86_64.rpm
>     -rw-r--r--. 1 root root 2489356 Jun  1 06:00 nginx-debuginfo-1.14.1-1.el7_4.ngx.x86_64.rpm

*Теперь можно установить наш пакет и убедиться что nginx работает*

``[root@rpmlab ~]# yum localinstall -y rpmbuild/RPMS/x86_64/nginx-1.14.1-1.el7_4.ngx.x86_64.rpm``

>       Verifying  : 1:nginx-1.14.1-1.el7_4.ngx.x86_64                                                                           
>     Installed:
>     nginx.x86_64 1:1.14.1-1.el7_4.ngx
>     Complete!
>     [root@rpmlab ~]#

``[root@rpmlab ~]# systemctl start nginx``
``[root@rpmlab ~]# systemctl status nginx``

>     ● nginx.service - nginx - high performance web server
>     Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
>       Active: active (running) since Wed 2022-06-01 06:06:17 UTC; 50s ago

*Далее мы будем использовать его для доступа к своему репозиторию*

## Создадим свой репозиторий и разместим там ранее собранный RPM

*Теперь приступим к созданию своего репозитория. Директория для статики у NGINX по
умолчанию /usr/share/nginx/html. Создадим там каталог repo:*

``[root@rpmlab ~]# mkdir /usr/share/nginx/html/repo``

*Копируем туда наш собранный RPM и, например, RPM для установки репозитория
Percona-Server:*

``[root@rpmlab ~]# cp rpmbuild/RPMS/x86_64/nginx-1.14.1-1.el7_4.ngx.x86_64.rpm /usr/share/nginx/html/repo/``

``[root@rpmlab ~]# wget https://downloads.percona.com/downloads/percona-release/percona-release-0.1-6/redhat/percona-release-0.1-6.noarch.rpm -O /usr/share/nginx/html/repo/percona-release-0.1-6.noarch.rpm``

>     --2022-06-01 06:21:42--  https://downloads.percona.com/downloads/percona-release/percona-release-0.1-6/redhat/percona-release-0.1-6.noarch.rpm
>     Resolving downloads.percona.com (downloads.percona.com)... 162.220.4.221, 162.220.4.222, 74.121.199.231
>     Connecting to downloads.percona.com (downloads.percona.com)|162.220.4.221|:443... connected.
>     HTTP request sent, awaiting response... 200 OK
>     Length: 14520 (14K) [application/octet-stream]
>     Saving to: ‘/usr/share/nginx/html/repo/percona-release-0.1-6.noarch.rpm’
>     100%[================================================================================================================>] 14,520      --.-K/s   in 0.1s
>     2022-06-01 06:21:43 (103 KB/s) - ‘/usr/share/nginx/html/repo/percona-release-0.1-6.noarch.rpm’ saved [14520/14520]

*Инициализируем репозиторий командой:*

``[root@rpmlab ~]# createrepo /usr/share/nginx/html/repo/``
>     Spawning worker 0 with 1 pkgs
>     Spawning worker 1 with 1 pkgs
>     Workers Finished
>     Saving Primary metadata
>     Saving file lists metadata
>     Saving other metadata
>     Generating sqlite DBs
>     Sqlite DBs complete

*Длā прозрачности настроим в NGINX доступ к листингу каталога:*

*В location / в файле /etc/nginx/conf.d/default.conf добавим директиву autoindex on. В
результате location будет выглядеть так:

``[root@rpmlab ~]# nano /etc/nginx/conf.d/default.conf``

>     location / {
>     root /usr/share/nginx/html;
>     index index.html index.htm;
>     autoindex on; 
>     }

*Проверяем синтаксис и перезапускаем NGINX:*

``[root@rpmlab ~]# nginx -t``
>     nginx: the configuration file /etc/>nginx/nginx.conf syntax is ok
>     nginx: configuration file /etc/nginx/nginx.conf test is successful
```[root@rpmlab ~]# nginx -s reload```

*Теперь ради интереса можно посмотреть в браузере или curl-ануть:*

``[root@rpmlab ~]# lynx http://localhost/repo/``


``[root@rpmlab ~]# curl -a http://localhost/repo/``
>     <html>
>     <head><title>Index of /repo/</>     title></head>
>     <body bgcolor="white">
>     <h1>Index of /repo/</>          h1><hr><pre><a href="../">../</a>
>     <a href="repodata/">repodata/</a>                                          01-Jun-2022 06:23                   -
>     <a href="nginx-1.14.1-1.el7_4.ngx.x86_64.rpm">nginx-1.14.1-1.el7_4.ngx.x86_64.rpm</a>                01-Jun-2022 06:20             2007420
>     <a href="percona-release-0.1-6.noarch.rpm">percona-release-0.1-6.noarch.rpm</a>                   11-Nov-2020 21:48               14520
>     </pre><hr></body>
>     </html>
>     [root@rpmlab ~]# 

*Все готово для того, чтобы протестировать репозиторий.*

*Добавим его в /etc/yum.repos.d:*

>     [root@rpmlab ~]#  cat >> /etc/yum.repos.d/otus.repo << EOF
>     >[otus]
>     > name=otus-linux
>     > baseurl=http://localhost/repo
>     > gpgcheck=0
>     > enabled=1
>     > EOF

*Убедимся что репозиторий подключился и посмотрим что в нем есть:*

``[root@rpmlab ~]# yum repolist enabled | grep otus``

>     otus          otus-linux                 2

``[root@rpmlab ~]#  yum list | grep otus
percona-release.noarch``   

>     percona-release.noarch       0.1-6       otus

``[root@rpmlab ~]# ll  /usr/share/nginx/html/repo/``
>     total 1984
>     -rw-r--r--. 1 root root 2007420 Jun  1 06:20 nginx-1.14.1-1.el7_4.ngx.x86_64.rpm
>     -rw-r--r--. 1 root root   14520 Nov 11  2020 percona-release-0.1-6.noarch.rpm
>     drwxr-xr-x. 2 root root    4096 Jun  1 06:23 repodata

*Так как NGINX у нас уже стоит установим репозиторий percona-release:*

``[root@rpmlab ~]# yum install percona-release -y``

*Все прошло успешно. В случае если вам потребуется обновить репозиторий (а это
делается при каждом добавлении файлов), снова то выполните команду createrepo
/usr/share/nginx/html/repo/*
