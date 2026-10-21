# FPM/Nginx - [view on github](https://github.com/vvasic/fpm-nginx)

This container is intended to run Laravel applications and thus comes with a few items to assist:

 * [Composer](https://getcomposer.org) (with [hirak/prestissimo](https://github.com/hirak/prestissimo) for parallel dependency installation)
 * PHP Extensions
   * [mbstring](http://php.net/manual/en/book.mbstring.php)
   * [pdo_mysql](http://php.net/manual/en/ref.pdo-mysql.php)
   * [gd](http://php.net/manual/en/book.image.php)
   * [opcache](http://php.net/manual/en/book.opcache.php) - Automatically enabled when `APP_ENV=production`
   * [xdebug](https://xdebug.org) - Only enabled when `XDEBUG_ENABLE=true`.  Only installed when `XDEBUG_ENABLE=true` when building the container
 * Adding a default virtual host serving apps from `/var/www/html/public`
   
For Laravel applications, see [bkuhl/laravel-fpm-nginx](https://github.com/bkuhl/laravel-fpm-nginx).

For a container to run cron and other CLI tasks, check out [bkuhl/php](https://github.com/bkuhl/php).

**Why 2 processes in 1 container?**

 1. DNS issues - Both the fpm/nginx containers need to be redeployed when your application is updated.  Nginx maintains an internal DNS cache, so while Docker may ensure zero downtime for fpm containers, nginx's internal workings can still create problems.  The only way to solve this (that I've found) is to restart the nginx process.  Having them on the same container eliminates the problem.
 2. Laravel Mix - The front/backend of applications are kind of coupled when using something like Laravel Mix.  The index of assets it creates need to be on both containers and running these separately is possible, but redundant.  

## Adding Processes

This container uses [S6 Overlay](https://github.com/just-containers/s6-overlay) as it's process monitoring solution.  Add a new directory to `services.d` with a `run` file in it where `run` in a shell script that kicks off the process.  The rest is taken care of for you.

## Building

`Dockerfile` starts `FROM local/php:7.4-fpm-alpine3.11-cve2026`, which is not on any registry. Build that base from `php74-cve/` first (PHP 7.4 from php-src commit 712e60f9ed, the CVE-2026-109618/109619 backport), then this image on top of it:

```
docker build --platform linux/amd64 -t local/php:7.4-fpm-alpine3.11-cve2026 php74-cve
docker build --platform linux/amd64 -t snixx/s6-fpm-nginx:latest .
```

`--platform linux/amd64` is required on ARM hosts such as Apple Silicon: the Dockerfile installs the amd64 build of s6-overlay, and the published image is amd64. Run both builds with the default `docker` builder so the second one finds the base image locally.

## Example Dockerfile

```
FROM snixx/s6-fpm-nginx:latest

WORKDIR /var/www/html

# Copy the application files to the container
ADD --chown=www-data:www-data  . /var/www/html

USER www-data

    # production-ready dependencies
RUN composer install  --no-interaction --optimize-autoloader --no-dev --prefer-dist \

    # keep the container light weight
    && rm -rf /home/www-data/.composer/cache

USER root
```

[Forked from bkuhl](https://hub.docker.com/r/bkuhl/fpm-nginx/) to add freetype/jpeg/png support for gd and nginx options
