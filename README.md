Ansible Nginx
=========

An Ansible role that installs Nginx.

Requirements
------------

Only CentOS and Ubuntu are supported. Running this role against other platforms will trigger an early termination and pop an "OS not supported" error message.

Role Variables
--------------

Available role variables and their default values are listed as below, you can override them as you wish.

To change the Nginx user and group:

```yaml
nginx_user:  nginx
nginx_group: nginx
```

To change the log directory:

```yaml
nginx_log_dir:        /var/log/nginx
nginx_log_dir_user:   nginx
nginx_log_dir_group:  nginx
nginx_log_dir_mode:   0644
```

To change the configuration directory:

```yaml
nginx_conf_dir:       /etc/nginx
nginx_conf_dir_user:  root
nginx_conf_dir_group: root
nginx_conf_dir_mode:  0644
```

To change the configuration values in the nginx.conf file:

```yaml
# parameters in the main context
nginx_conf_main_user:                 nginx
nginx_conf_main_worker_processes:     auto
nginx_conf_main_error_log:            /var/log/error.log
nginx_conf_main_pidfile:              /var/run/nginx.pid
nginx_conf_main_worker_rlimit_nofile: 1024

# parameters in the events context
nginx_conf_events_worker_connections: 1024
nginx_conf_events_use:                epoll

# parameters in the http context
nginx_conf_http_access_log:         /var/log/access.log main
nginx_conf_http_sendfile:           on
nginx_conf_http_tcp_nopush:         on
nginx_conf_http_tcp_nodelay:        on
nginx_conf_http_keepalive_timeout:  65
nginx_conf_http_keepalive_requests: 100
nginx_conf_http_log_format: |
    main '$remote_addr - $remote_user [$time_local]  $status '
         '"$request" $body_bytes_sent "$http_referer" '
         '"$http_user_agent" "$http_x_forwarded_for"'

# parameters in the stream context
# none.
```

What if you want to add additional parameters not covered above? Just add the parameters into the `nginx_conf_<context>_extra_params` array. For example, this directive will add some more parameters into the http context:

```yaml
- hosts: all
  role: ansible-nginx
  nginx_conf_http_extra_params:
    - gzip            on
    - gzip_comp_level 2
    - gzip_min_length 1000;
```

## Deploy Servers

To deploy an HTTP server, put your server block in the `nginx_conf_http_servers` array.

```yaml
- hosts: all
  role: ansible-nginx
  nginx_conf_http_servers:
    - |
      server {
        listen      8080;
        server_name www.example.com;
        root        /var/www/html;

        location / {
          try_files $uri $uri/ /index.html;
        }
      }
```

To deploy a stream server, put your server block in the `nginx_conf_stream_servers` array.

```yaml
- hosts: all
  role: ansible-nginx
  nginx_conf_stream_servers:
    - |
      server {
        listen [::1]:12345;
        proxy_pass unix:/tmp/stream.socket;
      }
```

## Use Cases

### Add or Override Nginx Configurations

In the following example, we override the value of `worker_rlimit_nofile` to 2048, and add gzip configurations.

```yaml
- hosts: all
  role: ansible-nginx
  nginx_conf_main_worker_rlimit_nofile: 2048
  nginx_conf_http_extra_params:
    - gzip            on
    - gzip_comp_level 2
    - gzip_min_length 1000;
```

### Configure Nginx with SSL as a Reverse Proxy

Before starting, you need to upload your certificate and key file to the server, and place them at the right path.

Then start up the internal service, say, listening on http://localhost:8080.

```yaml
- hosts: all
  role: ansible-nginx
  nginx_conf_http_servers:
    - |
      server {
        listen      443;
        server_name srv.example.com;
        
        ssl_certificate           /etc/nginx/cert.crt;
        ssl_certificate_key       /etc/nginx/cert.key;
        
        ssl                on;
        ssl_session_cache  builtin:1000  shared:SSL:10m;
        ssl_protocols      TLSv1 TLSv1.1 TLSv1.2;
        ssl_ciphers        HIGH:!aNULL:!eNULL:!EXPORT:!CAMELLIA:!DES:!MD5:!PSK:!RC4;
        ssl_prefer_server_ciphers on;

        location / {
          proxy_set_header        Host $host;
          proxy_set_header        X-Real-IP $remote_addr;
          proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
          proxy_set_header        X-Forwarded-Proto $scheme;
          
          proxy_pass          http://localhost:8080;
          proxy_read_timeout  90;
          proxy_redirect      http://localhost:8080 https://srv.example.com;
        }
      }
```

Dependencies
------------

None.

License
-------

MIT.
