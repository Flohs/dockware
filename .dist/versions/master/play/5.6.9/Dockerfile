# Official Dockware Image
# Tag: 5.6.9
# Copyright 2020 dasistweb GmbH
#
FROM ubuntu:18.04
LABEL title="Dockware play:5.6.9" \
      version="1.4.3" \
      maintainer="dasistweb GmbH"

# remember build-date
RUN date >/build-date.txt && \
    mkdir -p /var/www && \
    mkdir -p /var/www/scripts

# add our changelog to the containers
ADD ./assets/CHANGELOG.md /var/www/CHANGELOG.md

## ***********************************************************************
##  IMAGE VARIABLES
## ***********************************************************************
ENV TZ Europe/Berlin
RUN echo "export TZ=${TZ}" >> /etc/profile

ENV PHP_VERSION 7.2
RUN echo "export PHP_VERSION=${PHP_VERSION}" >> /etc/profile

ENV APACHE_DOCROOT /var/www/html
RUN echo "export APACHE_DOCROOT=${APACHE_DOCROOT}" >> /etc/profile

ENV COMPOSER_VERSION not-set
RUN echo "export COMPOSER_VERSION=${COMPOSER_VERSION}" >> /etc/profile

## ***********************************************************************
##  BASE REQUIREMENTS
## ***********************************************************************
RUN apt-get update \
        && apt-get install -y gosu \
    && apt-get install -y sudo \
    && apt-get install -y wget \
    && apt-get install -y curl \
    && apt-get install -y unzip \
    && apt-get install -y bzip2 \
    && apt-get install -y ssmtp \
    && apt-get install -y lsof \
    && apt-get install -y openssh-server \
    && apt-get install -y cron \
    && apt-get install -y vim \
    && apt-get install -y nano \
    && apt-get install -y jq \
    && mkdir /var/run/sshd \
    # TIMEZONE SETTINGS
    # otherwise we would have an interactive input dialog
    && ln -fs /usr/share/zoneinfo/Europe/Berlin /etc/localtime \
        && apt-get install -y tzdata \
    && dpkg-reconfigure --frontend noninteractive tzdata  \
     \
        && apt-get install -y xdg-utils \
            && apt-get install -y libsodium-dev \
    && apt-get install -y php-dev \
    && apt-get install -y php-pear \
    && pecl install -f libsodium \
    && apt-get remove -y php-pear \
    && apt-get remove -y php-dev \
            && rm -rf /var/lib/apt/lists/* /var/cache/apt/*

## ***********************************************************************
##  USER MANAGEMENT
## ***********************************************************************
# set easy root pwd for dev purpose
RUN echo "root:root" | chpasswd \
    # set password for www-data, and also
    # avoid shell login (we have a separate user for that)
    && echo 'www-data:www-data' | chpasswd \
    && usermod -s /usr/sbin/nologin www-data \
    # this option makes sure to avoid root SSH login
    # we just replace our lines with nothing
    && sed -i 's/PermitRootLogin without-password//' /etc/ssh/sshd_config \
    && sed -i 's/PermitRootLogin prohibit-password//' /etc/ssh/sshd_config \
    # allow root and sudo group to run sudo without password
    && sed -i /etc/sudoers -re 's/^%sudo.*/%sudo ALL=(ALL:ALL) NOPASSWD: ALL/g' \
    && sed -i /etc/sudoers -re 's/^root.*/root ALL=(ALL:ALL) NOPASSWD: ALL/g' \
    # remove include directory
    && sed -i /etc/sudoers -re 's/^#includedir.*/## **Removed the include directive** ##"/g'

## ***********************************************************************
## creates a new user as www-data alias and adds it to the sudo group
## along with privileges to run sudo without password
## params:
#       - string user
#       - string pwd
#       - bool sudo
## ***********************************************************************
RUN adduser --disabled-password --uid 5577 --gecos "" --ingroup www-data dockware \
    && usermod -m -d /var/www dockware | true \
    && echo "dockware:dockware" | chpasswd \
        && usermod -a -G sudo dockware \
    # allow sudo without pwd and dont require tty (for entrypoint commands)
    && echo "Defaults:dockware !requiretty" >> /etc/sudoers \
        && sed -i 's/dockware:x:5577:33:/dockware:x:33:33:/g' /etc/passwd
RUN echo 'AllowUsers dockware' >> /etc/ssh/sshd_config

## ***********************************************************************
##  APACHE INSTALLATION
## ***********************************************************************
#this conf is needed for enconf command ...
ADD ./config/apache/http2.conf /etc/apache2/conf-available/http2.conf

RUN apt-get update \
    && apt-get install -y apache2 \
    && apt-get install -y libapache2-mod-fcgid \
    && apt-get install -y software-properties-common \
    && LC_ALL=C.UTF-8 add-apt-repository ppa:ondrej/php \
    && a2enmod headers \
    && a2enmod rewrite \
    && a2enmod expires \
    && a2enmod proxy \
    && a2enmod proxy_http \
    && a2enmod proxy_wstunnel \
    && a2enmod actions \
    && a2enmod fcgid \
    && a2enmod alias \
    && a2enmod proxy_fcgi \
    && a2enmod http2 \
    && sudo a2enconf http2 \
    && sudo a2dismod mpm_prefork > /dev/null 2>&1 \
    && sudo a2enmod mpm_event > /dev/null 2>&1 \
    && rm -rf /var/lib/apt/lists/* /var/cache/apt/*

ADD ./config/apache/apache2.conf /etc/apache2/apache2.conf
ADD ./config/apache/ports.conf /etc/apache2/ports.conf
ADD ./config/apache/sites.conf /etc/apache2/sites-enabled/000-default.conf

# create a new id_rsa for the www-data dockeruser.
# thus we have a SSH Key for connections out of the box

RUN mkdir -p /var/www/.ssh \
    && rm -rf /var/www/.ssh/id_rsa; true  \
    && rm -rf /var/www/.ssh/id_rsa.pub; true  \
    && ssh-keygen -t rsa -b 4096 -f /var/www/.ssh/id_rsa -C "Dockware Container" -P ""  \
    && chown -R www-data:www-data /var/www/.ssh \
    && chmod 0700 /var/www/.ssh

## ***********************************************************************
##  PHP INSTALLATION
## ***********************************************************************
RUN apt-get update \
    && apt-get install -y php7.4-fpm \
    && apt-get install -y php7.4-gd \
    #make sure mbstring is installed BEFORE iconv see: https://github.com/dockware/dockware/issues/36 => https://github.com/docker-library/php/issues/240#issuecomment-355489551
    && apt-get install -y php7.4-mbstring \
    && apt-get install -y php7.4-iconv \
    && apt-get install -y php7.4-intl \
    && apt-get install -y php7.4-json \
    && apt-get install -y php7.4-xml \
    && apt-get install -y php7.4-pdo \
    && apt-get install -y php7.4-mysql \
    && apt-get install -y php7.4-apcu \
    && apt-get install -y php7.4-curl \
    && apt-get install -y php7.4-cli \
    && apt-get install -y php7.4-ldap \
    && apt-get install -y php7.4-pgsql \
    && apt-get install -y php7.4-gettext \
    && apt-get install -y php7.4-zip \
    && apt-get install -y php7.4-soap \
    && apt-get install -y php7.4-bcmath \ 
    && apt-get install -y php7.4-redis \
    && apt-get install -y php7.4-imap \
    && apt-get install -y dh-php \
    && apt-get install -y php-geoip \
    && apt-get install -y php-ssh2 \
    # shopware required pcre
    && apt-get install -y libpcre3 libpcre3-dev \
    # remove pecl again
    && apt-get remove -y dh-php \
    && rm -rf /var/lib/apt/lists/* /var/cache/apt/*

COPY ./config/php/general.ini /etc/php/7.4/fpm/conf.d/01-general.ini
COPY ./config/php/general.ini /etc/php/7.4/cli/conf.d/01-general.ini
COPY ./config/php/cli.ini /etc/php/7.4/cli/conf.d/01-general-cli.ini

RUN apt-get update \
    && apt-get install -y php7.3-fpm \
    && apt-get install -y php7.3-apcu \
    && apt-get install -y php7.3-curl \
    && apt-get install -y php7.3-cli \
    && apt-get install -y php7.3-gd \
    && apt-get install -y php7.3-json \
    && apt-get install -y php7.3-ldap \
    && apt-get install -y php7.3-mysql \
    && apt-get install -y php7.3-pgsql \
    && apt-get install -y php7.3-gettext \
    && apt-get install -y php7.3-intl \
    && apt-get install -y php7.3-xml \
    && apt-get install -y php7.3-mysql \
    && apt-get install -y php7.3-mbstring \
    && apt-get install -y php7.3-zip \
    && apt-get install -y php7.3-soap \
    && apt-get install -y php7.3-memcached \
    && apt-get install -y php7.3-redis \
    && apt-get install -y php7.3-bcmath \
    && apt-get install -y php7.3-imap \
    && apt-get install -y dh-php \
    && apt-get install -y php-geoip \
    && apt-get install -y php-ssh2 \
    # remove pecl again
    && apt-get remove -y dh-php \
    && rm -rf /var/lib/apt/lists/* /var/cache/apt/*

COPY ./config/php/general.ini /etc/php/7.3/fpm/conf.d/01-general.ini
COPY ./config/php/general.ini /etc/php/7.3/cli/conf.d/01-general.ini
COPY ./config/php/cli.ini /etc/php/7.3/cli/conf.d/01-general-cli.ini

RUN apt-get update \
    && apt-get install -y php7.2-fpm \
    && apt-get install -y php7.2-apcu \
    && apt-get install -y php7.2-curl \
    && apt-get install -y php7.2-cli \
    && apt-get install -y php7.2-gd \
    && apt-get install -y php7.2-json \
    && apt-get install -y php7.2-ldap \
    && apt-get install -y php7.2-mysql \
    && apt-get install -y php7.2-pgsql \
    && apt-get install -y php7.2-gettext \
    && apt-get install -y php7.2-intl \
    && apt-get install -y php7.2-xml \
    && apt-get install -y php7.2-mysql \
    && apt-get install -y php7.2-mbstring \
    && apt-get install -y php7.2-zip \
    && apt-get install -y php7.2-soap \
    && apt-get install -y php7.2-memcached \
    && apt-get install -y php7.2-redis \
    && apt-get install -y php7.2-bcmath \
    && apt-get install -y php7.2-imap \
    && apt-get install -y dh-php \
    && apt-get install -y php-geoip \
    && apt-get install -y php-ssh2 \
    # remove pecl again
    && apt-get remove -y dh-php \
    && rm -rf /var/lib/apt/lists/* /var/cache/apt/*

COPY ./config/php/general.ini /etc/php/7.2/fpm/conf.d/01-general.ini
COPY ./config/php/general.ini /etc/php/7.2/cli/conf.d/01-general.ini
COPY ./config/php/cli.ini /etc/php/7.2/cli/conf.d/01-general-cli.ini

#make sure the installation runs also in default php version
RUN sudo update-alternatives --set php /usr/bin/php7.2 > /dev/null 2>&1 &
# make sure the installation runs using our default php version
RUN service php7.2-fpm stop > /dev/null 2>&1 && \
    service php7.2-fpm start && \
    sudo update-alternatives --set php /usr/bin/php7.2 > /dev/null 2>&1

# make sure our php user has rights on the session
RUN chown www-data:www-data -R /var/lib/php/sessions

# remove the standard apache index file
RUN mkdir -p /var/www/html \
    && rm -rf /var/www/html/* \
    && chown -R www-data:www-data /var/www/html \
    && sudo -u www-data sh -c 'mkdir -p /var/www/html/public'

# make sure the configured log folder exists and is writeable
RUN chmod -R 0777 /var/www \
    && chgrp -R www-data /var/log/apache2 \
    && mkdir -p /var/log/mysql \
    && chgrp -R www-data /var/log/mysql\
    && mkdir /var/log/php -p  \
    && touch /var/log/php/cli_errors.log  \
    && touch /var/log/php/fpm_errors.log  \
    && chown -R www-data:www-data /var/log/php  \
    && chmod 0755 /var/log/php

## ***********************************************************************
##  MOD_SSL
##  create SSL certificate
## ***********************************************************************
RUN apt-get update \
    && apt-get install -y openssl \
    && a2enmod ssl \
    && mkdir /etc/apache2/ssl \
    && openssl req -new -x509 -days 365 -sha1 -newkey rsa:2048 -nodes -keyout /etc/apache2/ssl/server.key -out /etc/apache2/ssl/server.crt -subj '/O=Company/OU=Department/CN=localhost' \
    && rm -rf /var/lib/apt/lists/* /var/cache/apt/*

RUN apt-get update \

    && apt-get install -y git \

    && rm -rf /var/lib/apt/lists/* /var/cache/apt/*

## ***********************************************************************
##  MYSQL INSTALL
## ***********************************************************************

# prepare environment variables to allow a
# quiet install of the mysql server
# this sets the root password to root without user prompts
RUN echo debconf mysql-server/root_password password root | debconf-set-selections
RUN echo debconf mysql-server/root_password_again password root | debconf-set-selections

RUN DEBIAN_FRONTEND=noninteractive \
    # install mysql server
    && apt-get update \
    && apt-get install -y -q mysql-server \
    # set requires permissions
    && usermod -d /var/lib/mysql/ mysql \
    # cleanup download folders
    && rm -rf /var/lib/apt/lists/* /var/cache/apt/*

# copy our custom configuration to the image
ADD ./config/mysql/my.cnf /etc/mysql/my.cnf

# i dont know why, but this is suddenly required
RUN chmod 0444 /etc/mysql/my.cnf

RUN service mysql start && \
    mysql --user=root --password=root -e "use mysql; update user set host='%' where host='localhost';" && \
    service mysql restart

## ***********************************************************************
##  ADMINER
## ***********************************************************************

RUN mkdir /usr/share/adminer \
    && wget "http://www.adminer.org/latest.php" -O /usr/share/adminer/latest.php \
    && ln -s /usr/share/adminer/latest.php /usr/share/adminer/adminer.php \
    && echo "Alias /adminer.php /usr/share/adminer/adminer.php" | sudo tee /etc/apache2/conf-available/adminer.conf \
    && a2enconf adminer.conf

RUN apt-get update \
    && apt install -y build-essential libsqlite3-dev ruby-dev \
    && gem install mailcatcher --no-ri --no-rdoc \
    && phpenmod mailcatcher \
    && rm -rf /var/lib/apt/lists/* /var/cache/apt/*

COPY ./config/ssmtp/ssmtp.conf /etc/ssmtp/ssmtp.conf

RUN echo "sendmail_path = /usr/bin/env $(which catchmail) -f 'local@dockware'" >> /etc/php/7.4/mods-available/mailcatcher.ini && \
echo "sendmail_path = /usr/bin/env $(which catchmail) -f 'local@dockware'" >> /etc/php/7.3/mods-available/mailcatcher.ini && \
echo "sendmail_path = /usr/bin/env $(which catchmail) -f 'local@dockware'" >> /etc/php/7.2/mods-available/mailcatcher.ini && \
    echo ""

RUN mkdir -p /var/www/pimpmylog && \
    wget -O - https://github.com/potsky/PimpMyLog/tarball/master | tar xzf - && \
    mv potsky-PimpMyLog-* /var/www/pimpmylog && \
    mv /var/www/pimpmylog/potsky-PimpMyLog-2fed8c1/* /var/www/pimpmylog && \
    rm -rf /var/www/pimpmylog/potsky-PimpMyLog-*

COPY /config/pimpmylog/config.user.d /var/www/pimpmylog/config.user.d
COPY /config/pimpmylog/config.user.json /var/www/pimpmylog/config.user.json

# apply our custom file with fixes for PHP 8
# its used from here: https://github.com/potsky/PimpMyLog/pull/149/files
COPY /config/pimpmylog/global.inc.php /var/www/pimpmylog/inc/global.inc.php

RUN chown -R www-data:www-data /var/www/pimpmylog/

COPY ./assets/scripts/makefile /var/www/makefile
COPY ./assets/scripts/bin /var/www/scripts/bin

RUN chown www-data:www-data -R /var/www/scripts
#make sure for the whole isntallation xdebug is off for performance
RUN sh /var/www/scripts/bin/xdebug_disable.sh

RUN rm -rf /var/www/html/* \
    && wget https://www.shopware.com/en/Download/redirect/version/sw5/file/install_5.6.9_be8e9cd9a8f4b7ab6f81c7922e71cbe3c16d78eb.zip -qq -O /var/www/shopware.zip \
    && sudo -u www-data unzip -q /var/www/shopware.zip -d /var/www/html \
    && rm -rf /var/www/shopware.zip

RUN service mysql start && \
    # we have to update the used PHP version (for older shopware versions <= 5.3)
    update-alternatives --set php /usr/bin/php7.2 > /dev/null 2>&1 && \
    mysql --user=root --password=root -e "CREATE DATABASE shopware CHARACTER SET utf8 COLLATE utf8_general_ci;" && \
    cd /var/www/html && sudo -u www-data php recovery/install/index.php --no-interaction --no-skip-import --db-host="localhost" --db-user="root" --db-password="root" --db-name="shopware" --shop-locale="de_DE" --shop-host="localhost" --shop-name="Dockware" --shop-email="EMAIL" --shop-currency="EUR" --admin-username="demo" --admin-password="demo" --admin-email="ADMIN-EMAIL" --admin-name="Dockware Admin" --admin-locale="de_DE" && \
    cd /var/www/html && sudo -u www-data php bin/console sw:firstrunwizard:disable && \
        cd /var/www/html && sudo -u www-data php bin/console sw:store:download SwagDemoDataEN && \
    cd /var/www/html && sudo -u www-data php bin/console sw:plugin:install --activate SwagDemoDataEN && \
        cd /var/www/html && sudo -u www-data php bin/console sw:cache:clear && \
    mysql --user=root --password=root shopware -e "UPDATE s_core_config_elements SET value='s:4:\"smtp\";' WHERE name='mailer_mailer';" && \
    mysql --user=root --password=root shopware -e "UPDATE s_core_config_elements SET value='s:4:\"1025\";' WHERE name='mailer_port';" && \
    service mysql stop

# some images dont have that folder. make sure its there
RUN mkdir -p /var/www/.composer \
    && chown www-data:www-data /var/www/.composer -R \
    # find all files and folders owned by root and chown to www-data
    && find /var/www -group root -exec chown www-data:www-data {} \; \
    && find /var/www -user root -exec chown www-data:www-data {} \;

## ***********************************************************************
##  CLEAN UP / POST BUILD
## ***********************************************************************
ADD entrypoint.sh /entrypoint.sh
RUN chmod 755 /*.sh

# make the apache folder the working directory
WORKDIR /var/www/html

# switch to non root user
USER dockware

CMD ["/entrypoint.sh"]
