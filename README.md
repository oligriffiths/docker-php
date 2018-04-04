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

