# Serving multiple Django applications with Nginx and Gunicorn

Nginx makes a great server for your Gunicorn-powered Django applications. In
this article I will demonstrate how you can run multiple Django applications
on the same Nginx server, hosting sites on two different domains. Each
application will be set up in its own Virtualenv and each will be owned by and
run as a different user to limit consequences of a potential security breach.

### Prerequisites

This article is a continuation of [a previous article on setting up Django with
Nginx and Gunicorn](http://michal.karzynski.pl/blog/2013/06/09/django-nginx-
gunicorn-virtualenv-supervisor/). You should start by following instructions
in [that article](http://michal.karzynski.pl/blog/2013/06/09/django-nginx-
gunicorn-virtualenv-supervisor/) and prepare a server with the following
components:

  * Nginx
  * PostgreSQL
  * Virtualenv
  * Supervisor

Our goal in this article will be to create two applications, one called Hello
and one called Goodbye. The former will be served under the address
[http://hello.test](http://hello.test/) and the latter
[http://goodbye.test](http://goodbye.test/)

### Create a virtual environment for each app

In order to keep your apps independent, each will run in its own virtual
environment. Create an environment for each application using the `virtualenv`
command. In each environment install Django, Gunicorn, the application itself
and its other dependencies. Follow steps described in [my previous
article](http://michal.karzynski.pl/blog/2013/06/09/django-nginx-gunicorn-
virtualenv-supervisor/) for each app.

Let's say that for our `hello` and `goodbye` applications we would create
environments in `/webapps/hello_django` and `/webapps/goodbye_django`
respectively. We would get a directory structure containing the following
entries:

    
    
    /webapps/
    ├── hello_django <= virtualenv for the application Hello
    │   ├── bin
    │   │   ├── activate
    │   │   ├── gunicorn <= Hello app's gunicorn
    │   │   ├── gunicorn_start <= Hello app's gunicorn start script
    │   │   └── python
    │   ├── hello <= Hello app's Django project directory
    │   │   └── hello
    │   │   ├── settings.py <= hello.settings
    │   │   └── wsgi.py <= hello.wsgi
    │   ├── logs <= Hello app's logs will be saved here
    │   ├── media
    │   ├── run <= Gunicorn's socket file will be placed here
    │   └── static
    └── goodbye_django <= analogous virtualenv for the application Goodbye ├── bin │   ├── activate │   ├── gunicorn │   ├── gunicorn_start │   └── python ├── goodbye │   └── goodbye │   ├── settings.py │   └── wsgi.py ├── logs ├── media ├── run └── static
    

### Create system accounts for the webapps

Even though Django has a pretty good [security track
record](http://django.readthedocs.org/en/latest/releases/security.html), web
applications can become compromised. In order to make running multiple
applications safer we'll create a separate system user account for each
application. The apps will run on our system with the privileges of those
special users. Even if one application became compromised, an attacker would
only be able to take over the part of your system available to the hacked
application.

Create system users named `hello` and `goodbye` and assign them to a system
group called `webapps`.

    
    
    $ sudo groupadd --system webapps
    $ sudo useradd --system --gid webapps --home /webapps/hello_django hello $ sudo useradd --system --gid webapps --home /webapps/goodbye_django goodbye 

Now change the owner of files in each application's folder. I like to assign
the group `users` as the owner, because that allows regular users of the
server to access and modify parts of the application which are group-writable.
This is optional.

    
    
    $ sudo chown -R hello:users /webapps/hello_django
    $ sudo chown -R goodbye:users /webapps/goodbye_django
    

### Create Gunicorn start scripts

For each application create a simple shell script based on my
[gunicorn_start](https://gist.github.com/postrational/5747293#file-
gunicorn_start-bash) template. The scripts differ only in the values of
variables which they set.

For the Hello app, set the following values in
`/webapps/hello_django/bin/gunicorn_start`:

/webapps/hello_django/bin/gunicorn_start

    


|

    
    
    (...)
    NAME="hello_app" # Name of the application
    DJANGODIR=/webapps/hello_django/hello # Django project directory
    SOCKFILE=/webapps/hello_django/run/gunicorn.sock # we will communicte using this unix socket
    USER=hello # the user to run as
    GROUP=webapps # the group to run as
    NUM_WORKERS=3 # how many worker processes should Gunicorn spawn
    DJANGO_SETTINGS_MODULE=hello.settings # which settings file should Django use
    DJANGO_WSGI_MODULE=hello.wsgi # WSGI module name
    (...)
      
  
---|---  
  
And for the Goodbye app by analogy:

/webapps/goodbye_django/bin/gunicorn_start

    
    
 
    

|

    
    
    (...)
    NAME="goodbye_app" # Name of the application
    DJANGODIR=/webapps/goodbye_django/goodbye # Django project directory
    SOCKFILE=/webapps/goodbye_django/run/gunicorn.sock # we will communicte using this unix socket
    USER=goodbye # the user to run as
    GROUP=webapps # the group to run as
    NUM_WORKERS=3 # how many worker processes should Gunicorn spawn
    DJANGO_SETTINGS_MODULE=goodbye.settings # which settings file should Django use
    DJANGO_WSGI_MODULE=goodbye.wsgi # WSGI module name
    (...)
      
  
---|---  
  
### Create Supervisor configuration files and start the apps

Next, create a Supervisor configuration for each application. Add a file for
each app to the `/etc/supervisor/conf.d` directory.

One for Hello:

/etc/supervisor/conf.d/hello.conf

    
    
 

|

    
    
    [program:hello]
    command = /webapps/hello_django/bin/gunicorn_start ; Command to start app
    user = hello ; User to run as
    stdout_logfile = /webapps/hello_django/logs/gunicorn_supervisor.log ; Where to write log messages
    redirect_stderr = true ; Save stderr in the same log
      
  
---|---  
  
And one for Goodbye:

/etc/supervisor/conf.d/goodbye.conf

    
 

|

    
    
    [program:goodbye]
    command = /webapps/goodbye_django/bin/gunicorn_start ; Command to start app
    user = goodbye ; User to run as
    stdout_logfile = /webapps/goodbye_django/logs/gunicorn_supervisor.log ; Where to write log messages
    redirect_stderr = true ; Save stderr in the same log
      
  
---|---  
  
Reread the configuration files and update Supervisor to start the apps:

    
    
    $ sudo supervisorctl reread
    $ sudo supervisorctl update
    

You can also start them manually, if you prefer:

    
    
    $ sudo supervisorctl start hello
    $ sudo supervisorctl start goodbye
    

### Create Nginx virtual servers

Finally we can create virtual server configurations for each app based on
[this template](https://gist.github.com/postrational/5747293#file-hello-
nginxconf). These will be stored in `/etc/nginx/sites-available` and then
activated by links in `/etc/nginx/sites-enabled`.

/etc/nginx/sites-available/hello

    
    
  
    

|

    
    
    upstream hello_app_server {
     server unix:/webapps/hello_django/run/gunicorn.sock fail_timeout=0;
    }
    
    server {
     listen 80;
     server_name hello.test;
    
     client_max_body_size 4G;
    
     access_log /webapps/hello_django/logs/nginx-access.log;
     error_log /webapps/hello_django/logs/nginx-error.log;
    
     location /static/ {
     alias /webapps/hello_django/static/;
     }
    
     location /media/ {
     alias /webapps/hello_django/media/;
     }
    
     location / {
     proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
     proxy_set_header Host $http_host;
     proxy_redirect off;
     if (!-f $request_filename) {
     proxy_pass http://hello_app_server;
     break;
     }
     }
    }
      
  
---|---  
  
/etc/nginx/sites-available/goodbye

    
    
    

|

    
    
    upstream goodbye_app_server {
     server unix:/webapps/goodbye_django/run/gunicorn.sock fail_timeout=0;
    }
    
    server {
     listen 80;
     server_name goodbye.test;
    
     client_max_body_size 4G;
    
     access_log /webapps/goodbye_django/logs/nginx-access.log;
     error_log /webapps/goodbye_django/logs/nginx-error.log;
    
     location /static/ {
     alias /webapps/goodbye_django/static/;
     }
    
     location /media/ {
     alias /webapps/goodbye_django/media/;
     }
    
     location / {
     proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
     proxy_set_header Host $http_host;
     proxy_redirect off;
     if (!-f $request_filename) {
     proxy_pass http://goodbye_app_server;
     break;
     }
     }
    }
      
  
---|---  
  
Enable the virtual servers and restart Nginx:

    
    
    $ sudo ln -s /etc/nginx/sites-available/hello /etc/nginx/sites-enabled/hello
    $ sudo ln -s /etc/nginx/sites-available/goodbye /etc/nginx/sites-enabled/goodbye
    $ sudo service nginx restart
    

### Test the virtual servers

Now let's point each domain to our server for testing purposes. Making actual
changes to the Domain Name System is usually among the final steps when
working in production, performed after all tests are completed. For testing
you can simulate the DNS changes by adding an entry to the `/etc/hosts` file
of a computer from which you will be connecting to your server (your laptop
for example).

Say you want to serve Django applications under the domains `hello.test` and
`goodbye.test` from a server with the IP address of `10.10.10.200`. You can
simulate the appropriate DNS entries locally on your PC by putting the
following line into your `/etc/hosts` file. On Windows the file is
[conveniently hidden](http://en.wikipedia.org/wiki/Hosts_\(file\)#Location_in_
the_file_system) in `%SystemRoot%\system32\drivers\etc\hosts`.

/etc/hosts

    
    
    1
    2
    
    10.10.10.200 hello.test goodbye.test
      
  
---|---  
  
You can now navigate to each domain from your PC to test that each app on the
server is working correctly:

  * [http://hello.test](http://hello.test/)
  * [http://goodbye.test](http://goodbye.test/)

Good luck!
