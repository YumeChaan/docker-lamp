FROM phusion/baseimage:jammy-1.0.4
MAINTAINER Matthew Rayner <hello@rayner.io>
ENV REFRESHED_AT 2025-09-16

# based on dgraziotin/lamp
# MAINTAINER Daniel Graziotin <daniel@ineed.coffee>
# updated for Ubuntu 22.04 LTS/PHP 8.x Ferdinand Kasper <fkasper@modus-operandi.at>

ENV DOCKER_USER_ID 501
ENV DOCKER_USER_GID 20

ENV BOOT2DOCKER_ID 1000
ENV BOOT2DOCKER_GID 50

ENV PHPMYADMIN_VERSION=5.1.1
ENV SUPERVISOR_VERSION=4.2.2

# Pass PHP version at build: --build-arg PHP_VERSION=8.3 (or 8.2)
ARG PHP_VERSION
ENV PHP_VERSION=${PHP_VERSION}

# Tweaks to give Apache/PHP write permissions to the app
RUN usermod -u ${BOOT2DOCKER_ID} www-data && \
    usermod -G staff www-data && \
    useradd -r mysql && \
    usermod -G staff mysql && \
    groupmod -g $(($BOOT2DOCKER_GID + 10000)) $(getent group $BOOT2DOCKER_GID | cut -d: -f1) && \
    groupmod -g ${BOOT2DOCKER_GID} staff

# ---- Packages (Jammy) ----
# - Install tools needed to add the PHP PPA
# - Add ppa:ondrej/php (no apt-key on jammy)
# - Install Apache + mod_php + MySQL server + PHP ${PHP_VERSION} and common extensions
ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get install -y \
      software-properties-common \
      ca-certificates \
      lsb-release \
      gnupg \
      curl \
      wget && \
    add-apt-repository -y ppa:ondrej/php && \
    apt-get update && \
    apt-get -y upgrade && \
    apt-get -y install \
      postfix \
      python3-setuptools \
      git \
      apache2 \
      php${PHP_VERSION} \
      php${PHP_VERSION}-xdebug \
      libapache2-mod-php${PHP_VERSION} \
      mysql-server \
      php${PHP_VERSION}-mysql \
      pwgen \
      php${PHP_VERSION}-apcu \
      php${PHP_VERSION}-gd \
      php${PHP_VERSION}-xml \
      php${PHP_VERSION}-mbstring \
      zip unzip \
      php${PHP_VERSION}-zip \
      curl \
      php${PHP_VERSION}-curl && \
    apt-get -y autoremove && \
    apt-get -y clean && \
    echo "ServerName localhost" >> /etc/apache2/apache2.conf

# Apache: mod_php needs prefork MPM (not event)
RUN a2dismod mpm_event || true && \
    a2enmod mpm_prefork && \
    a2enmod php${PHP_VERSION}

# ---- Supervisor 4 ----
RUN curl -L https://pypi.io/packages/source/s/supervisor/supervisor-${SUPERVISOR_VERSION}.tar.gz | tar xvz && \
  cd supervisor-${SUPERVISOR_VERSION}/ && \
  python3 setup.py install

# ---- Image configuration and scripts ----
ADD supporting_files/start-apache2.sh /start-apache2.sh
ADD supporting_files/start-mysqld.sh /start-mysqld.sh
ADD supporting_files/run.sh /run.sh
RUN chmod 755 /*.sh
ADD supporting_files/supervisord-apache2.conf /etc/supervisor/conf.d/supervisord-apache2.conf
ADD supporting_files/supervisord-mysqld.conf /etc/supervisor/conf.d/supervisord-mysqld.conf
ADD supporting_files/supervisord.conf /etc/supervisor/supervisord.conf

# Remove pre-installed database
RUN rm -rf /var/lib/mysql

# MySQL utils
ADD supporting_files/create_mysql_users.sh /create_mysql_users.sh

# ---- phpMyAdmin ----
# Note: 5.1.1 is quite old; for PHP 8.2/8.3 you may need >= 5.2.x.
RUN wget -O /tmp/phpmyadmin.tar.gz https://files.phpmyadmin.net/phpMyAdmin/${PHPMYADMIN_VERSION}/phpMyAdmin-${PHPMYADMIN_VERSION}-all-languages.tar.gz && \
    tar xfvz /tmp/phpmyadmin.tar.gz -C /var/www && \
    ln -s /var/www/phpMyAdmin-${PHPMYADMIN_VERSION}-all-languages /var/www/phpmyadmin && \
    mv /var/www/phpmyadmin/config.sample.inc.php /var/www/phpmyadmin/config.inc.php

# ---- Composer ----
RUN php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');" && \
    php composer-setup.php && \
    php -r "unlink('composer-setup.php');" && \
    mv composer.phar /usr/local/bin/composer

# Environment variables to configure php
ENV PHP_UPLOAD_MAX_FILESIZE=10M
ENV PHP_POST_MAX_SIZE=10M
ENV MYSQL_PASS:-$(pwgen -s 12 1)

# Apache vhost & modules
ADD supporting_files/apache_default /etc/apache2/sites-available/000-default.conf
RUN a2enmod rewrite

# App folder
RUN mkdir -p /app && rm -fr /var/www/html && ln -s /app /var/www/html
ADD app/ /app

# Volumes for the app and MySQL
VOLUME ["/var/lib/mysql", "/app"]

EXPOSE 80 3306
CMD ["/run.sh"]
