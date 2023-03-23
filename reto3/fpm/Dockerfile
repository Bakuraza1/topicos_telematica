FROM php:7.4-fpm

RUN docker-php-ext-install pdo_mysql && docker-php-ext-enable pdo_mysql
RUN docker-php-ext-install mysqli && docker-php-ext-enable mysqli
COPY php.ini-prod "$PHP_INI_DIR/php.ini"