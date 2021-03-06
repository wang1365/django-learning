# Django application development and deployment

## 1. MySQL support
To connect to mysql, Django depends on mysqlmysqldb lib provided by *[mysqlclient](https://github.com/PyMySQL/mysqlclient-python)*.  
Why django doesn't use **[PyMySQL](https://github.com/PyMySQL/PyMySQL)**? Refer to **[What's the difference between PyMySQL and mysqlclient-python?](https://github.com/PyMySQL/PyMySQL/issues/342)**

#### 1.1 Install mysqlclient 
* For windows, download wheel from *[Unoffical PYPI](http://www.lfd.uci.edu/~gohlke/pythonlibs)* and install it.  
* For Ubuntu, execute the following commands:  
>> `sudo apt-get install python-dev libmysqlclient-dev`   
`pip install mysqlclient`

#### 1.2 Django setting for mysql 
Modify project setting file `setting.py`:  
```python
DATABASES = {
    'default': {
        # 'ENGINE': 'mysql.connector.django',
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'I321761',
        'USER': 'root',
        'PASSWORD': '******',
        'HOST': '10.128.184.199',
        'PORT': '3306',
    },
}
```

## 2. Deployment
We use **uwsgi** to run the application, here we use **[uWSGI](https://uwsgi-docs.readthedocs.io)** as uwsgi server. For a real productive deployment, we also need to use
nginx as a HTTP reverse proxy.

### 2.1 Prerequisites
* **[Python](https://www.python.org/)** 2.7 is installed in Ubuntu
* **[pip](https://pip.pypa.io/en/stable/installing/)** is installed

### 2.2 Install application
* Copy project archive to you ubuntu host.
* Install python libraries that application depends 
    Go to project folder, and execute:  
    `pip install -r requirements.txt`
    
### 2.3 Install uWSGI
For Ubuntu:
* Install c header compiler dependency:  
    `apt-get install build-essential python-dev`
* Install uswgi via pip:   
    `pip install uwsgi`
    * Optionally, you can install *uwsgitop* which is used to monitor uwsgi:  
    `pip install uwsgitop`

### 2.4 Run Django application with uwsgi
Go to django-learning directory and execute one of below commands:
* `uwsgi --http :8080 --wsgi-file djangoproject/wsgi.py`
* `uwsgi --http :9090 -w djangoproject.wsgi`
* `uwsgi -s :9090 -w djangoproject.wsgi -M --pidfile /tmp/uwsgi.pid -p 4 -d uwsgi.log` or  
 `uwsgi --socket :9090 -wsgi djangoproject.wsgi --master --pidfile /tmp/uwsgi.pid --processes --daemonize`
  * -M/--master: Manager worker
  * --pidfile: pid file, with it we can easily to stop uwsgi via:  
  `uwsgi --stop /tmp/uwsgi.pid`
  * -p/--processes: Worker process count
  * -d/--daemonize: Daemonize uWSGI
  * -w/--wsgi: Load a wsgi module
* `uwsgi uwsgi.ini` -- start uWSGI via configuration file.

### 2.5 Install nginx  
Refer to [nginx offical guide](http://nginx.org/en/linux_packages.html).
* Download PGP key for nginx repository signature:  
`wget http://nginx.org/keys/nginx_signing.key`  
`apt-key add nginx_signing.key`

* Add repo
Modify `/et/apt/source.list`, add below 2 lines at the end of file:  
   * For Ubuntu 16.04:  
   `deb http://nginx.org/packages/ubuntu/ xenial nginx`  
   `deb-src http://nginx.org/packages/ubuntu/ xenial nginx`
   * For Ubuntu 14.04:  
   `deb http://nginx.org/packages/ubuntu/ trusty nginx`  
   `deb-src http://nginx.org/packages/ubuntu/ trusty nginx`

* Execute following commands:  
   `apt-get update`  
   `apt-get install nginx`
   
### 2.6 Run Django application with *nginx + uwsgi*
* Modify nginx configuration  
  By default, it locates at `/etc/nginx/`, main configuration is `/etc/nginx/nginx.conf`,
  and sub-configuration is `/etc/nginx/conf.d/default.conf`.  
  Open default.conf and modify `location` like:  
  ```
      location / {
        include /etc/nginx/uwsgi_params;
        uwsgi_pass 127.0.0.1:9090;
      }
  ```
  Additionally, modify `server_name` to IP, like `10.128.184.199`, but not `localhost`.

* Start nginx  
  * Start: `nginx`  
  * Shutdown: `nginx -s stop` or `nginx -s quit`  
  * Reloading configuration: `nginx -s reload`

* Start uWSGI  
  Go to django-learning directory and execute:  
  `uwsgi -s :9090 -w djangoproject.wsgi`
  
### 2.7 Serving static files
* Prerequisites  
Set `STATIC_ROOT` in `setting.py`, and after application is deployed, execute:
    `python manage.py collectstatic`  
Then all static files that applications uses will be copied to `STATIC_ROOT`.

* Use uWSGI  
uWSGI serving static file refers to [here](http://uwsgi-docs.readthedocs.io/en/latest/StaticFiles.html?highlight=static).  
  * Option 1: Add `--static-check` or `--check-static` option when running uWSGI, i.e.:  
`uwsgi -s :9090 -w djangoproject.wsgi --static-check /wxc/workspace/`  
With this option, when server received a http request `"http://hostname:9090/static/admin/favorate.icon"`,
uWSGI will check `static/admin/favorate.icon` under `/wxc/workspace/` first.
  
  * Option 2: Add `--static-map` in running command:  
  `uwsgi -s :9090 -w djangoproject.wsg --static-map /static=/wxc/workspace/static`  
With this option, when server received a http request `"http://hostname:9090/static/favorate.icon"`,
uWSGI will return `favorate.icon` under `/wxc/workspace/static`.


### 2.8 Using `virtualenv`
* Install `virutualenv`  
`pip install virutalenv`

* Deploy application  
  * Create virtual python environment  
  `virualenv .env` -- ".env" folder is created, which contains a isolated python.
  * Activate virtual environment  
  `source .env/bin/activate`  
  (If you want to quit the virtual env, execute `deactivate`)
  * Install dependency libraries   
  `cd django-learning`  
  `pip install -r requirements.txt`  
  * Collect static files  
  `python manage.py collectstatic`
  * Start uwsgi  
  `uwsgi uwsgi.ini`


# 3. Database Modeling

## 3.1 MySQL model tool
You can use model tool of MySQL workbench to create the model file and import to your database.  
Refer to [MySQL modeling](https://dev.mysql.com/doc/workbench/en/wb-data-modeling.html).

## 3.2 Django database migration 
Use `manage.py` to manage database model.
* Before first run application, we need map django models to database table. 
    `python manage.py migrate [app]`
* After django model changes
    * Create migrations  
    `python manage.py makemigrations [app]`
    * View one migration  
    `python manage.py sqlmigrate [app] [000x]`
    * Implement all or one migration  
    `python manage.py migrate [app] [000x]`
    * Create super user  
    `python manage.py createsuperuser`
    
* Make application's model administrated in `"django.contrib.admin"`  
Register model in app's `admin.py`, like:  
```python
from django.contrib import admin
from .models import Country, User

admin.site.register((Country, User))
```
    
# 4. Redis support
* Install *`django-redis-cache`*  
`pip install django-redis-cache`

* Django setting  
Modify `settings.py`:  
```python
CACHES: {
    'default': {
        'BACKEND': 'redis_cache.RedisCache',
        'LOCATION': [
            '10.128.184.199:6379',
        ],
        'OPTIONS': {
            'DB': 1,
            'PASSWORD': 'yadayada',
            'PARSER_CLASS': 'redis.connection.HiredisParser',
            'CONNECTION_POOL_CLASS': 'redis.BlockingConnectionPool',
            'CONNECTION_POOL_CLASS_KWARGS': {
                'max_connections': 50,
                'timeout': 20,
            },
            'MAX_CONNECTIONS': 1000,
            'PICKLE_VERSION': -1,
        },
    }
}
```

* Read & Write to Redis cache
```python
from django.core.cache import cache
from .models import User

def get_user(req):
    id = req.GET.get('id')
    key = 'myapp.user.%d'%id
    user = cache.get(key)
    if not user:
        user = User.objects.get(id=id)
        cache.set(key, user)
    
    return HttpResponse(user)
```

# 5. Docker support
### 5.1 Install docker CE(Communication Edition) in Ubuntu
Refer to *[Docker for Ubuntu](https://www.docker.com/docker-ubuntu)*.  
Here we use Ubuntu Xenial 16.04 (LTS).

* If you have old version and you want to remove it  
 `$ sudo apt-get remove docker docker-engine docker.io`

* Setup the repository

  * Update `apt` package index: `$ sudo apt-get update`
  * Allow `apt` to use a repository over HTTPS:
  ```
  $ sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    software-properties-common
  ```
  * Add docker official GPG key:  
  `$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -`
  
  * Setup the `stable` repository  
  ```
  $ sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
  ```
* Install Docker CE
  * Update `apt` package index
  `$ sudo apt-get update`
  * Install the latest version of Docker CE  
  `$ sudo apt-get install docker-ce` OR  
  `$ sudo apt-get install docker-ce=<VERSION>`
  * Verify installation by running `hello-world` image  
  `$ sudo docker run hello-world`
  
 * Configure Docker to start on boot  
 `$ sudo systemctl enbale docker`
 * Start docker daemon  
 `$ sudo systemctl start docker` OR  
 `$ sudo service docker start`
 * Set HTTP/HTTPS proxy
   * `$ mkdir -p /etc/systemd/system/docker.service.d`
   * Create file `/etc/systemd/system/docker.service.d/http-proxy.conf` and add:
   ```
   [Service]
    Environment="HTTP_PROXY=http://proxy.example.com:80/" "HTTPS_PROXY=https://proxy.example.com:443/"
   ```
   * Flush changes and restart docker
   `$ sudo systemctl daemon-reload `  
   `$ sudo systemctl restart docker `
   
### 5.2 Build django application to docker image
    `$ docker build -t mydjango:0.1 . `