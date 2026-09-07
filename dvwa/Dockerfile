FROM debian:bookworm-slim

RUN apt-get update && apt-get install -y \
    apache2 php php-mysql php-gd libapache2-mod-php \
    mariadb-server git curl \
    && rm -rf /var/lib/apt/lists/*

RUN rm -rf /var/www/html/* /var/www/html/.[!.]* \
    && git clone https://github.com/digininja/DVWA.git /var/www/html \
    && cp /var/www/html/config/config.inc.php.dist /var/www/html/config/config.inc.php \
    && chown -R www-data:www-data /var/www/html \
    && chmod -R 755 /var/www/html/hackable/uploads

COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

EXPOSE 80
ENTRYPOINT ["/entrypoint.sh"]
