# Using django-auth-ldap with Active Directory

There are a couple of online tutorials showing how to get up and running with
django-auth-ldap, but I couldn't find anything that discussed start to finish
how to get it working against Active Directory and the Django admin site.

Hopefully this will help anyone trying to achieve a similar goal. The
following was carried out on Django 1.4.5 with django-auth-ldap 1.2.1 on
Debian 7.

The idea is to use Active Directory to authenticate users to a Django admin
based application, and to map users to Django defined groups and permissions
based on Active Directory group membership.

First you will need to install python-ldap and django-auth-ldap The process to
do this isn't obvious, you will need to install a number of packages before it
will install properly with pip.

> apt-get install libldap2-dev apt-get install python-dev apt-get install
libsasl2-dev pip-install python-ldap

>

> pip-install django-auth-ldap

Next you are going to need the following AD related things:

  * User in order to bind to AD. You will need the full distinguished name.
  * The object class that you use for username within AD. This is normally "sAMAccountName".
  * The object class name that you use for groups. This defaults to "group".
  * Ideally a working domain, or at least a domain controller you can talk LDAP to.
  * Full distinguished name for any groups you want to associate with Django groups.

I'd highly recommend installing ADSIEdit on a windows desktop if you don't
already have it. This is an LDAP browser that lets you get at the full
distinguished names of objects. Its much easier to find the object in
ADSIEdit, and copy and paste than troubleshoot mistyped DNs! ADSIEdit is part
of the Windows Server 2003 support tools, you can get them here:
<http://www.microsoft.com/en-us/download/details.aspx?id=15326>.  
![ADSIEdit Screenshot](https://supportforums.cisco.com/sites/default/files/leg
acy/5/4/3/180345-adsiedit.png)

Now you need to add the following config to your django application
settings.py file. I'll go through what each bit does in the next section.

    
    
     import ldap from django_auth_ldap.config import LDAPSearch, NestedActiveDirectoryGroupType # Binding and connection options AUTH_LDAP_SERVER_URI = "ldap://domain.example:389" AUTH_LDAP_BIND_DN = "CN=Bind_User,OU=Users,DC=domain,DC=example" AUTH_LDAP_BIND_PASSWORD = "password" AUTH_LDAP_CONNECTION_OPTIONS = { ldap.OPT_DEBUG_LEVEL: 1, ldap.OPT_REFERRALS: 0, } # User and group search objects and types AUTH_LDAP_USER_SEARCH = LDAPSearch("OU=Users,DC=domain,DC=example", ldap.SCOPE_SUBTREE, "(sAMAccountName=%(user)s)") AUTH_LDAP_GROUP_SEARCH = LDAPSearch("OU=Groups,DC=domain,DC=example", ldap.SCOPE_SUBTREE, "(objectClass=group)") AUTH_LDAP_GROUP_TYPE = NestedActiveDirectoryGroupType() # Cache settings AUTH_LDAP_CACHE_GROUPS = True AUTH_LDAP_GROUP_CACHE_TIMEOUT = 300 # What to do once the user is authenticated AUTH_LDAP_USER_ATTR_MAP = { "first_name": "givenName", "last_name": "sn", "email": "mail" } AUTH_LDAP_USER_FLAGS_BY_GROUP = { "is_staff": ["CN=Djano_Users,OU=Groups,DC=domain,DC=example", "CN=Django_AdminUsers,OU=Groups,DC=domain,DC=example"] } AUTH_LDAP_FIND_GROUP_PERMS = True # The backends needed to make this work. AUTHENTICATION_BACKENDS = ( 'django_auth_ldap.backend.LDAPBackend', 'django.contrib.auth.backends.ModelBackend') 

Let's look at each section in more detail:

    
    
     AUTH_LDAP_SERVER_URI = "ldap://domain.example:389" AUTH_LDAP_BIND_DN = "CN=Bind_User,OU=Users,DC=domain,DC=example" AUTH_LDAP_BIND_PASSWORD = "password" 

First you need to specify the domain controller. In a properly configured
domain, its best to use the domain name and let DNS resolve this to a domain
controller using sites and services. This way if domain controllers are
changed over time, you won't have to update anything. This example is using
standard LDAP. I gave up trying to get LDAPS working, although this article is
worth a go… <http://www.djm.org.uk/using-django-auth-ldap-active-directory-
ldaps/>

The distinguished name (DN) should be provided for the account used to bind.
The idea is the bind account is used to do the initial lookup of the user, and
then the user credentials are used.

    
    
     AUTH_LDAP_CONNECTION_OPTIONS = { ldap.OPT_DEBUG_LEVEL: 1, ldap.OPT_REFERRALS: 0, } 

The Debug level should be set if you are struggling to get something working.
Details further down on how to actually get at the debug output. The
OPT_REFERRALS needs to be set to 0 and things just don't work without this…

    
    
     AUTH_LDAP_USER_SEARCH = LDAPSearch("OU=Users,DC=domain,DC=example", ldap.SCOPE_SUBTREE, "(sAMAccountName=%(user)s)") AUTH_LDAP_GROUP_SEARCH = LDAPSearch("OU=Groups,DC=domain,DC=example", ldap.SCOPE_SUBTREE, "(objectClass=group)") AUTH_LDAP_GROUP_TYPE = NestedActiveDirectoryGroupType() 

First of all, this section sets the OU in which to search for users. If there
are multiple places that users are held within your Active Directory there are
two options - either remove the OU and allow it to search the whole directory,
or have a read of the Search Unions section of the django-auth-ldap
documentation: https://pythonhosted.org/django-auth-ldap/authentication.html
#search-bind.

The example above uses a username in the sAMAccountName field, if you use
email for your username try this "(&amp;(objectClass=user)(mail=%(user)s))".

Finally, the group type should be set to either
NestedActiveDirectoryGroupType() or ActiveDirectoryGroupType() - make sure you
have imported these from django-auth-ldap.config at the top of your
settings.py file. ActiveDirectoryGroupType() is quicker, but will require your
users to live directly within any groups that you are mapping to Django
groups. NestedActiveDirectoryGroupType() is slower, but as its name suggests
lets you use nested groups.

    
    
     AUTH_LDAP_CACHE_GROUPS = True AUTH_LDAP_GROUP_CACHE_TIMEOUT = 300 

Django quite frequently makes LDAP requests. These two lines will cache group
details received from Active Directory to improve performance. If you do this,
bear in mind if you make an AD group change, you will have to wait for the
cache timeout before they will be reflected in Django.

    
    
     AUTH_LDAP_USER_ATTR_MAP = { "first_name": "givenName", "last_name": "sn", "email": "mail" } 

This section will populate the Django user meta data with fields from Active
Directory.

    
    
     AUTH_LDAP_USER_FLAGS_BY_GROUP = { "is_staff": ["CN=Djano_Users,OU=Groups,DC=domain,DC=example", "CN=Django_AdminUsers,OU=Groups,DC=domain,DC=example"] } 

The above will set the is_staff flag for a user if they are a member of one of
the groups specified. The full DN should be supplied for each group.

The is_staff flag needs to be set for a user to be able to view the admin
site. Using the example config, when a user logs in, even if they don't exist
in Django - a user will automatically be created. There are a couple of
options for controlling who is allowed to login:

  * Set your user search OU appropriate to restrict users.
  * Set the is_staff flag using a group, and put the users you want to access within this group. If you have multiple groups within Django that provide different permissions, create multiple AD groups and add them to the is_staff dictionary entry (as seen in the above example).
  * Read the Limiting Access section of the documentation for some very basic methods. https://pythonhosted.org/django-auth-ldap/groups.html
    
    
     AUTH_LDAP_FIND_GROUP_PERMS = True 

This command will automatically add users to appropriate Django groups based
on the AD group membership. The documentation here is very scarce. You need to
make sure that your AD group is named exactly the same as your Django group
for the mapping to work. When testing this, be careful, any group settings you
make through the admin interface manually will take priority over the AD group
mappings.

    
    
     AUTHENTICATION_BACKENDS = ( 'django_auth_ldap.backend.LDAPBackend', 'django.contrib.auth.backends.ModelBackend') 

Finally these lines are required for any of the above to work!

If something is not configured right you tend to get a standard error message:

> Please enter the correct username and password for a staff account. Note
that both fields are case-sensitive.

If you get a 500 server error message, chances are you have syntax problems in
your settings.py file.

You can add the following code to your settings.py file in order to help debug
issues:

    
    
     LOGGING = { 'version': 1, 'disable_existing_loggers': False, 'handlers': { 'mail_admins': { 'level': 'ERROR', 'class': 'django.utils.log.AdminEmailHandler' }, 'stream_to_console': { 'level': 'DEBUG', 'class': 'logging.StreamHandler' }, }, 'loggers': { 'django.request': { 'handlers': ['mail_admins'], 'level': 'ERROR', 'propagate': True, }, 'django_auth_ldap': { 'handlers': ['stream_to_console'], 'level': 'DEBUG', 'propagate': True, }, } } 

This can be daunting if you are not familiar with Django logging. The example
above will output django_auth_ldap debug messages to the console if you use
the Django Runserver functionality to run the webserver for your site. In
theory its possible to output to a file using a FileHandler, but I always
encountered 500 server errors when I tried this. If you are using apache. The
easiest thing to do is to stop apache, and start the django runserver in order
to see the messages.

> python /path/to/django/manage.py runserver x.x.x.x 80

Supplement with your server's IP address, or 127.0.0.1 if you are running on
localhost. Also make sure that ldap.OPT_DEBUG_LEVEL is set to 1 for messages
to appear.
