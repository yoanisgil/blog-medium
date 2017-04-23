# Python + Docker: From development to production: Episode I

For the last two years I've been using Docker to deliver almost every single project I've been working on.It's been a long journey where I've had the opportunity to learn an interesting set of new technologies and most importantly how to get code shipped to production in a reliable, predictable and trustable way.

Python it's the language I feel most comfortable with, which means that those projects, which has been mostly web applications or web APIs, have been developer with heavy usage of frameworks and libraries available under the language's ecosystem.

I'd like to share my journey this far, and well it all starts with the development process. I'll split this post into two main articles:

- **Episode I** (i.e this article): Focus on setting up a development environment, powered by Docker/, and will cover as well a glance at Continuous Integration/Continuous Delivery
- **Episode II**: An overview of the different Cloud solutions I've worked with for deploying Docker applications.

# The Application

For the purpose of this demo we're going to use a very simple [Flask](http://flask.pocoo.org/) application named [Easy GeoIP](https://github.com/yoanisgil/easygeoip). The application comprises three main components:

    - An endpoint which takes-in a Domain Name or IP Address and uses the MaxMind City database to return all of the information associated to the given Domain Name/IP Address.
    - A web page, which will present the information returned by the endpoint mentioned above, in a user-friendly way (see screenshot below).
    - A PostgresSQL database which is used to fill-in the timezone for the provided Domain Name or IP Address, when not available in the MaxMind City Database.
    
![Easy GeoIP Screenshot](https://raw.githubusercontent.com/yoanisgil/medium-blog/master/python%2Bdocker/episode-i/assets/easy-geoip-resolve.png)

The PostgresSQL database was generated from the Shapefiles available [here](http://efele.net/maps/tz/world/). For more details on how this database was created take a look at this [link](https://github.com/yoanisgil/tz_world)

# Running the Python App for the first time

Let's start with the basis and get the application running. First let''s clone the repository and checkout the Git tag we will be using through this article:

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

```yml
version: '2'

services:
    app:
      build: .
      image: easygeoip_app
      command: ["python", "/srv/app/main.py"]
      volumes:
        - .:/srv/app
      ports:
       - "5000:5000"
      environment:
        - DB_PASSWORD=thepassword
        - DEBUG=1
```

Before we take an in-depth look to the Docker Compose file let's go through the Dockerfile we're using to build the application's image:

```Dockerfile
FROM python:2.7-slim
MAINTAINER Yoanis Gil<gil.yoanis@gmail.com>

RUN mkdir /srv/app
WORKDIR /srv/app

# Install require software and libraries
ADD ./docker/nginx.list /etc/apt/sources.list.d/nginx.list

RUN apt-get update && \
    apt-get -y install curl && \
    curl http://nginx.org/keys/nginx_signing.key | apt-key add - && \
    apt-get update && \
    apt-get -y install build-essential libpq-dev nginx supervisor && \
    rm -rf /var/lib/apt/lists/*

ADD ./docker/nginx.conf /etc/nginx/nginx.conf
ADD ./docker/supervisord.conf /etc/supervisor/supervisord.conf
RUN rm /etc/nginx/conf.d/default.conf
ADD ./docker/site.conf /etc/nginx/conf.d/site.conf

# Create nginx temporary folders.
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
