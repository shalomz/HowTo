# How I Replaced SSH with ZeroMQ and Salt


SSH has been a constant presence in my daily life for a very long time. Just
like everyone who has to deal with Linux servers on a daily basis, SSH is the
"status quo", like it or not - this is how things are and there is nothing you
can do about that. Before I get started I want to cover what I think about SSH
in general.

SSH is functional and it gets the job done. That is if you are single guy with
a handful of servers. When you have to connect to multiple servers at the same
time, establish inter-server communication, give access to other members of
your team SSH becomes a chore.

Establishing a SSH connection is really slow, there is no such thing as - "I
just want to quickly check these 3 servers". For me personally this slowness
really became an issue in the last year or so with the rise of Docker and
containers in general. With Docker we not only have multiple servers, there
are also multiple containers running on each one. In the real world that
translates to more frequent interactions with our servers - for deploying,
debugging and provisioning.

With all that said - My goal is not, to completely replace SSH with some new
hyped technology. It is a reliable default and it gets the job done. I just
want to reduce the daily interactions I have with it to the absolute minimum.
Zero if possible.

## ZeroMQ: A viable solution?

I first discovered ZeroMQ in 2011, when I was looking for a non-blocking
alternative to http. ZeroMQ is a really good networking library - you can have
a fully functional fast server to server communication in 40-50 lines of code.

Back then I created a small library for streaming log data to a centralized
server. It was really fast and it worked very well, with one exception - it
was not secure. This is the main issue I had with ZeroMQ back then and this
issue is still present today.

The basic ZeroMQ patterns are easy to grasp, you can skim through the docs and
build something in an afternoon. Building a secure ZeroMQ communication is
extremely difficult. There are a several third party libraries, but the
documentation and the examples for all these projects is sparse.

To use ZeroMQ you can be a newbie developer, to secure a ZeroMQ connection -
you have to be proficient in Cryptography. ZeroMQ could be way more popular if
they decide to include the security layer in the core.

## ZeroMQ and SaltStack

I discovered Ansible and SaltStack almost at the same time. I was overwhelmed
by the complexity in Salt and eventually settled for Ansible. I have been
using Ansible ever since. Coming back to the beginning of the post - one of
the major issues I have with Ansible is that you have to deal with SSH and and
it is slow. Everything takes forever and there is no such thing as real-time
debugging in Ansible. Behind the scenes Ansible tries to speed things up bu
keeping those connections persistent and forking itself for each server.
Memory usage in gigabytes is common, if you use Ansible for a lot of hosts.

For all these reasons I've decided to try Salt once again. My idea right from
the start was to try and strip all the complexity and just use the secure
ZeroMQ connection between master(s) and minions for real-time debugging.

You can check my [full review / experience](https://www.amon.cx/blog
/saltstack-review/) with SaltStack here. In this section I want to cover what
happened after that.

## Experiments with SaltStack

The first thing you do in Salt is:

    
    
    salt "*" cmd.run shell command
    

The line above will execute the **shell command** on all your minions. You can
check log files, install packages, restart services, etc.

Thanks to ZeroMQ it is almost instant and it doesn't matter if you want to do
this on 1 or 5 servers. Even if this is the only thing you use in Salt, it
will be enough to cut the number of the required debugging SSH sessions in
half.

    
    
    salt "webserver" cmd.run cat /var/log/nginx.error.log
    salt "webserver" cmd.run systemctl nginx reload
    # Or you can use the nginx module in Salt
    salt 'webserver' nginx.signal reload
    

### Using the Salt modules

Salt comes with over [100 modules](http://docs.saltstack.com/en/latest/salt-
modindex.html) for almost every app imaginable. The modules expose system
functionality like creating users or modifying IP tables. You can use all
these modules directly from the terminal(Everything in the salts.modules
section).

    
    
    salt '*' git.clone /path/to/repo git://github.com/saltstack/salt.git
    salt 'webserver' nginx.configtest
    salt 'dbserver' mysql.db_create 'myproject'
    salt '*' docker.get_containers
    

### Editing files

One of the most annoying things you could do on a fresh new server is edit a
config file or fix a typo. Usually you can choose between a bare-bone version
of vim or nano. I used Vim as a main editor for almost 2 years and even I am
not comfortable working in a minimal Vim - you still need to apply a theme,
because the default is dark blue on black, to install plugins for highlighting
and bracket matching, so you can spot typos easier. Configuring Vim could be
added to the provisioning playbooks/states/manifests, but it is an overkill to
do so on every server.

With Salt you can get a config file from the server, edit on your own dev
machine, add it to your provisioning scripts and then send it back to the
server:

    
    
    salt 'webserver' cp.get_file_str salt://nginx.conf
    subl nginx.conf
    salt 'webserver' cp.get_file salt://nginx.conf /etc/nginx/nginx.conf
    salt "nginx" cmd.run systemctl restart nginx
    

I hope that you could see the usefulness in the examples above. They are all
nice time savers, but some issues remain. Connecting to your servers from a
different machine and giving access to your team. With SSH you can at least in
theory copy the private keys or generate new ones (that is not recommended at
all, but it is actually a practice among developers). With Salt you cant even
do that, your minions connect to a salt-master which is and IP and that
doesn't work well if you are on the move or you have a dynamic IP.

To solve this problem I started experimenting in the browser. If you have used
any cloud server in the past few years you probably know the "terminal"
emulators they all have. These emulators are some kind of a Java or nodejs
whatever app and thanks to SSH, they are really slow and you can't connect to
more than one server at the same time.

But what if that terminal was based on Salt/ZeroMQ, not SSH. You can see the
results below:

![](https://d3qko0oh50jdzc.cloudfront.net/images/blog/how-i-replaced-ssh/remote_execution.gif)

It is really awesome for debugging - instant, fluid and best of all,
accessible from anywhere. This was all build on the [Salt
API](http://docs.saltstack.com/en/latest/ref/clients/).The only limitation at
the moment is that Salt provides only a client in Python.

But if you can get around that, you have all the power available in a typical
web application - precise permissions for your team members, two factor
authentication is not that hard to implement, accessible from anywhere - on
the beach, on your laptop or phone without creepy mobile SSH terminals and you
can disable certain dangerous commands quite easily.

One last thing before I finish this post. Salt could be really intimidating
when you are getting started. I wrote a blog post about Salt awhile ago that
you can [check it out](https://www.amon.cx/blog/saltstack-review/) \- there is
a nice info-graphic explaining how Salt works and there is a Up and running in
5 minutes section.

Thanks for reading and I really hope you enjoyed this post, I am going back to
my terminal browser app to add some sweat oh-my-zsh auto-complete goodies. At
least for me - at this point, there is no going back.
