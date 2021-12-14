# Установка и развертывание Janus сервера
```sh
root@test-janus:~# apt-get update -y ; apt-get upgrade -y ; apt-get install apt-utils -y
root@test-janus:~# apt-get install -y libmicrohttpd-dev libjansson-dev libssl-dev libsofia-sip-ua-dev libglib2.0-dev libopus-dev libogg-dev libusrsctp1 libusrsctp-dev libcurl4-openssl-dev liblua5.3-dev libconfig-dev pkg-config gengetopt libtool automake autoconf cmake gtk-doc-tools libini-config-dev libcollection-dev autotools-dev make git doxygen graphviz ffmpeg python3-pip sudo
root@test-janus:~# mkdir janus-build
root@test-janus:~# cd janus-build
root@test-janus:~/janus-build# git clone https://libwebsockets.org/repo/libwebsockets
root@test-janus:~/janus-build# cd libwebsockets
root@test-janus:~/janus-build/libwebsockets# git checkout v3.2.0
root@test-janus:~/janus-build/libwebsockets# mkdir build
root@test-janus:~/janus-build/libwebsockets# cd build
root@test-janus:~/janus-build/libwebsockets/build# cmake -DLWS_MAX_SMP=1 -DCMAKE_INSTALL_PREFIX:PATH=/usr -DCMAKE_C_FLAGS="-fpic" ..
root@test-janus:~/janus-build/libwebsockets/build# make && sudo make install
root@test-janus:~/janus-build/libwebsockets/build# cd ../..
root@test-janus:~/janus-build# pip3 install meson ninja
root@test-janus:~/janus-build# git clone https://gitlab.freedesktop.org/libnice/libnice
root@test-janus:~/janus-build# cd libnice
root@test-janus:~/janus-build/libnice# meson --prefix=/usr build && ninja -C build && sudo ninja -C build install
root@test-janus:~/janus-build/libnice# cd ..
root@test-janus:~/janus-build# wget https://github.com/cisco/libsrtp/archive/v2.2.0.tar.gz
root@test-janus:~/janus-build# tar xfv v2.2.0.tar.gz
root@test-janus:~/janus-build# cd libsrtp-2.2.0
root@test-janus:~/janus-build/libsrtp-2.2.0# ./configure --prefix=/usr --enable-openssl
root@test-janus:~/janus-build/libsrtp-2.2.0# make shared_library && sudo make install
root@test-janus:~/janus-build/libsrtp-2.2.0# cd ..
root@test-janus:~/janus-build# git clone https://github.com/meetecho/janus-gateway.git
root@test-janus:~/janus-build# cd janus-gateway
root@test-janus:~/janus-build/janus-gateway# sh autogen.sh
root@test-janus:~/janus-build/janus-gateway# ./configure --prefix=/opt/janus
root@test-janus:~/janus-build/janus-gateway# make
root@test-janus:~/janus-build/janus-gateway# make install 
root@test-janus:~/janus-build/janus-gateway# make config
```
***
## ***Внимание***
Библиотека libsrtp обязательно должна быть ниже 2.3 иначе стрим не будет ловится Janus сервером
***
Janus висит на портах :
8088    http
8089    https  
8188    ws

Чтобы Janus сервер мог ловить udp стрим необходимо раскоментировать конфиг h264-sample который находится в файле :
```sh
/opt/janus/etc/janus/janus.plugin.streaming.jcfg
```
По умолчанию Janus сервер ожидает udp стрим на порту 8004, это можно изменить поправив конфиг упомянутый выше.

# Развертывание сайта для Janus сервера
По сути можно отредачить конфиги nginx'а и заставить сайтик Janus'а крутится на нем, а не поднимать отдельный сервер, но я сделал вот так :
```
root@test:~# cd /opt/janus/bin/html
root@test:/opt/janus/bin/html# gedit streamingtest.js
```

Меняем в этом файлике строки
```js
if(window.location.protocol === 'http:')
	server = "http://" + window.location.hostname + ":8088/janus";
else
	server = "https://" + window.location.hostname + ":8089/janus";
```

на вот такие

```js
if(window.location.protocol === 'http:')
	server = "http://" + "ip на котором хостится Janus" + ":8088/janus";
else
	server = "https://" + "ip на котором хостится Janus" + ":8089/janus";
```

Запускаем Janus
```sh
test@test-PC:~$ /opt/janus/bin/janus
```
Запускаем сайтик
```sh
test@test-PC:~$ cd /opt/janus/bin/html
test@test-PC:~$ python3 -m http.server <порт> --bind <ip>
```
Теперь в браузере переходим по ip:port и радуемся

Запускаем стрим
```sh
test@test-PC:~$ gst-launch-1.0 nvarguscamerasrc sensor-id=0 ! 'video/x-raw(memory:NVMM),width=1920, height=1080, framerate=60/1, format=NV12' ! nvvidconv ! capsfilter caps='video/x-raw,format=I420'  ! omxh264enc profile=8 control-rate=3 bitrate = 10000000 iframeinterval=5 preset-level=1 ! 'video/x-h264,stream-format=(string)byte-stream' ! h264parse ! rtph264pay config-interval=1 pt=96 ! multiudpsink clients="<ip на котором хостится Janus сервер>:8004", sync=false async=false
```

Теперь на сайтике во вкладке Demos/Stream мы можем включить стрим и посмотреть его
***
## ***Внимание***
Если стрим не включается, а в Janus сервер ругается вот так
```sh
[ERR] [dtls.c:janus_dtls_srtp_incoming_msg:717] [2148577725907611] Handshake error: error:1416F086:SSL routines:tls_process_server_certificate:certificate verify failed
```
То ваш браузер скорее всего не той версии, либо ещё что-то.
***

Для создания собсвенной странички со стримом необходимо в <название>.html подключить файлы :
str.js
janus.js
adapter.min.js
jquery.min.js

создать элемент 

```html
<video id="stream">
```

функции отвечающие за воспроизведение стрима
```js
Janus.attachMediaStream($('#stream').get(0), stream);
$("#stream").get(0).play();
```

janus.js можно найти в папке /opt/janus/bin/html
str.js на моем компе (Сергей) в папке /media/grey_man/dataL/grey_man/janus-gateway-master/html

либо на репе
https://github.com/grey1man/work_repa

Там же на репе лежат пайплайны для gstreamer'a

















