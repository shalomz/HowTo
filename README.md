# HowTo
A Curated list on How-To's, ranging from any and all JS, Python, Django.
Feel free to use all the snippets here for whatever functionality/scenario. I hope this becomes helpful!
Contribute, contribute, contribute!
# 1. How to Render A Datepicker Widget on Multiple Input Fields
Say you want several input fields to have a datepicker widget, without explicitly defining the inputs for each one, or for the case where the input fields are generated dynamically, a possible fix (using jQuery) is written below:
::
    
    <script>
		$(document).ready(function () {
			$('body').on('focus',".dateinput", function(){
				$(this).datepicker();
			});
		})
	</script>
Whereby `dateinput` is the class of the input fields.
The complete markup is coming.
Where I found this useful: I had input fields that needed datepicker widgets within Django Formsets.

# 2. How to find if today is in a range of given dates
Say you hava an object with 2 Date attributes `start_date` and `end_date` and you would like to find the dates between the 2 dates:
:: 

	# Somewhere in your views.py
	from datetime import date, timedelta
	
	def daterange(start_date, end_date):
            """
            Function to find all dates within a particular range of 2 dates
   	    """
    	    for n in range(int ((end_date - start_date).days)+1):
	        yield start_date + timedelta(n)
			
			
	def your_view(request):
	    # Logic here, for capturing the 2 dates and assigning them names, i.e. start_date and end_date
	     now = date.today()  #Finding today's date
	     if now in daterange(start_date, end_date): 
                # Logic for if the date today is among the dates
             else:
                # Logic for if date today is not among the dates
		
# 3. Celery + Django
# How to run celery as a daemon?

  1. Create `/etc/init.d/celeryd` with the content from [celery repo](https://github.com/celery/celery/blob/master/extra/generic-init.d/celeryd)

  2. Make `celeryd` executable
    
    
    sudo nano /etc/init.d/celeryd
    

Copy-paste code from [celery
repo](https://github.com/celery/celery/blob/master/extra/generic-
init.d/celeryd) to the file

Save `celeryd` (`CTR+X, y, Enter` from nano)

Run following commands from the terminal:

    
    
    sudo chmod 755 /etc/init.d/celeryd
    sudo chown root:root /etc/init.d/celeryd
    

## 1.2. Configuration

### What to do?

  1. Create `/etc/default/celeryd`
  2. Configure it

### How?

    
    
    sudo nano /etc/default/celeryd
    

Configure it depending on what you need. Options and template can be found in
the [docs](http://docs.celeryproject.org/en/latest/userguide/daemonizing.html
#init-script-celeryd)

### My example:

    
    
    CELERY_BIN="project/venv/bin/celery" # App instance to use
    CELERY_APP="project_django_project" # Where to chdir at start.
    CELERYD_CHDIR="/home/username/project/" # Extra command-line arguments to the worker
    CELERYD_OPTS="--time-limit=300 --concurrency=8" # %n will be replaced with the first part of the nodename.
    CELERYD_LOG_FILE="/var/log/celery/%n%I.log"
    CELERYD_PID_FILE="/var/run/celery/%n.pid" # Workers should run as an unprivileged user.
    # You need to create this user manually (or you can choose
    # a user/group combination that already exists (e.g., nobody).
    CELERYD_USER="username"
    CELERYD_GROUP="username" # If enabled pid and log directories will be created if missing,
    # and owned by the userid/group configured.
    CELERY_CREATE_DIRS=1 export SECRET_KEY="foobar"

**Note**

You can set your environment variables in `/etc/default/celeryd`. For more
info about environment variable take a look at this [SO
answer](http://stackoverflow.com/questions/22759798/why-are-my-environment-
variables-not-detected-when-starting-up-celery)

### Test it

You can check if the worker is active by:

    
    
    sudo /etc/init.d/celeryd start
    sudo /etc/init.d/celeryd status
    

Don't forget to stop the worker:

    
    
    sudo /etc/init.d/celeryd stop 

## 2\. Init-script: celerybeat

  1. Create `/etc/init.d/celerybeat` with the content from [celery repo](https://github.com/celery/celery/blob/master/extra/generic-init.d/celerybeat)

  2. Make `celerybeat` executable
    
    
    sudo nano /etc/init.d/celerybeat
    

Copy-paste code from [celery
repo](https://github.com/celery/celery/blob/master/extra/generic-
init.d/celerybeat) to the file

Save `celerybeat` (`CTR+X, y, Enter` from nano)

Run following commands from the terminal:

    
    
    sudo chmod 755 /etc/init.d/celerybeat
    sudo chown root:root /etc/init.d/celerybeat
    

## 2.2. Configuration

### What to do?

Configurate either `/etc/default/celerybeat` or stick with
`/etc/default/celeryd`

### How?

    
    
    sudo nano /etc/default/celerybeat
    

Configure it depending on what you need. Options and template can be fount in
the [docs](http://docs.celeryproject.org/en/latest/userguide/daemonizing.html
#init-script-celerybeat)

**Note**

If you don't require any special configuration you can use
`/etc/default/celeryd` as you config file. Just don't create
`/etc/default/celerybeat`

### Test it

You can check if the beat is active by:

    
    
    sudo /etc/init.d/celerybeat start
    sudo /etc/init.d/celerybeat status
    

Don't forget to stop the worker:

    
    
    sudo /etc/init.d/celerybeat stop 

## 3\. Maintenance

As it was show you can control worker and beat with the following commands:

    
    
    /etc/init.d/celeryd {start|stop|restart}
    /etc/init.d/celerybeat {start|stop|restart}
    

If you want to see the logs you can open `CELERYD_LOG_FILE` provided in
`/etc/default/celeryd`. In our example is was `/var/log/celery/worker1.log`:

    
    
    sudo nano /var/log/celery/worker1.log 

## Resources

		
