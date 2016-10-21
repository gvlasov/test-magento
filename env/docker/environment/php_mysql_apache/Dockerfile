FROM php:5.6-fpm

# Add a repository that has mysql-server-5.6;
RUN echo "deb http://httpredir.debian.org/debian/ jessie-backports main contrib non-free" >> /etc/apt/sources.list
# Add contrib and non-free to be able to install libapache-mod-fastcgi
# instead of libapache2-mod-fcgid
RUN sed -i -e 's/debian jessie main/debian jessie main contrib non-free/' /etc/apt/sources.list

# Install php, mysql, apache and supervisord
RUN apt-get update \
  && echo "mysql-server mysql-server/root_password password pw" | debconf-set-selections  \
  && echo "mysql-server mysql-server/root_password_again password pw" | debconf-set-selections  \
  && apt-get install -y \
    -t jessie-backports \
    apache2 \
    libapache2-mod-fastcgi \
    supervisor \
    cron \
    libfreetype6-dev \
    libicu-dev \
    libjpeg62-turbo-dev \
    libmcrypt-dev \
    libpng12-dev \
    libxslt1-dev \
    mysql-server-5.6 \
    mysql-client-5.6

# Install php extensions
RUN docker-php-ext-configure \
  gd --with-freetype-dir=/usr/include/ --with-jpeg-dir=/usr/include/
RUN docker-php-ext-install \
  gd \
  intl \
  mbstring \
  mcrypt \
  pdo_mysql \
  soap \
  xsl \
  zip

# Configure php
COPY conf/www.conf /usr/local/etc/php-fpm.d/
COPY conf/php.ini /usr/local/etc/php/
COPY conf/php-fpm.conf /usr/local/etc/

ENV WEB_ROOT /var/www/html

# Install composer
RUN curl -sS https://getcomposer.org/installer | \
  php -- \
    --install-dir=/usr/local/bin \
    --filename=composer \
    --version=1.1.2

# Install Magento 2
ENV COMPOSER_HOME /root/.composer
RUN rm -rf ${WEB_ROOT} && mkdir ${WEB_ROOT}
COPY conf/composer.json ${WEB_ROOT}/composer.json
RUN mkdir -p ${WEB_ROOT}/.composer
COPY conf/auth.json /root/.composer/auth.json
COPY conf/auth.json ${WEB_ROOT}/auth.json
RUN mkdir -p /var/run/mysqld && chmod -R 777 /var/run/mysqld
RUN cd ${WEB_ROOT} && composer install
RUN bash -c "mysqld &" \
  && sleep 1 \
  && mysql -ppw -u root -h localhost -e "CREATE DATABASE magento" \
  && cd ${WEB_ROOT} \
  && php -d memory_limit=-1 \
    bin/magento setup:install \
    --admin-firstname=Admin \
    --admin-lastname=Admin \
    --admin-email=sergey@smartsites.com \
    --admin-user=admin \
    --admin-password=admin@m2 \
    --base-url=http://magentotemplate.local/ \
    --backend-frontname=admin \
    --db-host=localhost \
    --db-name=magento \
    --db-user=root \
    --db-password=pw \
  && killall mysqld
RUN php bin/magento deploy:mode:set developer
RUN bash -c "rm ${WEB_ROOT}/composer.{json,lock}" \
  && rm -rf /root/.composer
RUN chmod -R a+wX pub/ var/ app/etc/
RUN rm ${WEB_ROOT}/app/etc/config.php

# Install codeception
RUN curl -sS http://codeception.com/codecept.phar > /bin/codecept
RUN chmod +x /bin/codecept

# Configure Apache 2
RUN a2enmod proxy_fcgi rewrite headers
COPY conf/000-default.conf /etc/apache2/sites-available/000-default.conf

WORKDIR ${WEB_ROOT}

# Configure supervisord
COPY conf/supervisord.conf /etc/supervisor/conf.d/php-mysql.conf
# Run `composer install` when this container starts and volumes have mounted:
# http://stackoverflow.com/questions/37771963/automatically-run-command-inside-docker-container-after-starting-up-volume-mou
CMD supervisord -c /etc/supervisor/supervisord.conf
