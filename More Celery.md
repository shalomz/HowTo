# Orchestrating a Background Job Workflow in Celery for Python

Modern web applications and their underlying systems are faster and more
responsive than ever before. However, there are still many cases where you
want to offload execution of a heavy task to other parts of your entire system
architecture instead of tackling them on your main thread. Identifying such
tasks is as simple as checking to see if they belong to one of the following
categories:

  * Periodic tasks -- Jobs that you will schedule to run at a specific time or after an interval, e.g., monthly report generation or a web scraper that runs twice a day.
  * Third-party tasks -- The web app must serve users quickly without waiting for other actions to complete while the page loads, e.g., sending an email or notification or propagating updates to internal tools (such as gathering data for A/B testing or system logging).
  * Long-running jobs -- Jobs that are expensive in resources, where users need to wait while they compute their results, e.g., complex workflow execution (DAG workflows), graph generation, Map-Reduce like tasks, and serving of media content (video, audio).

A straightforward solution to execute a background task would be running it
within a separate thread or process. Python is a high-level Turing complete
programming language, which unfortunately does not provide built-in
concurrency on a scale matching that of Erlang, Go, Java, Scala, or Akka.
Those are based on Tony Hoare's Communicating Sequential Processes
([CSP](https://en.wikipedia.org/wiki/Communicating_sequential_processes)).
Python threads, on the other hand, are coordinated and scheduled by the global
interpreter lock ([GIL](https://wiki.python.org/moin/GlobalInterpreterLock)),
which prevents multiple native threads from executing Python bytecodes at
once. Getting rid of the GIL is a topic of much discussion among [Python
developers](https://www.toptal.com/python), but it is not the focus of this
article. Concurrent programming in Python is old-fashioned, although you're
welcome to read about it in _[Python Multithreading
Tutorial](https://www.toptal.com/python/beginners-guide-to-concurrency-and-
parallelism-in-python)_ by fellow Toptaler Marcus McCurdy. So, designing
communication between processes consistently is an error-prone process and
leads to code coupling and bad system maintainability, not to mention that it
negatively affects scalability. Additionally, the Python process is a normal
process under an Operating System (OS) and, with the entire Python standard
library, it becomes a heavyweight. As the number of processes in the app
increases, switching from one such process to another becomes a time-consuming
operation.

To understand concurrency with Python better, watch this incredible speech by
David Beazley at [PyCon'15](https://www.youtube.com/watch?v=MCs5OvhV9S4).

A much better solution is to serve a _distributed queue_ or its well-known
sibling paradigm called _publish-subscribe_. As depicted in Figure 1, there
are two types of applications in which one, called the _publisher_, sends
messages and the other, called the _subscriber_, receives messages. Those two
agents do not interact with each other directly and are not even aware of each
other. Publishers send messages to a central queue, or _broker_, and
subscribers receive messages of interest from this broker. There are two main
advantages in this method:

  * Scalability -- agents don't need to know about each other in the network. They are focused by topic. So it means that each can continue to operate normally regardless of the other in asynchronous fashion.
  * Loose coupling -- each agent represents its part of the system (service, module). Since they are loosely coupled, each can scale individually beyond the datacenter.

There are lots of messaging systems that support such paradigms and provide a
neat API, driven either by TCP or HTTP protocols, e.g., JMS, RabbitMQ, Redis
Pub/Sub, Apache ActiveMQ, etc.

![Publish-Subscribe paradigm with Celery
Python](https://uploads.toptal.io/blog/image/123889/toptal-blog-
image-1503063593706-b7c7423cd2d8cd40ab2e765ffeb25c6c.png) Figure 1: Publish-
Subscribe paradigm

## What Is Celery?

[Celery](http://www.celeryproject.org/) is one of the most popular background
job managers in the Python world. Celery is compatible with several message
brokers like RabbitMQ or Redis and can act as both producer and consumer.

> Celery is an asynchronous task queue/job queue based on distributed message
passing. It is focused on real-time operations but supports scheduling as
well. The execution units, called tasks, are executed concurrently on one or
more worker servers using multiprocessing, [Eventlet](http://eventlet.net/),
or [gevent](http://gevent.org/). Tasks can execute asynchronously (in the
background) or synchronously (wait until ready).  
- [Celery Project](http://www.celeryproject.org/)

To getting started with Celery, just follow a step-by-step guide at the
[official docs](http://docs.celeryproject.org/en/latest/getting-started/first-
steps-with-celery.html).

The focus of this article is to give you a good understanding of which use
cases could be covered by Celery. In this article, we will not only show
interesting examples but also try to learn how to apply Celery to real-world
tasks such as background mailing, report generation, logging, and error
reporting. I will share my way of testing the tasks beyond emulation and,
finally, I will provide a few tricks that are not (well) documented in the
official documentation which took me hours of research to discover for myself.

If you have no previous experience with Celery, I encourage you first to try
it out following the official tutorial.

## Whetting Your Appetite

If this article intrigues you and makes you want to dive into code
immediately, then follow to this [GitHub repository](https://github.com/Rustem
/toptal-blog-celery-toy-ex) for the code used in this article. The `README`
file there will give you the quick and dirty approach for running and playing
with the example applications.

## First Steps with Celery

For starters, we will walk through a series of practical examples that will
show a reader how simply and elegantly Celery solves seemingly non-trivial
tasks. All examples will be presented within the Django framework; however,
most of them could easily be ported to other Python frameworks (Flask,
Pyramid).

The project layout was generated by [Cookiecutter
Django](https://github.com/pydanny/cookiecutter-django); however, I only kept
a few dependencies that, in my opinion, facilitate the development and
preparation of these use cases. I also removed unnecessary modules for this
post and applications to reduce noise and make the code easier to understand.

    
    
     - celery_uncovered/ - celery_uncovered/__init__.py - celery_uncovered/{toyex,tricks,advex} - celery_uncovered/celery.py - config/settings/{base,local,test}.py - config/urls.py - manage.py
    

  * `celery_uncovered/{toyex,tricks,advex}` contains different applications that we will cover in this post. Each application contains set of examples organized by the level of Celery understanding required.
  * `celery_uncovered/celery.py` defines a Celery instance.

_**File:** `celery_uncovered/celery.py`:_

    
    
    from __future__ import absolute_import import os
    from celery import Celery, signals # set the default Django settings module for the 'celery' program.
    os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'config.settings.local') app = Celery('celery_uncovered') # Using a string here means the worker will not have to
    # pickle the object when using Windows.
    app.config_from_object('django.conf:settings', namespace='CELERY')
    app.autodiscover_tasks()
    

Then we need to be ensured that Celery will start together with Django. For
that reason, we import the app in `celery_uncovered/__init__.py`.

_**File:** `celery_uncovered/__init__.py`:_

    
    
    from __future__ import absolute_import # This will make sure the app is always imported when
    # Django starts so that shared_task will use this app.
    from .celery import app as celery_app # noqa __all__ = ['celery_app'] __version__ = '0.0.1'
    __version_info__ = tuple([int(num) if num.isdigit() else num for num in __version__.replace('-', '.', 1).split('.')])
    

`config/settings` is the source of configuration for our app and Celery.
Depending on the execution environment, Django will launch corresponding
settings: `local.py` for development or `test.py` for testing. You may also
define your own environment if you want by creating a new python module (e.g.,
`prod.py`). Celery configurations are prefixed with `CELERY_`. For this post,
I configured RabbitMQ as the broker and SQLite as the result bac-end.

_**File:** `config/local.py`:_

    
    
    CELERY_BROKER_URL = env('CELERY_BROKER_URL', default='amqp://guest:guest@localhost:5672//')
    CELERY_RESULT_BACKEND = 'django-db+sqlite:///results.sqlite'
    

## Scenario 1 - Report generation and export

The first case we will cover is report generation and export. In this example,
you will learn how to define a task that produces a CSV report and schedule it
at regular intervals with
[celerybeat](http://docs.celeryproject.org/en/latest/userguide/periodic-
tasks.html).

_**Use case description:** fetch the five hundred hottest repositories from
GitHub per chosen period (day, week, month), group them by topics, and export
the result to the CSV file._

If we provide an HTTP service that will execute this feature triggered by
clicking a button labeled "Generate Report," the application would stop and
wait for the task to complete before sending an HTTP response back. This is
bad. We want our web application to be fast and we don't want our users to
wait while our back-end computes the results. Instead of waiting for the
results to be produced, we would rather queue the task to worker processes via
a registered queue in Celery and respond with a `task_id` to the front-end.
Then the front-end would use the `task_id` to query the task result in an
asynchronous fashion (e.g., AJAX) and will keep the user updated with the task
progress. Finally, when the process finishes, the results can be served as a
file to download via HTTP.

### Implementation Details

First of all, let us decompose the process into its smallest possible units
and create a pipeline:

  1. **Fetchers** are the workers that are responsible for getting repositories from the GitHub service.
  2. The **Aggregator** is the worker that is responsible for consolidating results into one list.
  3. The **Importer** is the worker that is producing CSV reports of the hottest repositories in GitHub.
![A pipeline of Celery Python
workers](https://uploads.toptal.io/blog/image/123890/toptal-blog-
image-1503064713226-92ba78eff2a0df5ff20e3ef6c84d2640.png) Figure 2: A pipeline
of workers with Celery and Python

Fetching repositories is an HTTP request using the [GitHub Search
API](https://developer.github.com/v3/search/) `GET /search/repositories`.
However, there is a limitation of the GitHub API service that should be
handled: The API returns up to 100 repositories per request instead of 500. We
could send five requests one at a time, but we don't want to keep our user
waiting for five individual requests since HTTP requests are an I/O bound
operation. Instead, we can execute five concurrent HTTP requests with an
appropriate page parameter. So the page will be in the range [1..5]. Let's
define a task called `fetch_hot_repos/3 -> list` in the `toyex/tasks.py`
module:

_**File:** `celery_uncovered/toyex/local.py`_

    
    
    @shared_task
    def fetch_hot_repos(since, per_page, page): payload = { 'sort': 'stars', 'order': 'desc', 'q': 'created:>={date}'.format(date=since), 'per_page': per_page, 'page': page, 'access_token': settings.GITHUB_OAUTH} headers = {'Accept': 'application/vnd.github.v3+json'} connect_timeout, read_timeout = 5.0, 30.0 r = requests.get( 'https://api.github.com/search/repositories', params=payload, headers=headers, timeout=(connect_timeout, read_timeout)) items = r.json()[u'items'] return items
    

So `fetch_hot_repos` creates a request to the GitHub API and responds to the
user with a list of repositories. It receives three parameters that will
define our request payload:

  * `since` -- Filters repositories on the date of creation.
  * `per_page` -- Number of results to return per request (limited by 100).
  * `page` --\- Requested page number (in the range [1..5]).

_**Note:** In order to use GitHub Search API, you will need an OAuth Token to
pass authentication checks. In our case, it's saved in the settings under
`GITHUB_OAUTH`._

Next, we need to define a master task that will be responsible for aggregating
results and exporting them into a CSV file:
`produce_hot_repo_report_task/2->filepath:`

_**File:** `celery_uncovered/toyex/local.py`_

    
    
    @shared_task
    def produce_hot_repo_report(period, ref_date=None): # 1. parse date ref_date_str = strf_date(period, ref_date=ref_date) # 2. fetch and join fetch_jobs = group([ fetch_hot_repos.s(ref_date_str, 100, 1), fetch_hot_repos.s(ref_date_str, 100, 2), fetch_hot_repos.s(ref_date_str, 100, 3), fetch_hot_repos.s(ref_date_str, 100, 4), fetch_hot_repos.s(ref_date_str, 100, 5) ]) # 3. group by language and # 4. create csv return chord(fetch_jobs)(build_report_task.s(ref_date_str)).get() @shared_task
    def build_report_task(results, ref_date): all_repos = [] for repos in results: all_repos += [Repository(repo) for repo in repos] # 3. group by language grouped_repos = {} for repo in all_repos: if repo.language in grouped_repos: grouped_repos[repo.language].append(repo.name) else: grouped_repos[repo.language] = [repo.name] # 4. create csv lines = [] for lang in sorted(grouped_repos.keys()): lines.append([lang] + grouped_repos[lang]) filename = '{media}/github-hot-repos-{date}.csv'.format(media=settings.MEDIA_ROOT, date=ref_date) return make_csv(filename, lines)
    

This task uses `celery.canvas.group` to execute five concurrent calls of
`fetch_hot_repos/3`. Those results are waited for and then reduced to a list
of repository objects. Then our result set is grouped by topic and finally
exported into a generated CSV file under the `MEDIA_ROOT/` directory.

In order to schedule the task periodically, you may want to add an entry to
the schedule list in the configuration file:

_**File:** `config/local.py`_

    
    
    from celery.schedules import crontab CELERY_BEAT_SCHEDULE = { 'produce-csv-reports': { 'task': 'celery_uncovered.toyex.tasks.produce_hot_repo_report_task', 'schedule': crontab(minute=0, hour=0) # midnight, 'args': ('today',) },
    } 

### Trying It Out

In order to launch and test how the task is working, first we need to start
the Celery process:

    
    
     $ celery -A celery_uncovered worker -l info
    

Next, we need to create the `celery_uncovered/media/` directory. Then, you
will be able to test its functionality either via Shell or Celerybeat:

**Shell**:
    
    
    from datetime import date
    from celery_uncovered.toyex.tasks import produce_hot_repo_report_task
    produce_hot_repo_report_task.delay('today').get(timeout=5)
    

**Celerybeat**:
    
    
     # Start celerybeat with the following command $ celery -A celery_uncovered beat -l info
    

You can watch the results under the `MEDIA_ROOT/` directory.

## Scenario 2 - Report on _Server 500_ Errors via Email

One of the most common use cases for Celery is sending email notifications.
Email notification is an offline I/O bound operation that leverages either a
local SMTP server or a third-party SES. There are many use cases that involve
sending an email and, for most of them, the user doesn't need to wait until
this process is finished before receiving an HTTP response. That's why it is
preferred to execute such tasks in the background and respond to the user
immediately.

_**Use case description:** Report 50X errors to administrator email via
Celery._

Python and Django have the necessary background to perform [system
logging](https://docs.djangoproject.com/en/1.11/topics/logging/). I won't get
into details of how Python's logging actually works. However, if you have
never tried it before or need a refresher, read the documentation of the
built-in [logging](https://docs.python.org/3/library/logging.html#module-
logging) module. You definitely want this in your production environment.
Django has special logger handler called [AdminEmailHandler](https://docs.djan
goproject.com/en/1.11/topics/logging/#django.utils.log.AdminEmailHandler) that
emails administrators for each log message it receives.

### Implementation Details

The main idea is to extend the `send_mail` method of the `AdminEmailHandler`
class in such a way that it could send mail via Celery. This could be done as
illustrated in the figure below:

![Handling admin emails with Celery &
Python](https://uploads.toptal.io/blog/image/123891/toptal-blog-
image-1503064907768-03491c5268ba537c6bc6eac67e684999.png) Figure 3: Handling
admin emails with Celery and Python

First, we need to set up a task called `report_error_task` that calls
`mail_admins` with the provided subject and message:

_**File:** `celery_uncovered/toyex/tasks.py`_

    
    
    @shared_task
    def report_error_task(subject, message, *args, **kwargs): mail_admins(subject, message, *args, **kwargs)
    

Next, we actually extend AdminEmailHandler so that it will internally call
just the defined Celery task:

_**File:** `celery_uncovered/toyex/admin_email.py`_

    
    
    from django.utils.log import AdminEmailHandler
    from celery_uncovered.handlers.tasks import report_error_task class CeleryHandler(AdminEmailHandler): def send_mail(self, subject, message, *args, **kwargs): report_error_task.delay(subject, message, *args, **kwargs)
    

Finally, we need to set up logging. The configuration of logging in Django is
fairly straightforward. What you need is to override `LOGGING` so that the
logging engine starts using a newly defined handler:

_**File** `config/settings/local.py`_

    
    
    LOGGING = { 'version': 1, 'disable_existing_loggers': False, ..., 'handlers': { ... 'mail_admins': { 'level': 'ERROR', 'filters': ['require_debug_true'], 'class': 'celery_uncovered.toyex.log_handlers.admin_email.CeleryHandler' } }, 'loggers': { 'django': { 'handlers': ['console', 'mail_admins'], 'level': 'INFO', }, ... }
    }
    

Notice that I intentionally set up handler filters `require_debug_true` in
order to test this functionality while the application is running in debug
mode.

### Trying It Out

To test it, I prepared a Django view that serves a "division-by-zero"
operation at `localhost:8000/report-error`. You also need to start a MailHog
Docker container to test that the email is actually sent.

    
    
     $ docker run -d -p 1025:1025 -p 8025:8025 mailhog/mailhog $ CELERY_TASKSK_ALWAYS_EAGER=False python manage.py runserver $ # with your browser navigate to [http://localhost:8000](http://localhost:8000) $ # now check your outgoing emails by vising web UI [http://localhost:8025](http://localhost:8025)
    

As a mail testing tool, I set up MailHog and configured Django mailing to use
it for SMTP delivery. There are many ways to [deploy and
run](https://github.com/mailhog/MailHog#getting-started) MailHog. I decided to
go with a Docker container. You can find the details in the corresponding
README file:

_**File:** `docker/mailhog/README.md`_

    
    
     $ docker build . -f docker/mailhog/Dockerfile -t mailhog/mailhog:latest $ docker run -d -p 1025:1025 -p 8025:8025 mailhog/mailhog $ # navigate with your browser to localhost:8025
    

To configure your application to use MailHog, you need to add the following
lines in your config:

_**File:** `config/settings/local.py`_

    
    
    EMAIL_BACKEND = env('DJANGO_EMAIL_BACKEND', default='django.core.mail.backends.smtp.EmailBackend')
    EMAIL_PORT = 1025
    EMAIL_HOST = env('EMAIL_HOST', default='mailhog')
    

## Beyond Default Celery Tasks

Celery tasks could be created out of any callable function. By default, any
user-defined task is injected with `celery.app.task.Task` as a parent
(abstract) class. This class contains the functionality of running tasks
asynchronously (passing it via the network to a Celery worker) or
synchronously (for testing purposes), creating signatures and many other
utilities. In the next examples, we will try to extend `Celery.app.task.Task`
and then use it as a base class in order to add a few useful behaviors to our
tasks.

## Scenario 3 - File Logging per Task

In one of my projects, I was developing an app that provides the end user with
an Extract, Transform, Load (ETL)-like tool that was able to ingest and then
filter a huge amount of hierarchical data. The back-end was split into two
modules:

  * Orchestration of a data processing pipeline with Celery
  * Data processing with Go

Celery was deployed with one Celerybeat instance and more than 40 workers.
There were more than twenty different tasks that composed the pipeline and
orchestration activities. Each such task may fail at some point. All these
failures were dumped into the system log of each worker. At some point, it
started becoming inconvenient to debug and maintain the Celery layer.
Eventually, we decided to isolate the task log to a task specific file.

_**Use case description:** Extend Celery so that each task logs its standard
output and errors to files_

Celery provides Python applications with great control over what it does
internally. It ships with a familiar signals framework. Applications that are
using Celery can subscribe to a few of those in order to augment the behavior
of certain actions. We are going to leverage task-level signals to provide
verbose tracking of individual task lifecycles. Celery always comes with a
logging back-end, and we are going to take benefit from it while only slightly
overriding in a few places to achieve our goals.

### Implementation Details

Celery already supports logging per task. To save to a file, it is necessary
to dispatch the log output to the proper location. In our case, proper
location of the task is a file matching the name of the task. On the Celery
instance, we will override the built-in logging configuration with dynamically
inferred logging handlers. It is possible to subscribe to the
`celeryd_after_setup` signal and then configure system logging there:

_**File:** `celery_uncovered/toyex/celery_conf.py`_

    
    
    @signals.celeryd_after_setup.connect
    def configure_task_logging(instance=None, **kwargs): tasks = instance.app.tasks.keys() LOGS_DIR = settings.ROOT_DIR.path('logs') if not os.path.exists(str(LOGS_DIR)): os.makedirs(str(LOGS_DIR)) print 'dir created' default_handler = { 'level': 'DEBUG', 'filters': None, 'class': 'logging.FileHandler', 'filename': '' } default_logger = { 'handlers': [], 'level': 'DEBUG', 'propogate': True } LOG_CONFIG = { 'version': 1, # 'incremental': True, 'disable_existing_loggers': False, 'handlers': {}, 'loggers': {} } for task in tasks: task = str(task) if not task.startswith('celery_uncovered.'): continue task_handler = copy_dict(default_handler) task_handler['filename'] = str(LOGS_DIR.path(task + ".log")) task_logger = copy_dict(default_logger) task_logger['handlers'] = [task] LOG_CONFIG['handlers'][task] = task_handler LOG_CONFIG['loggers'][task] = task_logger logging.config.dictConfig(LOG_CONFIG)
    

Notice that for each task registered in the Celery app, we are building a
corresponding logger with its handler. Each handler is of the type
`logging.FileHandler`, and therefore each such instance receives a filename as
an input. All you need to get this running is to import this module to
`celery_uncovered/celery.py` in the end of the file:

    
    
    import celery_uncovered.tricks.celery_conf
    

A particular task logger could be received by calling
`get_task_logger(task_name)`. In order to generalize such behavior for each
task, it is necessary to slightly extend `celery.current_app.Task` with a few
utility methods:

_**File:** `celery_uncovered/tricks/celery_ext.py`_

    
    
    class LoggingTask(current_app.Task): abstract = True ignore_result = False @property def logger(self): logger = get_task_logger(self.name) return logger def log_msg(self, msg, *msg_args): self.logger.debug(msg, *msg_args)
    

Now, in the case of a call to `task.log_msg("Hello, my name is: %s",
task.request.id)`, the log output will be routed to the corresponding file
under the task name.

### Trying It Out

In order to launch and test how this task is working, first start the Celery
process:

    
    
     $ celery -A celery_uncovered worker -l info
    

Then you will be able to test functionality via Shell:

    
    
    from datetime import date
    from celery_uncovered.tricks.tasks import add
    add.delay(1, 3)
    

Finally, to see the result, navigate to the `celery_uncovered/logs` directory
and open the corresponding log file called
`celery_uncovered.tricks.tasks.add.log`. You might see something similar as
below after running this task multiple times:

    
    
     Result of 1 + 2 = 3 Result of 1 + 2 = 3 ...
    

## Scenario 4 - Scope-Aware Tasks

Let us imagine a Python application for international users that is built on
Celery and Django. The users can set which language (locale) they use your
application in.

You have to design a multilingual, locale-aware email notification system. To
send email notifications, you've registered a special Celery task that is
handled by a specific queue. This task receives some key arguments as input
and a current user locale so that email will be sent in the user's chosen
language.

Now imagine that we have many such tasks, but each of those tasks accepts a
locale argument. In this case, wouldn't it be better to solve it on a higher
level of abstraction? Here, we see just how to do that.

_**Use case description:** Automatically inherit scope from one execution
context and inject it into the current execution context as a parameter._

### Implementation Details

Again, as we did with the task logging, we want to extend a base task class
`celery.current_app.Task` and override a few methods responsible for calling
tasks. For the purpose of this demonstration, I'm overriding the
`celery.current_app.Task::apply_async` method. There are extra tasks for this
module that will help you to produce a fully functioning replacement.

_**File:** `celery_uncovered/tricks/celery_ext.py`_

    
    
    class ScopeBasedTask(current_app.Task): abstract = True ignore_result = False default_locale_id = DEFAULT_LOCALE_ID scope_args = ('locale_id',) def __init__(self, *args, **kwargs): super(ScopeBasedTask, self).__init__(*args, **kwargs) self.set_locale(locale=kwargs.get('locale_id', None)) def set_locale(self, scenario_id=None): self.locale_id = self.default_locale_id if locale_id: self.locale_id = locale_id else: self.locale_id = get_current_locale().id def apply_async(self, args=None, kwargs=None, **other_kwargs): self.inject_scope_args(kwargs) return super(ScopeBasedTask, self).apply_async(args=args, kwargs=kwargs, **other_kwargs) def __call__(self, *args, **kwargs): task_rv = super(ScopeBasedTask, self).__call__(*args, **kwargs) return task_rv def inject_scope_args(self, kwargs): for arg in self.scope_args: if arg not in kwargs: kwargs[arg] = getattr(self, arg) 

The key clue is to pass the current locale as a key-value argument into a
calling task by default. If a task was called with a certain locale as an
argument, then it is unchanged.

### Trying It Out

To test this functionality, let us define a dummy task of type
`ScopeBasedTask`. It locates a file by locale ID and reads its content as
JSON:

_**File:** `celery_uncovered/tricks/tasks.py`_

    
    
    @shared_task(bind=True, base=ScopeBasedTask)
    def read_scenario_file_task(self, **kwargs): fixture_parts = ["locales", "sc_%i.json" % kwargs['scenario_id']] return read_fixture(*fixture_parts)
    

Now what you need to do is to repeat the steps of launching Celery, starting
up the shell, and testing execution of this task on different scenarios.
Fixtures are located under the `celery_uncovered/tricks/fixtures/locales/`
directory.

## Conclusion

This post aimed to explore Celery from different perspectives. I demonstrated
Celery in conventional examples such as mailing and report generation as well
as shared tricks to some interesting niche business use-cases. Celery is built
upon a data-driven philosophy and your team may make their lives much simpler
by introducing it as part of their system stack. Developing Celery-based
services isn't very complicated if you have basic Python experience, and you
should be able to pick it up fairly quickly. The default configuration is good
enough for most uses, but if required, they can be very flexible.

Our team made the choice to use Celery as an orchestration back-end for
background jobs and long-running tasks. We use it extensively for a variety of
use-cases, of which only a few were mentioned in this post. We ingest and
analyze gigabytes of data every day, but this is only the beginning of
horizontal scaling techniques.

The publish-subscribe (or producer-consumer) pattern is a distributed
messaging pattern in computer systems where publishers broadcast messages over
a message broker, and subscribers listen for the messages. Both can be
isolated components of the system, neither aware nor in direct communication
with the other.

Celery is one of the most popular background job managers in the Python world.
Celery is compatible with several message brokers like RabbitMQ or Redis and
can act as both producer and consumer.
