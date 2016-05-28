# Config Python Server

These are instructions for serving **Django** and **Flask** applications in the simplest form. These are based on an Ubuntu Server.

**For these examples I will use:**
- **ubuntu** as my linux user
- **www-data** as the default server group
- **foo** as my **Flask or Django** project name

## Setup Your Project

1: Your server needs a virtual environment. I will be using **virtualenvwrapper**.

```
sudo apt-get install virtualenvwrapper
```

My **project path** will be `/home/ubuntu/projects/foo`

2: So the first thing to do is make the virtualenvwrapper:

```
mkdir ~/projects && cd ~/projects
mkvirtualenv foo
(or, if using my dotfiles below):
mkproject foo
workon foo 
```

3: Place your Django or Flask project in `/home/ubuntu/projects/foo` 

## Setup Nginx
1: First install nginx, and make a new file for our virtual host.
```
sudo apt-get install nginx
sudo touch /etc/nginx/sites-available/foo
sudo ln -s /etc/nginx/sites-available/foo/etc/nginx/sites-enabled/foo
```

2a: For **Flask**, Add the following
```
server {
    location / {
        proxy_pass http://localhost:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
    location /static {
        alias  /home/ubuntu/projects/foo/static/;
    }
}
```

2b: For **Django** Add the following (Whatever your Paths are):
```
server {
    location /media  {
        alias  /home/ubuntu/projects/foo/foo/static/;
    }

    location /static {                                                         
        alias  /home/ubuntu/projects/foo/foo/static/;
    }

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

3: Restart Nginx
```
sudo service nginx restart
```

## Setup Gunicorn
This is the Python WSGI HTTP server.

1: You should have a **requirements.txt** of some sort in your project that you made for PIP packages. Simply include **gunicorn** in your project's requirements.txt _(or include a version like I do below):_
```
...
gunicorn==19.4.5
```

2: Go to your project, and **make sure you are in the virtual enviroment** to install it.
```
workon foo
pip install -r requirements.txt
```

#### Test Gunicorn
To manually test while still in your virtual environment do one of the following commands below. Then load your webserver's IP address or domain name if you pointed it. _(Services like AWS need you to open port 80 in your security policy for HTTP)_

3a: For **Flask**, if you created a default `app` module, simply run it as:
```
gunicorn app:app --bind localhost:8000

```

3b: For **Django**, go to the folder your **wsgi.py** file is and run:
```
gunicorn --bind localhost:8000
```

4: If all is working as it should you can `CTRL+C` to cancel it. If you have problems, see how you run it here:  http://docs.gunicorn.org/en/stable/run.html

### Setup Supervisor
Supervisor is a **process controller** that makes it easy to manage running programs/tasks and keep them alive.

1: Install supervisor globally.
```
sudo apt-get install supervisor
```

2: Create a project file for supervisor
```
sudo vim /etc/supervisor/conf.d/foo.conf
```

3a: Enter the following for **Flask**:
```
[program:foo]
command = /home/ubuntu/.virtualenvs/foo/bin/gunicorn app:app --bind 0.0.0.0:8000
directory = /home/ubuntu/projects/foo
autorestart=true

```

3b: Enter the following for **Django**:
```
[program:foo]
command = /home/ubuntu/.virtualenvs/foo/bin/gunicorn --bind 0.0.0.0:8000
directory = /home/ubuntu/projects/foo
autorestart=true
```

### Lets Reload Everything 
1: Let's **stop any gunicorn** instances and **stop supervisor** to make sure that it loads the default configuration file and we start with a clean slate

```
sudo pkill gunicorn
sudo service supervisor stop
sudo unlink /var/run/supervisor.sock
```

2: Load the default configuration for supervisor and re-read the folders

```
sudo supervisord -c /etc/supervisor/supervisord.conf
```

3: Now update supervisor and start our application
```
sudo supervisorctl update
sudo supervisorctl start foo
```

4: **You are done.** To see many of the options provided in the CLI or configuration files visit: http://supervisord.org/running.html 

Note there are two CLI commands `supervisor` and `supervisorctl`.

### Important Supervisor Commands to Know

First, `sudo supervisorctl reread` - Only updates the changes. It does not restart any of the managed applications, even if their configuration has changed. New application configurations cannot be started with this command.

Second, `sudo supervisorctl update` - Restarts the applications whose configuration has changed. After the update command, new application configurations becomes available to start, but do not start automatically until the supervisor service restarts or system reboots (Even if autostart is on) you still need to run it.

Third, `sudo supervisorctl start foo` to run the program.
`sudo supervisorctl stop foo` to stop the program.

To debug a problem, either check the job, or check the logs:
```
supervisorctl tail foo
cd /var/log/supervisor/  # tail logs here.
```


## Optional Dotfiles I Use
_(Skip this if you don't need it)_ If you'd like to see my **dotfiles** which make **virtualenvwrapper** easier to manage, visit my [config-ubuntu](https://github.com/JREAM/config-ubuntu/tree/master/files) repository. 

On your server, you can do the following:

```
git clone git@github.com:JREAM/config-ubuntu.git
cd config-ubuntu
sudo ./install.sh
Type a Command: dot
```

Once that is complete, make sure to install the VIM plugins.
```
vim +PluginInstall +qall
```


motivated by [Dan Sackett](https://github.com/dansackett). 

 You can simply copy the `.virtualenvs` folder and the `.bashrc` file where you see the block called **PYTHON**. 

First, place your project on your web server. I prefer my applications in `~/projects/<projectname>` the absolute path is `/home/<username>/<projectname>`

