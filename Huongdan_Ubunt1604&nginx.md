# Hướng dẫn chạy chương trình trên Remote Server (Ubuntu16.04 & Nginx, PostGreSQL)

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

## 2.Cài đặt Nginx
  - Cài đặt gói
    ```sh
    apt-get install nginx -y
    ```

  - Tạo file cấu hình /etc/nginx/sites-available/items-rest.conf
    ```sh
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
    ```

  - Tạo soft link cho file cấu hình vừa tạo 
    ```sh
    ln -s /etc/nginx/sites-available/items-rest.conf /etc/nginx/sites-enabled/items-rest.conf
    service nginx restart
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

  - Tạo thư mục chứa file log
    ```sh
    mkdir /var/www/html/items-rest/log
    ```

  - Sửa file /var/www/html/items-rest/uwsgi.ini
    ```sh 
    [uwsgi]
    base = /var/www/html/items-rest
    app = run
    module = %(app)
    home = %(base)/venv
    pythonpath = %(base)
    # Giao tiếp với nginx qua unix socket
    socket = %(base)/socket.sock
    chmod-socket =777
    processes = 8
    threads = 8
    harakiri = 15
    callable = app
    logto = /var/www/html/items-rest/log/%n.log
    ```

  - Tạo file service cho uwsgi `/etc/systemd/system/uwsgi_item_rest.service` (Ubuntu16.04 dùng Systemd)
    ```sh
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
    ```

  - Khởi động uwsgi service
    ```sh
    systemctl start uwsgi_item_rest
    ```
  - Cấu hình để uwsgi service chạy khi khởi động máy
    ```sh
    systemctl enable uwsgi_item_rest
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
