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
    
![Easy GeoIP Screenshot](https://raw.githubusercontent.com/yoanisgil/medium-blog/master/python%2Bdocker/episode-i/assets/easy-geoip-screenshot.png)

The PostgresSQL database was generated from the Shapefiles available [here](http://efele.net/maps/tz/world/). For more details on how this database was created take a look at this [link](https://github.com/yoanisgil/tz_world)

# Setting up the development environment

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
    


