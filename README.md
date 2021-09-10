# cerbot_odoo

apt-get install nginx

apt-get update
apt-get install software-properties-common
add-apt-repository ppa:certbot/certbot
apt-get update 
apt-get install certbot


service nginx stop
certbot certonly --standalone -d dominio.com
service nginx start

mkdir /etc/nginx/ssl
openssl dhparam -out /etc/nginx/ssl/dhp-2048.pem 2048

nano /etc/nginx/sites-available/dominio.com

#pegar el siguiente codigo y reemplazar dominio.com por el dominio nuestro

upstream odoo {
    server 127.0.0.1:8069;
}
 
server {
    listen      443 default;
    server_name dominio.com;
 
    access_log  /var/log/nginx/odoo.access.log;
    error_log   /var/log/nginx/odoo.error.log;
 
    ssl on;
    ssl_certificate     /etc/letsencrypt/live/dominio.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/dominio.com/privkey.pem;
    keepalive_timeout   60;
 
    ssl_ciphers "ECDH+AESGCM:DH+AESGCM:ECDH+AES256:DH+AES256:ECDH+AES128:DH+AES:ECDH+3DES:DH+3DES:RSA+AESGCM:RSA+AES:RSA+3DES:!aNULL:!MD5:!DSS:!AES256";
    ssl_protocols           TLSv1 TLSv1.1 TLSv1.2;
    ssl_prefer_server_ciphers on;
    ssl_dhparam /etc/nginx/ssl/dhp-2048.pem;
 
    proxy_buffers 16 64k;
    proxy_buffer_size 128k;
 
    location / {
        proxy_pass  http://odoo;
        proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_504;
        proxy_redirect off;
 
        proxy_set_header    Host            $host;
        proxy_set_header    X-Real-IP       $remote_addr;
        proxy_set_header    X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header    X-Forwarded-Proto https;
    }
 
    location ~* /web/static/ {
        proxy_cache_valid 200 60m;
        proxy_buffering on;
        expires 864000;
        proxy_pass http://odoo;
    }
}
 
server {
    listen      80;
    server_name dominio.com;
 
    add_header Strict-Transport-Security max-age=2592000;
    rewrite ^/.*$ https://$host$request_uri? permanent;
}


# despues crear los enlaces symbolicos
ln -s /etc/nginx/sites-available/dominio.com /etc/nginx/sites-enabled/dominio.com
/etc/init.d/nginx restart

#crear un crontab para renovar el certificado
crontab -e

29 2 * * 1 /etc/init.d/nginx stop
30 2 * * 1 certbot renew >> /var/log/le-renew.log
35 2 * * 1 /etc/init.d/nginx start
37 2 * * 1 /etc/init.d/nginx reload

