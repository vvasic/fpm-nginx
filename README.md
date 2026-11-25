# FPM/Nginx - [view on github](https://github.com/vvasic/fpm-nginx)

This container is intended to run Laravel applications and thus comes with a few items to assist:

 * [Composer](https://getcomposer.org)
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

The image runs nginx 1.30.8, built from the nginx.org source tarball checked against the SHA-256 in `Dockerfile`, on top of `snixx/php:7.4-fpm-alpine3.11-cve2026-nov`. That base is PHP 7.4 built by the recipe in `php74-cve/` from php-src commit 712e60f9ed (the October CVE-2026-109618/109619 backport), with one patch per CVE of the 19 Nov 2026 PHP security release applied on top:

| Patch in `php74-cve/patches/` | Fixes |
|---|---|
| `CVE-2026-126714.patch` | session files left partly overwritten (RCE) |
| `CVE-2026-115560.patch` | session serializer mismatch via upload progress (RCE); `session.upload_progress.enabled` now defaults to 0 |
| `CVE-2026-129289.patch` | PHP-FPM request smuggling through an unread body |
| `CVE-2026-129496.patch` | timezone cache use-after-free |
| `CVE-2026-129481.patch` | use-after-free after an interrupted request shutdown |
| `CVE-2026-129291.patch` | PHP-FPM worker pipe closed twice (epoll use-after-free) |
| `CVE-2026-129292.patch` | XSS on the PHP-FPM status page |
| `CVE-2026-128090.patch` | phar OpenSSL signature downgrade |
| `CVE-2026-128064.patch` | mixed-case `HTTPS://` and `FTPS://` URLs sent in plaintext |
| `CVE-2026-127960.patch` | folded response headers (7.4 has no affected code; the patch adds a test) |

PHP 7.4 is end of life, so these are 7.4 adaptations of the upstream php-src fixes. Each patch starts with the upstream commit it carries and how it was adapted. The build applies them with `git apply`, which stops on any hunk that does not match, and reports its version as `7.4.34-dev-nov2026` (the October build reported `7.4.34-dev`). `upload_progress.ini` also sets `session.upload_progress.enabled = 0` for the CLI and FPM.

`Dockerfile` starts `FROM snixx/php:7.4-fpm-alpine3.11-cve2026-nov`, pinned by digest. That base is on Docker Hub, so building this image needs no local base:

```
docker build --platform linux/amd64 -t snixx/s6-fpm-nginx:7.4-cve2026-nov -t snixx/s6-fpm-nginx:latest .
```

`--platform linux/amd64` is required on ARM hosts such as Apple Silicon: the Dockerfile installs the amd64 build of s6-overlay, and the published image is amd64.

To change the base, rebuild it from `php74-cve/`, push it under a new tag, and point the `FROM` line at the new tag and digest:

```
docker build --platform linux/amd64 -t snixx/php:<new-tag> php74-cve
docker push snixx/php:<new-tag>
```

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
