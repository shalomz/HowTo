# Deploying web apps with Docker


![](https://static.amon.cx/images/blog/deploying-web-apps-docker/cover.jpg)

In 2014 Docker has grown immenesely and has a become a prefered tool for many
people in the devops community. I've been using it for over a year and the
moment Docker hit 1.0 a couple of months ago I decided to bite the bullet and
migrate my project Amon to Docker. These are some of the mistakes I made and
the lessons I learned during the last couple of months.

## Don't put everything in the container

Some articles compare Docker to Git and that is not entirely true. With Git
you are encouraged to commit often, with Docker however "commiting"
often(running `docker build`) will add history layers, eventually hitting the
limit of [127.](https://github.com/docker/docker/issues/1171)

Docker containers are designed to be portable and immutable - you can install
libraries, databases, servers. The dynamic parts like data directories and log
files however remain outside - on the host machine.

Putting your web application in a container should follow the exact same
principles - install the requirements beforehand with `docker build`, but
leave the source code in a directory on the host machine and mount it on boot.

    
    
     ADD myapp /home/myapp VOLUMES ['myapp']
    docker run -v /myapp:/home/myapp
    

## Use the Docker API

The recomended way to boot a container is:

    
    
    docker run --volume=/webapp/:/home/webapp:ro --publish=8080:8080 --name=myapp myapp:latest
    

Written like this it already looks error prone and in a dire need of
simplification. This is not all though - we have to add a couple more commands
for restarting, stopping the container. It doesn't get better when you put
these commands in a deployment capistrano/fabric script.

There are a lot of tools out there for managing docker containers -
[Panamax](http://panamax.io/), [Fig](https://github.com/docker/fig),
[Flynn](https://flynn.io/) and [Deis](http://deis.io/) to name a few, but at
least for me a lot can be acomplished with a simple wrapper around the Docker
API The example below shows my own Python script, but I think it is easy to
adapt and write something similar for Ruby, Nodejs, etc.

    
    
    from datetime import datetime
    from docker import Client
    from djangoapp.instance.models import Instance 
    class AmonInstance(object): 
        def __init__(self, account=None): 
            self.client = Client(base_url='unix://var/run/docker.sock', version='1.15') 
            self.account_id = 1 
            self.instance = Instance.objects.get(account_id=self.account_id) 
        def reload_container(self): 
            try: 
                self.stop_container() 
            except: 
                pass self.start_container() 
        def start_container(self): 
            container = self.client.create_container('martinrusev/djangoapp', name='djangoapp', ports=[8000], volumes=['/djangoapp/'] ) self.client.start(container, port_bindings={8000: 8000}, binds={ '/home/djangoapp': { 'bind': '/djangoapp', 'ro': True }, '/var/log/djangoapp': { 'bind': '/var/log', } }) container_id = container['Id'] Instance.objects.filter(account_id=self.account_id).update( container_id=container_id, modified_date=datetime.now() ) def stop_container(self): self.client.stop(self.instance.container_id) self.client.remove_container(self.instance.container_id) Instance.objects.filter(account=self.account).update(container_id=None)
    

## Debugging running containers

The container is running, but your web app is not working. There are two ways
to the debug a docker container:

  * You can run `docker inspect container_id` \- this will give you a nice JSON represenation of all the mounted directories, env variables, etc.
  * If you have configured some sort of logging in the container, `docker logs container_id` will print those logs.

With a more complicated web app with multiple services running on different
machines and ports, ENV variables, etc these methods are not very helpful. In
my case with a Django app I was given generic errors that the server failed to
start or I could not get cron to work.

Loggin in the container is the next logical step, but this is something you
can't easily do with the default docker commands. I found a tool called
**docker-bash** part of the popular [Phusion
Baseimage](https://github.com/phusion/baseimage-docker). It is not locked to
Baseimage though - you can log into any image/container. To install it follow
the instructions [here](https://github.com/phusion/baseimage-docker#docker_bash) and then:

    
    
    
    $ docker ps -a $ docker-bash container_id
    

## Running more than 1 process in a container

This is a highly debatable issue in the Docker community. Some people prefer
to stick to the Unix philosophy ("Make each program do one thing well."),
others like me prefer to group containers into logical units. I think there is
no right or wrong way to do this and it really depends on the problem you are
trying to solve.

In my specific case I am grouping cron and django app in the same container:

## Config files vs ENV

This is not a common issue, but I will put it here anyway. You will encounter
this problem if your app has some soer of background processing with cron
jobs.

The most important thing you should remember when running cron in a container
is the fact that cron does not have access to the `ENV` variables set on boot.
Things like `DB_URL=postgres://myhost`

One way to solve the problem (and simplify that `docker run` command) is
config files instead of ENV variables.

    
    
    [webapp]
    DB_URL=postgres://myhost
    REDIS_URL=redis:// CONFIG_FILE = '/home/webapp/config.cfg'
    config = ConfigParser.ConfigParser()
    config.read(CONFIG_FILE)
    

## Conclusion

If you are a web developer and still on the fences about giving Docker a try,
I can asure you that it has become a very stable and reliable tool.

Even if you don't go all the way and deploy it in production - you can use it
for building local enviroments, sharing those enviroments and keeping your dev
machine clean.

I definitely prefer `docker pull redis` vs [all these steps required to
install just one libary](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-redis) and then replicating those steps on the production
server(I use Ansible, Docker is still easier)
