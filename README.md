# Docker with PHP

Running PHP in docker is pretty trivial, once you have docker installed:

* https://www.docker.com/docker-mac
* https://www.docker.com/docker-windows


You can rn the following command:

`docker run --rm --name=php php:7.2 /usr/local/bin/php -v`

You should see:
 
```sh
PHP 7.2.3 (cli) (built: Mar 22 2018 21:24:10) ( NTS )
Copyright (c) 1997-2018 The PHP Group
Zend Engine v3.2.0, Copyright (c) 1998-2018 Zend Technologies
```

What this does is as follows:

* `docker run` is the primary command to get Docker to create a container
* `--rm` will remove the container after it's been run
* `--name=php` this is any arbitrary name, it's optional and Docker will make up a name if you don't supply it
* `php:7.2` tells Docker what image to use for the container, this is an upstream image available publicly
[https://hub.docker.com/_/php/] https://hub.docker.com/_/php/
* `/usr/local/bin/php` tells the container to run that as the foreground process
* `-v` is an argument passed to the foreground process `php`, this tells php to output the version

That's it, you've now run PHP in docker, and this will work exactly the same way on ALL machines that run docker, neat!


## Running PHP Files

Just outputting the PHP version isn't a super helpful app to be honest, let's get some PHP files in there.

Lets make a hello world PHP file:
```sh
mkdir -p src/public
echo '<?php echo "hello world\\n";' > src/public/index.php
```

Now run:

`docker run --rm --name=php --volume=$(pwd)/src/:/var/www/html php:7.2 /usr/local/bin/php /var/www/html/public/php`

You should see `hello world` output to your console.

Notice two additions:

* `--volume` "mounts" the `src` directory inside the container to `/var/www/html`
* `/var/www/html/public/php` in place of the `-v` tells php to execute the hello world script passed to it


## Custom Extensions

You will most likely want to install some custom extensions in the container, things like `gd` or `iconv` for example.
You can do this easily, but it requires making your own `Dockerfile`. A `Dockerfile` is like a blueprint for an image,
it tells docker to run custom commands, copy files into the image, set environment variables, etc. 

So, make a `Dockerfile` in the root of the project, with the content:

```
FROM php:7.2

RUN apt-get update && \
    apt-get install -y libmemcached-dev zlib1g-dev && \
    docker-php-ext-install pdo pdo_mysql && \
    pecl install \
        xdebug-2.6.0 \
        memcached-3.0.4

RUN docker-php-ext-enable xdebug memcached
```

This will install the packages required for xdebug, memcache and pdo_mysql.

Now, build an image: `docker build -t php-docker .`

You'll get a tonne of logging to your console from installing all the thing, but eventually it should output at the end:

```sh
Successfully built 049438a38eb5
Successfully tagged php-docker:latest
```

Now run: `docker run --rm --name=php --volume=$(pwd)/src/:/var/www/html php-docker /usr/local/bin/php -m`

You should see:

```
pdo_mysql
memcached
Xdebug
```

All listed in the output.

## XDebug

Configuring XDebug in Docker can be a little tricky. I'm only going to cover getting it working in PHPStorm.

Add run in your terminal:

```sh
mkdir -p etc/php
echo "[Xdebug]
xdebug.remote_enable=1
xdebug.idekey=PHPSTORM_XDEBUG
xdebug.max_nesting_level=1000
xdebug.remote_host=docker.for.mac.host.internal" > etc/php/php.ini
```

This will create a basic XDebug config that will allow it to connect back to the local host via the "remote" debugging
functionality.

Now, in PHPStorm, go to Preferences > Languages & Frameworks > PHP > Servers

Hit the `+` button, and add:

* Name: `PHPSTORM_DOCKER`
* Host: `localhost`
* Check `use path mappings`
* Set `/var/www/html` for `src`

If you're familiar with XDebug in PHPStorm, you should be able to enable the listener, put a breakpoint in the `index.php`
file, and run:

```
docker run --rm --name=php --volume=$(pwd)/src/:/var/www/html --volume=$(pwd)/etc/php/php.ini:/usr/local/etc/php/conf.d/custom.ini -e PHP_IDE_CONFIG=serverName=PHPSTORM_DOCKER php-docker /usr/local/bin/php -d xdebug.remote_autostart=1 /var/www/html/public/index.php
```

* `--volume=$(pwd)/etc/php/php.ini:/usr/local/etc/php/conf.d/custom.ini` Adds in our custom PHP config to enable XDebug
* `-e PHP_IDE_CONFIG=serverName=PHPSTORM_DOCKER` Sets an environment variable required to make XDebug work
* `-d xdebug.remote_autostart=1` Sets a PHP ini variable to enable auto-starting XDebug, you need this for CLI

You should see the XDebug dialog box appear in PHPStorm and stop at the break point.

## Docker Compose

As you may have noticed, the command is getting kind of long now, and we will also probably want to add in some other
services into the mix, like `nginx`, `php-fpm`, `mysql` and `memcache`.

To do this, we can use a tool called `docker-compose`. It allows us to configure a set of containers that can all talk
to each other, and provides a handy place for all our configuration options.

So, go ahead and create a `docker-composer.yaml` in the root of this project with the contents:

```yaml
version: '3.3'

services:

  php:
    build: ./
    volumes:
      - "./src:/var/www/html"
      - "./etc/php/php.ini:/usr/local/etc/php/conf.d/custom.ini"
    environment:
      - "PHP_IDE_CONFIG=serverName=PHPSTORM_DOCKER"
    command: "php -d xdebug.remote_autostart=1 /var/www/html/public/index.php"
```

This should all make sense, we're telling it to use a out custom docker file to build the image, to make 2 volume mounts,
set an environment variable for xdebug, and run the index file command.

Now run `docker-compose build` to have it build a new image in case it needs that.

Then run: `docker-compose up` and you should see:

```sh
Recreating phpdocker_php_1 ... done
Attaching to phpdocker_php_1
php_1  | hello world
phpdocker_php_1 exited with code 0
```

Great, `docker-compose` has create a container for us, called `php_1`


## Serving web traffic

Running a single PHP script is great and all, but I'm getting you probably want to run a website, rather than a sing
PHP script.

To do this, we're going to use `nginx` and `php-fpm`.

Update the `docker-compose.yaml` to the following:

```yaml
version: '3.3'

services:
  nginx:
    image: nginx:1.13.11
    restart: always
    volumes:
      - "./etc/nginx/default.vhost.conf:/etc/nginx/conf.d/default.conf"
      - "./src:/var/www/html"
    ports:
      - "8000:80"
    depends_on:
      - php

  php:
    build: ./dockerfiles/php
    restart: always
    volumes:
      - "./src:/var/www/html"
      - "./etc/php/php.ini:/usr/local/etc/php/conf.d/custom.ini"
    environment:
      - "PHP_IDE_CONFIG=serverName=PHPSTORM_DOCKER"
```

What this does it start 2 services, `php` and `nginx`, and exposes port `8000` on localhost to map to port `80` in the
container, which is the default port that Nginx is listening on.

```sh
mkdir -p dockerfiles/php
mv Dockerfile dockerfiles/php/
mkdir -p etc/nginx
touch etc/nginx/default.vhost.conf
```

Open `dockerfiles/php/Dockerfile` and set the `FROM` line at the top, to: `FROM php:7.2-fpm`

Now set the `etc/nginx/default.vhost.conf` to the following:

```
# Nginx configuration

server {
    listen 80 default_server;
    listen [::]:80 default_server;

    index index.php index.html;
    error_log  /var/log/nginx/error.log;
    access_log /var/log/nginx/access.log;
    root /var/www/html/public;

    location ~ \.php$ {
        try_files $uri =404;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass php:9000;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
    }
}
```

This is just a simple nginx configuration that will pass requests for PHP files to php-fpm that will run the PHP process,
and set the default index file to `index.php`.

Now run `docker-compose build` to build a new image, then `docker-compose up`, you should see:

```sh
phpdocker_php_1 is up-to-date
Recreating phpdocker_nginx_1 ... done
Attaching to phpdocker_php_1, phpdocker_nginx_1
php_1    | [04-Apr-2018 21:36:32] NOTICE: fpm is running, pid 1
php_1    | [04-Apr-2018 21:36:32] NOTICE: ready to handle connections
php_1    | 172.18.0.3 -  04/Apr/2018:21:36:44 +0000 "GET /index.php" 200
php_1    | 172.18.0.3 -  04/Apr/2018:21:37:16 +0000 "GET /index.php" 200
```

Now hit `http://localhost:8000` and you should see `hello world` in your browser 🎉

You can even try `http://localhost:8000/?XDEBUG_SESSION_START` and the breakpoint you sent previously should also be hit!

