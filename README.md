# Pleroma-Nginx-Conf
Configuration of Pleroma Nginx Conf

# Pleroma官方的安装步骤(Debian和Ubuntu系列):
https://docs.pleroma.social/debian_based_en.html

# 因为我是使用宝塔面板(bt.cn)安装的，nginx配置，如果按照Pleroma官方的，会出错。
所以修改了一下。
主要是根据宝塔提供的免费的Let's Encrypt生成的不同路径的nginx配置，进行了调整。
另外，由于安装宝塔之后，已经装了Nginx，所以在安装Pleroma过程中，可能会有端口冲突，所以我把127.0.0.1改成了127.0.0.1，把4000端口改成了4001。

# 以下是修改后的(替换掉Your_Domain为实际域名）：

------------------

proxy_cache_path /tmp/pleroma-media-cache levels=1:2 keys_zone=pleroma_media_cache:10m max_size=10g
                 inactive=720m use_temp_path=off;

server {
    server_name    Your_Domain;

    listen         80;
    listen         [::]:80;

    # Uncomment this if you need to use the 'webroot' method with certbot. Make sure
    # that the directory exists and that it is accessible by the webserver. If you followed
    # the guide, you already ran 'mkdir -p /var/lib/letsencrypt' to create the folder.
    # You may need to load this file with the ssl server block commented out, run certbot
    # to get the certificate, and then uncomment it.
    #
    # location ~ /\.well-known/acme-challenge {
    #     root /opt/pleroma/;
    # }
    location / {
      return         301 https://$server_name$request_uri;
    }
}

# Enable SSL session caching for improved performance
ssl_session_cache shared:ssl_session_cache:10m;

server {
    server_name Your_Domain;

    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    ssl_session_timeout 5m;

    ssl_certificate    /www/server/panel/vhost/cert/Your_Domain/fullchain.pem;
    ssl_certificate_key    /www/server/panel/vhost/cert/Your_Domain/privkey.pem;

    # Add TLSv1.0 to support older devices
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
    # Uncomment line below if you want to support older devices (Before Android 4.4.2, IE 8, etc.)
    # ssl_ciphers "HIGH:!aNULL:!MD5 or HIGH:!aNULL:!MD5:!3DES";
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE;
    ssl_prefer_server_ciphers on;
    # In case of an old server with an OpenSSL version of 1.0.2 or below,
    # leave only prime256v1 or comment out the following line.
    ssl_ecdh_curve X25519:prime256v1:secp384r1:secp521r1;
    ssl_stapling on;
    ssl_stapling_verify on;

    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_buffers 16 8k;
    gzip_http_version 1.1;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript application/activity+json application/atom+xml;

    # the nginx default is 1m, not enough for large media uploads
    client_max_body_size 16m;

    location / {
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $http_host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        # this is explicitly IPv4 since Pleroma.Web.Endpoint binds on IPv4 only
        # and `localhost.` resolves to [::0] on some systems: see issue #930
        proxy_pass http://127.0.0.2:4001;

        client_max_body_size 16m;
    }

    location ~ ^/(media|proxy) {
        proxy_cache        pleroma_media_cache;
        proxy_http_version 1.1;
        proxy_cache_valid  200 206 301 304 1h;
        proxy_cache_lock   on;
        proxy_ignore_client_abort on;
        proxy_buffering    on;
        chunked_transfer_encoding on;
        proxy_ignore_headers Cache-Control;
        proxy_hide_header  Cache-Control;
        proxy_pass         http://127.0.0.2:4001;
    }
}
