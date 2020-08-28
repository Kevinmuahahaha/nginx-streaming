# nginx-streaming
A walk through of how to stream videos/audios with nginx (on linux).

Serving HLS or DASH stream.

# The Concept

|rtmp provider	|	cuts stream into chunks	|	web server|
|----|----|----|
|ffmpeg -->   |  nginx rtmp module --> | nginx https|

# Setting up ffmpeg

1. install it
2. Issue the command below: 
```
  ffmpeg -re -i /path/to/video.mp4 -vcodec copy  -loop -1 -c:a aac -b:a 160k -ar 44100 -strict -2 -f flv rtmp://127.0.0.1/live/bbb
```
Note that ffmpeg streams to port 1935 by default.

# Getting a cert ( for https )

1. install socat

2. install and configure acme
```
curl https://get.acme.sh | sh
( installed to ~/.acme.sh/acme.sh by default )
```
3. configure acme
```
# issue
acme.sh --issue -d $domain --keylength ec-256
# and then install
acme.sh --installcert -d $domain --fullchainpath /path/to/cert/stream.crt --keypath /path/to/key/stream.key --ecc
```


# Building nginx with RTMP
```
Ubuntu:
	$ sudo apt update
	$ sudo apt install build-essential git
	$ sudo apt install libpcre3-dev libssl-dev zlib1g-dev


For CentOS, Oracle Linux, and RHEL:
	$ sudo yum update
	$ sudo yum groupinstall "Development Tools"
	$ sudo yum install git
	$ sudo yum groupinstall pcre-devel zlib-devel openssl-devel

Nginx:
	$ cd /path/to/build/dir
	$ git clone https://github.com/arut/nginx-rtmp-module.git
	$ git clone https://github.com/nginx/nginx.git
	$ cd nginx
	$ ./auto/configure --add-module=../nginx-rtmp-module --with-http_ssl_module
	$ make
	$ sudo make install

	# installed /usr/local/nginx/sbin/nginx
	# not using /etc/nginx.conf by default, check with nginx -t

more nginx usage:
	nginx -s reload/stop ...
	nginx -V 
```

# Configure Nginx
add rtmp to config file
```
rtmp { 
	server { 
		listen 1935; 
		chunk_size 4096;
		application live { 
			live on; 
			interleave on;

			hls on; 
			hls_path /path/to/hls; 
			hls_fragment 3s; 
			hls_playlist_length 60s;

			dash on;
			dash_path /path/to/dash;
			dash_fragment 15s;
		} 
	} 
} 
```
and within http module
```
server { 
		listen 443 ssl default_server;
		listen [::]:443 ssl default_server;
		ssl on;
		ssl_certificate       /path/to/cert/stream.crt;
		ssl_certificate_key   /path/to/key/stream.key;
		ssl_protocols         TLSv1 TLSv1.1 TLSv1.2;
		ssl_ciphers           HIGH:!aNULL:!MD5;
		server_name           <server name>;
	}

	types {
		application/vnd.apple.mpegurl m3u8;
		video/mp2t ts;
		text/html html;
		application/dash+xml mpd;
	} 
```

# Serving the stream through web (optional)
```
Now that you can access your stream through https://<server>/hlspath/bbb.m3u8
Grab a stream player (e.g. hls.js) and give it a go.
```
