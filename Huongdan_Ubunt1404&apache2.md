# Hướng dẫn chạy chương trình trên Remote Server (Ubuntu14.04 & Apache2, PostGreSQL)

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
	a2enmod proxy_uwsgi
	service apache2 restart
	```

  - Tạo file cấu hình /etc/apache2/sites-available/items-rest.conf
	```sh
	Listen 0.0.0.0:80
	# Option để apache2 giao tiếp với uwsgi thông qua socket (chú ý từ apache 2.4.9 trở lên mới hỗ trợ)
	ProxyPass / unix:/var/www/html/items-rest/socket.sock|uwsgi://127.0.0.1:5000/
	# Option để apache2 giao tiếp với uwsgi thông qua http
	ProxyPass / uwsgi://127.0.0.1:5000/
	```

  - Tạo soft link cho file cấu hình vừa tạo 
	```sh
	ln -s /etc/apache2/sites-available/items-rest.conf /etc/apache2/sites-enabled/items-rest.conf
	service apache2 restart
	```
  - Nếu muốn giao tiếp với uwsgi qua socket, cần phải upgrade phiên bản apache2 lên mới nhất (> 2.4.9)
  	```sh
  	add-apt-repository ppa:ondrej/apache2
	apt-get update
	apt-get install --only-upgrade apache2
  	```

## 3. Tải chương trình
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

  - Sửa file /var/www/html/items-rest/uwsgi.ini
	```sh 
	[uwsgi]
	base = /var/www/html/items-rest
	app = run
	module = %(app)
	home = %(base)/venv
	pythonpath = %(base)
	# Giao tiếp với apache2 qua unix socket
	socket = %(base)/socket.sock
	# Giao tiếp với Apache2 qua http
	socket = 127.0.0.1:5000
	chmod-socket =777
	processes = 8
	threads = 8
	harakiri = 15
	callable = app
	logto = /var/www/html/items-rest/log/%n.log
	```

  - Tạo file service cho uwsgi `/etc/init/uwsgi.conf` (Ubuntu14.04 dùng Upstart (init))
  	```sh
	description "uWSGI items rest"
	export DATABASE_URL=postgres://root:a@localhost:5432/root
	start on runlevel [2345]
	stop on runlevel [!2345]
	respawn
	exec /var/www/html/items-rest/venv/bin/uwsgi --master --emperor /var/www/html/items-rest/uwsgi.ini --die-on-term --uid root --gid root --logto /var/www/html/items-rest/emperor.log
	```

  - Khởi động uwsgi service
  	```sh
	service uwsgi start
	```
  - Kiểm tra log service uwsgi
  	```sh
  	tailf /var/www/html/items-rest/log/uwsgi.log
  	```
  	Kết quả:
  	```sh
  	*** Operational MODE: preforking+threaded ***
	added /var/www/html/items-rest/ to pythonpath.
	/var/www/html/items-rest/venv/lib/python3.5/site-packages/flask_sqlalchemy/__init__.py:794: FSADeprecationWarning: SQLALCHEMY_TRACK_MODIFICATIONS adds significant overhead and will be disabled by default in the future.  Set it to True or False to suppress this warning.
	  'SQLALCHEMY_TRACK_MODIFICATIONS adds significant overhead and '
	WSGI app 0 (mountpoint='') ready in 1 seconds on interpreter 0x13d4ab0 pid: 3853 (default app)
	*** uWSGI is running in multiple interpreter mode ***
	spawned uWSGI worker 1 (pid: 3853, cores: 8)
	spawned uWSGI worker 2 (pid: 3871, cores: 8)
	spawned uWSGI worker 3 (pid: 3872, cores: 8)
	spawned uWSGI worker 4 (pid: 3873, cores: 8)
	spawned uWSGI worker 5 (pid: 3874, cores: 8)
	spawned uWSGI worker 6 (pid: 3877, cores: 8)
	spawned uWSGI worker 7 (pid: 3879, cores: 8)
	spawned uWSGI worker 8 (pid: 3887, cores: 8)
	```

## 4. Thử nghiệm API
  - Dùng curl gọi 1 GET request tới API
	```sh
	curl http://127.0.0.1:80/items
	```
  - Kết quả nhận được một dict dữ liệu rỗng:
	```sh
	{}
	```

Tham khảo:

[1] - http://uwsgi-docs.readthedocs.io/en/latest/Apache.html

[2] - https://stackoverflow.com/questions/21019717/apache-mod-proxy-uwsgi-and-unix-domain-sockets/28009898

[3] - https://www.digitalocean.com/community/questions/updating-apache-to-the-latest-version




