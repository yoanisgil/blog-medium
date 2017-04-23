# Python + Docker: From development to production: Episode I

For the last two years I've been using Docker to deliver almost every single project I've been working on.It's been a long journey where I've had the opportunity to learn an interesting set of new technologies and most importantly how to get code shipped to production in a reliable, predictable and trustable way.

Python it's the language I feel most comfortable with, which means that those projects, which has been mostly web applications or web APIs, have been developer with heavy usage of frameworks and libraries available under the language's ecosystem.

I'd like to share my journey this far, and well it all starts with the development process. I'll split this post into two main articles:

- **Episode I** (i.e this article): Focus on setting up a development environment, powered by Docker/, and will cover as well a glance at Continuous Integration/Continuous Delivery
- **Episode II**: An overview of the different Cloud solutions I've worked with for deploying Docker applications.

# The Application

For the purpose of this demo we're going to use a very simple application: [Easy GeoIP](https://github.com/yoanisgil/easygeoip) which three main components:

    - An endpoint which takes-in a Domain Name or IP Address and uses the MaxMind City database to return all of the information associated to the given Domain Name/IP Address.
    - A web page, which will present the information returned by the endpoint mentioned above, in a user-friendly way (see screenshot below).
    - A PostgresSQL database which is used to fill-in the timezone for the provided Domain Name or IP Address, when not available in the MaxMind City Database.
    
![Easy GeoIP Screenshot](https://raw.githubusercontent.com/yoanisgil/medium-blog/master/python%2Bdocker/episode-i/assets/easy-geoip-screenshot.png)

The PostgresSQL database was generated from the Shapefiles available [here](http://efele.net/maps/tz/world/). For more details on how this database was created take a look at this [link](https://github.com/yoanisgil/tz_world)
    

# Setting up the development environment

When it comes to development environments there is an endless list of options (as well as endless discussion as to which one is the best or better). Personally I've been using Intellij's PyCharm EAP for the last 3 years or so, mainly because of the consistent development experience between their other products (PHPStorm, WebStorm and AppCode), but also because the Docker support is very good.

