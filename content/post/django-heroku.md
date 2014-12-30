+++
date = "2014-12-22T13:38:33-05:00"
draft = false
title = "Deploying Django to Heroku"
tags = [ "django", "tutorial"]

+++

For years, I've used Amazon EC2 and Digital Ocean for hosting my various web apps, but when I started playing around with blogs and other apps, I didn't want to spend a whole lot of time on Dev Ops. Since Heroku seems to be the big name in PaaS, I decided to check them out first. They have a very well written tutorial, however, it's not perfect, and it left me hanging a few times.

The Set Up:
-----------
When I started the Heroku tutorial, I already had my django blog working on my development machine inside a virtual environment, therefore I picked up the tutorial at the install step.  A simple pip install supposedly loads all the required packages on your machine, including gunicorn, dj-static (a server for serving static files), dj-database (database configuration helper) and Django if it's not already installed.  But as you will see later, I had to install some additional ruby gems to get things going.  I'm still not sure why.

    $ pip install django-toolbelt

Configure the settings and server files:
----------------------------------------
The next step was new to me.  I had to create a file with the name "Procfile" in my project root directory.  This file contains the instructions for Heroku to start your web app with a "web dyno".  A dyno is Heroku's word for a container that can run any command.  In this case, it will start the webserver.  The free tier includes 1 dyno at a time.   So, I created the file and added the following to it.

    web: gunicorn me.wsgi
where "me" is the inspired name for my blog project.

According to the tutorial, you should now be able to run

    $ foreman start
and it should run your procfile, and thus start Gunicorn and run your app. However, this resulted in an error message for me "command not found"...My computer had no clue who or what this "foreman" was.  After doing some digging, I found out that it and the Heroku command, which will be needed soon, are both installed with ruby gems.  So after running a couple commands, I got back up to speed.  If anyone could  tell me why this is and why it's not in the tutorial, I would be very grateful.

    $ sudo gem install heroku
	$ sudo gem install foreman
If you're familiar with virtualenv then you have already been doing the next step.

	$ pip freeze > requirements.txt
This creates a file in your project directory that lists all the dependencies for your django app.  When you push to Heroku later, it reads this file, and installs all the dependencies and installs them.

The next step is to paste the following code into the bottom of your settings.py file

{{% highlight python %}}
```
# Parse database configuration from $DATABASE_URL
import dj_database_url
DATABASES['default'] =  dj_database_url.config()

# Honor the 'X-Forwarded-Proto' header for request.is_secure()
SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')

# Allow all host headers
ALLOWED_HOSTS = ['*']

# Static asset configuration
import os
BASE_DIR = os.path.dirname(os.path.abspath(__file__))
STATIC_ROOT = 'staticfiles'
STATIC_URL = '/static/'

STATICFILES_DIRS = (
    os.path.join(BASE_DIR, 'static'),
)
```
{{% /highlight %}}
The problem with this is that simply pasting this into your settings.py file will allow your app to work on heroku, but it will break it in development.  The solution to this is to maintain two settings files, a settings.py for heroku, and a local_settings.py for development.

 

Next on the tutorial was a little unclear.  The instructions were to "add the following code to wsgi.py"

{{% highlight python %}}
```
from django.core.wsgi import get_wsgi_application
from dj_static import Cling

application = Cling(get_wsgi_application())
```
{{% /highlight %}}
The problem is, that after a default django install, by wsgi.py file already looked like:

{{% highlight python %}}
```
import os
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "me.settings")

from django.core.wsgi import get_wsgi_application
application = get_wsgi_application()
```
{{% /highlight %}}
So I figured that I should just add in the Cling function, and that turned out to be correct:  

{{% highlight python %}}
```
import os
from dj_static import Cling
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "me.settings")

from django.core.wsgi import get_wsgi_application
application = Cling(get_wsgi_application())
```
{{% /highlight %}}

Git Goin
--------
This part is pretty straight forward if you're familiar with git.  Heroku uses git to push your project to their servers.  It is highly advised that you create a .gitignore file in your project root directory, and add some lines to prevent any of the .pyc or your local_settings.py file from being sent to Heroku.

{{% highlight bash %}}
```
#.gitignore

venv
*.pyc
staticfiles
local_settings.py
```
{{% /highlight %}}

The following commands should be all you need to start a git repo, add the project files to it, and commit them:

	$ git init
	$ git add .
	$ git commit -m "first commit"

Deploy to Heroku
----------------
The moment you have been waiting for is here.  Create a new Heroku project with 

	$ heroku create

Push your code to your newly created heroku project

	$ git push heroku master
Start one dyno 

	$ heroku ps:scale web=1
This starts the dyno and runs the command in that procfile we made earlier.  Your app should now be live and running on the interweb. You can then check your websites status with the following command.

	$ heroku ps
Of course, when I ran this command, I got a message saying that my website had crashed shortly after take off. So I checked the error logs with the command 

	$ heroku logs
And it looked like heroku had no clue where my settings.py file was.  After some more digging I found out that I could explicity tell heroku where it was with the command 

	$ heroku config:add DJANGO_SETTINGS_MODULE=me.settings
Not sure why Heroku didn't know where the settings were on a default django install, but this fixed all my issues, and I was then able to visit my site with the simple command

	$ heroku open

Impressions
-----------
Even though I had a few issues, I managed to get through eveything relatively easily.  I would like to try out some other PaaS providers, but as of right now, I don't know if Heroku can be beat when it comes to a quality free tier and ease of use.