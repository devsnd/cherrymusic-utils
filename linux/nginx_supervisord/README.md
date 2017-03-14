# Deploy using Nginx, Supervisord and Virtualenv

We assume the deployment is done in `/home/user/music.domain.com`

### Cherrymusic

1. Go to the deployment folder and clone the [cherrymusic](https://github.com/devsnd/cherrymusic) repo

```
cd /home/user/music.domain.com
git clone https://github.com/devsnd/cherrymusic
```

2. Create and enabled the virtualenv

```
python3 -m venv music_env
source music_env/bin/activate
```

3. Test if the cherrymusic server starts and stop it afterwords

```
python cherrymusic --setup --port 8080
(ctrl+c)
```

4. If you executed this commands under another user than the one under which you want to run cherrymusic 
(eg: you ran the commands as root but you want to run under the user `user`)

```
mkdir -p /home/user/.config/cherrymusic
cp ~/.config/cherrymusic/cherrymusic.conf /home/user/.config/cherrymusic/
```

5. Edit cherrymusic.conf from the `user`'s home and set the `basedir` with the path where your music collection is stored.
eg: /var/music

### Supervisord

1. Install [supervisord](http://supervisord.org/installing.html#installing-to-a-system-with-internet-access)

2. Create the file `/etc/supervisor/conf.d/music.conf` with this content

```
[program:music]
numprocs = 1
numprocs_start = 1
process_name = music
command=/home/user/music.domain.com/bin/python /home/user/music.domain.com/cherrymusic/cherrymusic --port=8081
user=<USER>
autostart=true
autorestart=true
```

Ajust the path `/home/user/music.domain.com` and the value of the `user` with your values.

3.  Reload supervisor service:

```service supervisor reload```

4. Check if the `music` service shows in the supervisor status:

```
supervisorctl status
```

You should see something like

```
music                            STOPPED   
```

5. Start the `music` service

```
supervisorctl start music
```

6. Check if there is any error in the logs and if you can access the service

```
tail -f /var/log/supervisor/*
(ctrl+c)
```

Try to check the connectivity:

```
telnet localhost 8081
```

Should see something like:

```
Trying ::1...
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.

( press: ctrl \ )
```

Also, if there is no firewall, should work to access from the browser: http://music.domain.com:8081/

### Nginx
1. Install nginx webserver

```apt-get install nginx```

2. Create the configuration file in `/etc/nginx/sites-enabled/music.domain.com.conf`

```
upstream music_servers {
    server 127.0.0.1:8081 fail_timeout=0;
}

server {
    listen              80;
    server_name         music.domain.com; # change this
    # root                /path/to/public_html;  # change this
    location / {
	   try_files   $uri @proxy_to_app;
    }

    location /robots.txt {
        access_log off;
        sendfile on;
    }

    location /favicon.ico {
        expires 30d;
        access_log off;
        sendfile on;
    }

    location @proxy_to_app {
        proxy_pass	   http://music_servers;
        proxy_redirect     off;
        proxy_set_header   Host $host;
        proxy_set_header   X-Real-IP $remote_addr;
        proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Host $server_name;
        proxy_set_header   X-Forwarded-Proto http;
    }

    error_log /var/log/music_error.log;
    access_log /var/log/music_access.log;
} 

```

(Adjust the domain name and paths)

3. Reload nginx web server:

```
service nginx reload
```

Test if everythings works from your browser: http://music.domain.com/

### Firewall

Do not allow direct access to the application, but only through nginx.

```
# ...
iptables -A INPUT -i lo -j ACCEPT
iptables -A OUTPUT -o lo -j ACCEPT
iptables -A INPUT -p tcp -m tcp -s localhost --dport 8081 -j ACCEPT
iptables -A INPUT -p tcp -m tcp --dport 8081 -j DROP
# ...
```

