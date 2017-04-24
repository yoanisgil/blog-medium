# Python + Docker: From development to production: Episode I

For the last two years or so I've been using Docker to deliver almost every single project I've been involved with. It's been a long journey where I've had the opportunity to learn an interesting set of new technologies and most importantly how to get code shipped to production in a reliable, predictable and trustable way.

[Python](https://www.python.org/) is the language I feel most comfortable with, which means that those projects (mostly web applications or web APIs) have been developed with heavy usage of frameworks and libraries available under the language's ecosystem.

I'll be sharing my journey this far in a series of blog posts (hopefully only two). The first one, which I named **Episode I** focuses on the development experience. **Episode II** is an overview of the different Cloud solutions I've worked with for deploying Docker applications and how to run the demo application on container friendly environment.

# The Application

For the purpose of this demo we're going to use a very simple [Flask](http://flask.pocoo.org/) application named [Easy GeoIP](https://github.com/yoanisgil/easygeoip). The application comprises three main components:

- An endpoint which takes-in a Domain Name or IP Address and uses the MaxMind City database to return all of the information associated to the given Domain Name/IP Address.
- A web page, which will present the information returned by the endpoint mentioned above, in a user-friendly way (see screenshot below).
- A PostgresSQL database which is used to fill-in the timezone for the provided Domain Name or IP Address, when not available in the MaxMind City Database.
    
Below a high-level view of the application architecture:

![Easy GeoIP Architecture](https://raw.githubusercontent.com/yoanisgil/medium-blog/master/python%2Bdocker/episode-i/assets/application-architecture.png)

and then you can enter any IP Address/Domain Name of your choice and you should something like this:

![Easy GeoIP Screenshot](https://raw.githubusercontent.com/yoanisgil/medium-blog/master/python%2Bdocker/episode-i/assets/easy-geoip-resolve.png)

**NOTE**: The PostgresSQL database was generated from the Shapefiles available [here](http://efele.net/maps/tz/world/). For more details on how this database was created take a look at this [link](https://github.com/yoanisgil/tz_world)

# Running the Python App for the first time

Let's start with the basis and get the application running. First we need to clone the application repository and checkout the Git tag we will be using through this article:

    - git clone https://github.com/yoanisgil/easygeoip.git
    - git checkout blog-episode-i

We can now build the Docker which we will later use to create the container running the application:
    
    - docker-compose build app

this might take a few mintues, but once the build is finished you should see something like this:

```bash
Removing intermediate container 4e33a37b9b69
Step 16/16 : CMD supervisord -u www-data -n -c /etc/supervisor/supervisord.conf
 ---> Running in 2f8f80a4405d
 ---> 368383a4d379
Removing intermediate container 2f8f80a4405d
Successfully built 368383a4d379
```

with the image built, we can now launch the application:

    - docker-compose up
        app_1          |  * Running on http://0.0.0.0:5000/ (Press CTRL+C to quit)
        app_1          |  * Restarting with stat
        tz_world_db_1  | LOG:  database system was shut down at 2017-04-20 10:53:35 UTC
        tz_world_db_1  | LOG:  MultiXact member wraparound protections are now enabled
        tz_world_db_1  | LOG:  autovacuum launcher started
        tz_world_db_1  | LOG:  database system is ready to accept connections
    
after which you can now go visit http://localhost:5000, and you should see something like this:

![Easy GeoIP First Time Launch](https://raw.githubusercontent.com/yoanisgil/medium-blog/master/python%2Bdocker/episode-i/assets/easy-geoip-first-time.png)

# First Dive into The Application

Ok, that was quick. In just a matter of minutes we were able to have a fully working application and with very little effort. Let's see how was this possible. The very first thing that we did was to run:

    - docker-comspoe build app

This command is telling Docker Compose to build the service app, which is defined as follows in  the [docker-compose.yml file](https://github.com/yoanisgil/easygeoip/blob/blog-episode-i/docker-compose.yml):

Before we take an in-depth look to the Docker Compose file let's go through the Dockerfile we're using to build the application's image:

```Dockerfile
FROM python:2.7-slim
MAINTAINER Yoanis Gil<gil.yoanis@gmail.com>

# Create the directory where we will copy the application code
RUN mkdir /srv/app
# ... and declare it as the default working directory
WORKDIR /srv/app

# Add Nginx repository so that we install a recent version
ADD ./docker/nginx.list /etc/apt/sources.list.d/nginx.list

# Install required software:
# - Supervisor (for running multiple processes)
# - Nginx
# - libpq-dev because we need to install PostgresSQL's binding for Python
RUN apt-get update && \
    apt-get -y install curl && \
    curl http://nginx.org/keys/nginx_signing.key | apt-key add - && \
    apt-get update && \
    apt-get -y install build-essential libpq-dev nginx supervisor && \
    rm -rf /var/lib/apt/lists/*

# Add Nginx configuration
ADD ./docker/nginx.conf /etc/nginx/nginx.conf
# Add supervisor which will take care of running Nginx and UWSGI
ADD ./docker/supervisord.conf /etc/supervisor/supervisord.conf
# Remove the Nginx's default configuration so that it doesn't create conflicts with our app's
RUN rm /etc/nginx/conf.d/default.conf
# Add the Nginx configuration related to the application. Nothing special here, just standard UWSGI configuration
ADD ./docker/site.conf /etc/nginx/conf.d/site.conf

# Because Nginx is not running as root, we need to make sure that we create the folder where the user www-data can read/write from/to.
RUN mkdir -p /tmp/nginx /tmp/app/logs && chown -R www-data: /tmp/nginx /tmp/app/logs

# Copy as early as possible so we can cache ...
ADD ./requirements.txt /srv/app/requirements.txt

# Install application dependencies
RUN pip install  --no-cache-dir -r requirements.txt

# Add the application
ADD . /srv/app

# Download Maxmind Database
RUN python cli.py update_maxmind_city_db

# Application entrypoint
CMD ["supervisord", "-u", "www-data", "-n", "-c", "/etc/supervisor/supervisord.conf"]
```
The Dockerfile it's pretty explanatory by itself, however let me highlight some of the most important elements present in it:

 - We're using `python:2.7-slim` as our base image since we want to keep our Docker image as light as possible. This is very handy specially when it comes to deployment since the image needs to be pulled before been deployed.
 - Those instructions which are not supposed to change very often (apt-get install, apt-get update, etc) are added very early in the file so that we can benefit from the caching mechanisms provided by Docker.
- The `requirements.txt` file is added indepndently from the application code. This is again, to speed-up the build process since app's dependencies should not change that often.
- The container entrypoint is [supervisord](http://supervisord.org/). The reason we need `supervisord` is because our container needs to run both NGINX and uWSGI. Also because other many goodies that comes with `supervisord`, like that it can act as a process reaper, make sure processes are always running, etc. 
- Neither NGINX, nor uWSGI are running as root (not even `supervisord`). They all run as the user `www-data`.

With the Dockerfile now explained, let's now break down the Docker Compose file:

```yaml
    app:
      image: easygeoip_app
      build: .
      command: ["python", "/srv/app/main.py"]
      volumes:
        - .:/srv/app
      ports:
       - "5000:5000"
      environment:
        - DB_PASSWORD=thepassword
        - DEBUG=1
```

|Instruction   | Comments  |
|--------------|-----------|
|`image: easygeoip_app`   |Name of the docker image after build   |
| `build: .`  |  Instruct docker compose to build the image with the Dockerfile under the current  directory|
|`command: ["python", "/srv/app/main.py"]`   |  Change the default command so that the application can be restarted whenever the source code is updated |
| `volumes`  |  Mount the current directory at `/srv/app` in the application's container. This is very handy, since an update to the source code won''t require rebuilding the Docker Image (and hence restarting the application container) |
| `ports`  | Map port `5000` on the container to port `5000`on the host. default all Flask's application use port `5000` the web server port so we're just making sure that it's accesible.   |
|  `environment` | Here were passing the environment variables required by the application. `DEBUG=1` it's very important since Flask will restart the application whenever the source code is updated.    |

Go ahead, play with the application, modify it and play with it. You will see the development experience it's quite the same as if the application was running outside Docker :).

# Bonus Track: Debugging the Application with PyCharm

So say that there is a bug in the application, a very tricky and difficult one. For sure you could debug the applicaiton using `print` statments or any of the available Flas/Python debuggers ([here](http://werkzeug.pocoo.org/docs/0.11/debug/) and [here](https://docs.python.org/2/library/pdb.html)). 

Personally I've been using Intellij's PyCharm EAP for the last 3 years or so, mainly because of the consistent development experience between their other products (PHPStorm, WebStorm and AppCode), but also because the Docker support is very good.













