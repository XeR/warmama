# Warmama

Warmama is a game statistics application for the [Qfusion engine](http://qfusion.net).

## Deployment

Somethings about deploying warmama. Warmama works as a cgi process for http-server implemented on python
using web.py library as a helper to interface with cgi mechanisms.
Supported ways to run warmama currently are warmama as a self-contained local http-server for
small local testing, wsgi and fcgi process.

## Requirements

What you need (at least):
- python 2.x
- webpy (package-name?)
- MySQLdb (package-name? mysql python driver anyway)
- mysql (doh..)

For fcgi:
- cgi-fcgi (package?)

For deployment with apache through wsgi:
- apache + mod_wsgi

For deployment with nginx through uWSGI:
- nginx

## Configuration

Setup configuration in src/config.py. Select the right cgi_mode (local/fcgi/wsgi).
Database vars should be self-explanatory,

- getauth_url is the full url to POST request for authenticating players.
- auth_response_url is the full url of warmama appended with /auth.
- set alpha_phase to 0 on production deployment.

You need to create the database tables, for this theres utility script in database.
Make sure mysql is on, and then do this:
- cd warmama/src
- PYTHONPATH=. python database/models.py

Another utility script is the one that creates server accounts. For this we need the server's
ip (either in ipv4 or ipv6 (TEST!), and without port numbers). You need to register 1 account
per each instance of gameserver, so if server-admin wants to run 10 gameservers on his server,
we need to do this (again, make sure mysql is on):
- cd warmama/src
- PYTHONPATH=. python database/util_servergen.py <ip> <email> 10

this will create new accounts to the database and also print out all 10 authkey's that you can
report back to the serveradmin.

## Local testing

- cd warmama/src
- if auth server isnt on, setup up fakeauth in config.py (TODO: explain).
- python ./wcgi.py <port>
- set url in wsw server and client, +set mm_url localhost:<port>

## nginx + uwsgi

Probably the best method. An example .ini file for setting up warmama as an uWSGI application is provided.

Add the following section to your nginx.conf:

```
	upstream uwsgi_mm_cluster {
		server 127.0.0.1:9001;
	}
```

And the corresponding server section:

```
server {
    listen 1337;

    server_name mm.example.org;

    # Proxying connections to application servers
    location / {
        include            uwsgi_params;
        uwsgi_pass         uwsgi_mm_cluster;

        proxy_redirect     off;
        proxy_set_header   Host $host;
        proxy_set_header   X-Real-IP $remote_addr;
        proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Host $server_name;
    }
}
```

Running the application:
```
uwsgi /home/hc/warmama/uwsgi-warmama.ini
```

## FCGI

FCGI method, probably the simplest one. Theres a script in warmama/ called cgi_ctrl.sh that you can
use to control fcgi process. Currently it does 1 process only but theres comments inside to hint you how
to modify it to launch more processes. Script has 4 functionalitys, give these as an argument:
start restart stop check.
Check sees if the process is alive and if not, it starts it up.

Apache configuration should be something like this (from http://webpy.org/install#apachefastcgi)
Add this to your .htaccess:

```	
	<Files wcgi.py>      SetHandler fastcgi-script
	</Files>
```

Also instead of having to call <url>/wcgi.py, you can use mod_rewrite on apache to fix the url.
Check [here](http://webpy.org/install#apachemodrewrite) for more info.

## Apache wsgi

- install mod_wsgi and enable it with this in httpd.conf

```	
	LoadModule wsgi_module modules/mod_wsgi.so
```	

- heres a snippet on how i made <hostname>/warmama as the base for warmama's http-requests:

```
## one way of setting up wsgi
## conf/extra/httpd-warmama.conf:

WSGIScriptAlias /warmama /home/hc/warmama/src/wcgi.py
WSGIPythonPath /home/hc/warmama/src/

# Alias /appname/static /var/www/webpy-app/static/
AddType text/html .py

<Directory /home/hc/warmama/src/>
  Order deny,allow
  Allow from all
</Directory>
```

warmama could be run in daemon mode here too, refer to mod_wsgi documentation. (TODO: write more about
it in here), but basic things required here are using WSGIScriptAlias to reference the script by custom path,
setting the PYTHONPATH environment variable with WSGIPythonPath (this can be done as an option
in WSGIDaemonProcess directive). 
