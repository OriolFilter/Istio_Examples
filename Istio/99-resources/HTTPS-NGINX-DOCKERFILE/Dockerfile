FROM nginx

ADD server.conf /etc/nginx/conf.d/default.conf

RUN mkdir -p /var/www/html
RUN echo "<h2>Howdy</h2>" | tee /var/www/html/index.html

RUN openssl req -x509 -sha256 -nodes -days 358000 -subj '/O=SSL EXAMPLE/CN=lb.net' -newkey rsa:2048 -keyout /cert.key -out /cert.crt