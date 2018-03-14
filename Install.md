# 1. install redis-server

    sudo apt install redis-server # slanger will use redis-server as default

# 2. install tengine

## tengine download

    cd ~; wget https://github.com/alibaba/tengine/archive/tengine-2.2.2.tar.gz
    tar -xvf tengine-2.2.2.tar.gz

## tengine install

    sudo adduser deploy
    sudo usermod -a -G sudo deploy
    su -l deploy
    sudo apt-get install libpcre3 libpcre3-dev openssl libssl-dev gcc
    cd tengine-2.2.2
    ./configure --prefix=/opt/tengine
    make
    sudo make install
## tengine config

    # sudo vim /opt/tengine/conf/nginx.conf

```
user deploy;
worker_processes auto;
worker_rlimit_nofile 65535;

events {
		worker_connections 65535;
		multi_accept on;
		use epoll;
}

http {
    upstream slanger_ws {
      server 127.0.0.1:8181;
      server 127.0.0.1:8182;

      check interval=3000 rise=2 fall=1 timeout=1000 type=tcp;
    }

    upstream slanger_api {
      server 127.0.0.1:7071;
      server 127.0.0.1:7072;

      check interval=3000 rise=2 fall=1 timeout=1000 type=http;
      check_http_send "HEAD /__sinatra__/404.png HTTP/1.0\r\n\r\n";
      check_http_expect_alive http_2xx http_3xx;
    }

    server {
      listen 8180 default;
      server_name _;

      location / {
    	proxy_pass http://slanger_ws;
    	proxy_http_version 1.1;
    	proxy_set_header Upgrade $http_upgrade;
    	proxy_set_header Connection "upgrade";
    	proxy_set_header Host $host;
      }
    }

    server {
      listen 7070 default;
      server_name _;

      location / {
    	proxy_pass http://slanger_api;
    	proxy_set_header Host $host;
      }
    }
}
```
    
## tengine tcp optimize

    # sudo vim /etc/sysctl.conf
```
fs.inotify.max_user_watches = 524288
fs.file-max = 51200
net.core.rmem_max = 67108864
net.core.wmem_max = 67108864
net.core.netdev_max_backlog = 250000
net.core.somaxconn = 4096
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_fin_timeout = 30
net.ipv4.tcp_keepalive_time = 1200
net.ipv4.ip_local_port_range = 10000 65000
net.ipv4.tcp_max_syn_backlog = 8192
net.ipv4.tcp_max_tw_buckets = 5000
net.ipv4.tcp_fastopen = 3
net.ipv4.tcp_rmem = 4096 87380 67108864
net.ipv4.tcp_wmem = 4096 65536 67108864
net.ipv4.tcp_mtu_probing = 1
net.ipv4.tcp_congestion_control = hybla

net.ipv4.tcp_max_syn_backlog = 65536
net.core.netdev_max_backlog =  32768
net.core.somaxconn = 32768
net.core.wmem_default = 8388608
net.core.rmem_default = 8388608
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_timestamps = 0
net.ipv4.tcp_synack_retries = 2
```
    # sudo sysctl -p
    
## tengine start & stop

    sudo /opt/tengine/sbin/nginx
    sudo /opt/tengine/sbin/nginx -s stop
	
## install rvm && ruby 2.2.7
```
   gpg --keyserver hkp://keys.gnupg.net --recv-keys 409B6B1796C275462A1703113804BB82D39DC0E3
    curl -sSL https://get.rvm.io | bash -s stable
    rvm install 2.2 # user must have sudo 
```

# 3. install slanger

##  gem install slanger

## config slanger
 
    # create yml as slanger config file

	# slanger_1.yml
```
app_key: 47e6f94a515e8dd9dee9bf061c7afaa613e86365
secret: d523112baf07de6ddc22df0d3a2aee915ce2aff2
api_host: 127.0.0.1
api_port: 7071
websocket_host: 127.0.0.1
websocket_port: 8181
redis_address: redis://192.168.1.105:6379/slanger_1
activity_timeout: 20
#tls_options:
#  cert_chain_file: 'server.crt'
#  private_key_file: 'server.key'
```
	# slanger_2.yml
```
app_key: 47e6f94a515e8dd9dee9bf061c7afaa613e86365
secret: d523112baf07de6ddc22df0d3a2aee915ce2aff2
api_host: 127.0.0.1
api_port: 7072
websocket_host: 127.0.0.1
websocket_port: 8182
redis_address: redis://192.168.1.105:6379/slanger_2
activity_timeout: 20
#tls_options:
#  cert_chain_file: 'server.crt'
#  private_key_file: 'server.key'
```
## add environment to current user 
  
    # vim ~/.profile
    source $HOME/.rvm/gems/ruby-2.2.7@slanger/environment # current gem environment
    export PATH=$PATH:$HOME/slanger/slanger-0.6.0/bin # slanger exec folder
    
## run background

    nohup slanger -C slanger_1.yml &> slanger_1.log&
    nohup slanger -C slanger_2.yml &> slanger_2.log&
	
# 4. config info(for client)

	PUSHER_APP: '470875'
	PUSHER_KEY: '47e6f94a515e8dd9dee9bf061c7afaa613e86365'
	PUSHER_SECRET: 'd523112baf07de6ddc22df0d3a2aee915ce2aff2'
	PUSHER_CLUSTER: 'ap1'

	PUSHER_HOST: SERVER_IP
	PUSHER_PORT: '7070'
	PUSHER_WS_PORT: '8180'
	PUSHER_WSS_PORT: '8180'
	PUSHER_ENCRYPTED: 'false'
   
