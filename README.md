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
echo 'hello world' > src/public/index.html
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
