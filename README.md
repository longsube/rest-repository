# Hướng dẫn chạy chương trình trên Remote Server

## 1.Cài đặt PostgreSQL
  - Cài đặt gói
  	```sh
  	apt-get install postgresql postgreqsl-contrib
  	```
  - Tạo user và db cho root
 	```sh
	su - postgres
	createuser root -P
	createdb root
	```

  - Kiểm tra DB
  	```sh
  	su - root
	#psql
	\conninfo
	You are connected to database "root" as user "root" via socket in "/var/run/postgresql" at port "5432".
	\q
	```

## 2.Cài đặt Apache2
  - Cài đặt gói
  	```sh
  	apt-get install apache2 -y
  	```
  - Cài đặt gói mod-proxy-uwsgi cho apache
	```sh
	apt-get install -y libapache2-mod-proxy-uwsgi
	a2enmod proxy
	service apache2 restart
	```

  - Tạo file cấu hình /etc/apache2/sites-available/items-rest.conf
	```sh
	Listen 0.0.0.0:80
	# Option để apache2 giao tiếp với uwsgi thông qua socket (chú ý từ apache 2.4.9 trở lên mới hỗ trợ)
	ProxyPass / unix:/var/www/html/items-rest/socket.sock|uwsgi://127.0.0.1:5000/
	# Option để apache2 giao tiếp với uwsgi thông qua port
	Listen 0.0.0.0:80
	ProxyPass / uwsgi://127.0.0.1:5000/
	```
  - Tạo soft link cho file cấu hình vừa tạo 
	```sh
	ln -s /etc/apache2/sites-available/items-rest.conf /etc/apache2/sites-enabled/items-rest.conf
	service apache2 restart
	```

## 2. Tải chương trình
  - Cài đặt git
  	```sh
  	apt-get update
  	apt-get install git -y
  	```
  - Clone chương trình
  	```sh
  	git clone https://github.com/longsube/rest-repository.git
  	mkdir /var/www/html/items-rest
  	cp /root/rest-repository/* /var/www/html/items-rest
  	```
vim uwsgi.ini
[uwsgi]
base = /var/www/html/items-rest
app = run
module = %(app)
home = %(base)/venv
pythonpath = %(base)
socket = %(base)/socket.sock
chmod-socket =777
processes = 8
threads = 8
harakiri = 15
callable = app
logto = /var/www/html/items-rest/log/%n.log

Tạo service cho uwsgi
Ubuntu14.04 dùng Upstart (init)
vim /etc/init/uwsgi.conf

uwsgi --master --emperor /var/www/html/items-rest/uwsgi.ini --die-on-term --uid root --gid root --logto /var/www/html/items-rest/emperor.log
description "uWSGI items rest"
export DATABASE_URL=postgres://root:a@localhost:5432/root
start on runlevel [2345]
stop on runlevel [!2345]
respawn
exec /var/www/html/items-rest/venv/bin/uwsgi --master --emperor /var/www/html/items-rest/uwsgi.ini --die-on-term --uid root --gid root --logto /var/www/html/items-rest/emperor.log

service uwsgi start

Ubuntu16.04 dùng systemd
vim /etc/systemd/system/uwsgi_item_rest.service

[Unit]
Description=uWSGI items rest


[Service]
Environment=DATABASE_URL=postgres://root:a@localhost:5432/root
ExecStart=/var/www/html/items-rest/venv/bin/uwsgi --master --emperor /var/www/html/items-rest/uwsgi.ini --die-on-term --uid root --gid root --logto /var/www/html/items-rest/emperor.log
Restart=always
KillSignal=SIGQUIT
Type=notify
NotifyAccess=all

[Install]
WantedBy=multi-user.target

systemctl start uwsgi_item_rest
systemctl enable uwsgi_item_rest




Cài đặt apache phiên bản 2.4.9 trở lên (hỗ trợ Proxypass socket)
add-apt-repository ppa:ondrej/apache2
apt-get update
apt-get install --only-upgrade apache2

Cài đặt gói mod-proxy-uwsgi cho apache
apt-get install -y libapache2-mod-proxy-uwsgi
a2enmod proxy


vi /etc/apache2/sites-available/items-rest.conf
Listen 0.0.0.0:80
ProxyPass / unix:/var/www/html/items-rest/socket.sock|uwsgi://127.0.0.1:5000/
ln -s /etc/apache2/sites-available/items-rest.conf /etc/apache2/sites-enabled/items-rest.conf
service apache2 restart




vim uwsgi.ini
[uwsgi]
base = /var/www/html/items-rest
app = run
module = %(app)
home = %(base)/venv
pythonpath = %(base)
socket =  127.0.0.1:5000
chmod-socket =777
processes = 8
threads = 8
harakiri = 15
callable = app
logto = /var/www/html/items-rest/log/%n.log

vim /etc/apache2/sites-available/items-rest.conf

ln -s /etc/apache2/sites-available/items-rest.conf /etc/apache2/sites-enabled/items-rest.conf
service apache2 restart


apt-get install nginx -y
server {
        listen 80;
        real_ip_header X-Forwarded-For;
        set_real_ip_from 127.0.0.1;
        server_name localhost;

        location / {
        include uwsgi_params;
        uwsgi_pass unix:/var/www/html/items-rest/socket.sock;
        uwsgi_modifier1 30;
        }

        error_page 404 /404.html;
        location = /404.html {
        root /usr/share/nginx/html;
        }

        error_page 500 502 503 504 /50x.html;
        location = /50x.html {
        root /usr/share/nginx/html;
        }
}

ln -s /etc/nginx/sites-available/items-rest.conf /etc/nginx/sites-enabled/items-rest.conf
rm /etc/nginx/sites-enabled/default
service nginx restart




