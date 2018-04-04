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

