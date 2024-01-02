# 基于MQTT的隧道服务端和客户端 mqtt tunnel for public access you private nat server 

轻量级,低功耗内网穿透隧道服务端和客户端, 带用户认证

mqtt tunnel allows you to expose services which are running on `localhost`, or on your local network, to the public internet.

This is very useful for testing webhooks, the generation of static-site compilers, and similar things.


## MQTT服务端 Mosquitto 安装配置
Eclipse Mosquitto™  An open source MQTT broker
~~~sh
#Debian/Ubuntu system:
apt-get install mosquitto
~~~

- Configure Mosquitto

Create `/etc/mosquitto/conf.d/acl.conf` with just the following contents:
~~~conf
# 访问规则配置文件
acl_file /etc/mosquitto/conf.d/acl.txt

# 禁止匿名用户访问
allow_anonymous false

# 用户密码文件
password_file /etc/mosquitto/conf.d/pwfile.security

# MQTT服务绑定IP 0.0.0.0 表示本地所有可用IP
bind_address 0.0.0.0
~~~

MQTT访问规则配置文件
/etc/mosquitto/conf.d/acl.txt 
~~~txt
# This affects access control for clients with no username.
topic read $SYS/#

# This only affects clients with username "admin".
user admin
topic foo/bar

# This affects all clients.
pattern readwrite clients/#
~~~

~~~sh
# 添加用户 admin/admin888
# 同样连续会提示连续输入两次密码 admin888 。注意第二次创建用户时不用加 -c 如果加 -c 会把第一次创建的用户覆盖。
mosquitto_passwd -c /etc/mosquitto/conf.d/pwfile.security admin

# 重启服务
systemctl restart mosquitto

# 订阅测试
mosquitto_sub -h 192.168.0.99 -u admin -P admin888 -d -t clients/admin

# 消息发布测试 另外打开一个窗口
mosquitto_pub -h 192.168.0.99 -u admin -P admin888 -d -t clients/admin -m "hello, tekin"
~~~

## MQTT隧道服务端和客户端服务启动

-username=admin -password=admin888 这个是链接MQTT服务的用户名和密码
~~~sh
# 服务端:
# -port=80 这个端口是你的服务访问的端口, 即 admin.example.com 的访问端口
./tunnel-server-linux-amd64 serve -port=80 -host=0.0.0.0 -username=admin -password=admin888

# 客户端 
# -expose="localhost:8000" 这个是你本地要暴露到公网的服务地址和端口, 
# -name=admin 这个是你的服务名 最终访问域名为 admin.example.com
./tunnel-client-linux-amd64 client -expose="localhost:8000" -name=admin -tunnel=example.com  -username=admin -password=admin888
~~~

修改你的域名DNS解析,增加泛域名解析到  
~~~
*.example.com
~~~

防火墙开放端口  1883

访问: http://admin.example.com/ 即可访问你在内网服务的 http://localhost:8000/



## mosquitto_sub命令参考

~~~sh
mosquitto_sub --help
mosquitto_sub is a simple mqtt client that will subscribe to a set of topics and print all messages it receives.
mosquitto_sub version 2.0.18 running on libmosquitto 2.0.18.

Usage: mosquitto_sub {[-h host] [--unix path] [-p port] [-u username] [-P password] -t topic | -L URL [-t topic]}
                     [-c] [-k keepalive] [-q qos] [-x session-expiry-interval]
                     [-C msg_count] [-E] [-R] [--retained-only] [--remove-retained] [-T filter_out] [-U topic ...]
                     [-F format]
                     [-W timeout_secs]
                     [-A bind_address] [--nodelay]
                     [-i id] [-I id_prefix]
                     [-d] [-N] [--quiet] [-v]
                     [--will-topic [--will-payload payload] [--will-qos qos] [--will-retain]]
                     [{--cafile file | --capath dir} [--cert file] [--key file]
                       [--ciphers ciphers] [--insecure]
                       [--tls-alpn protocol]
                       [--tls-engine engine] [--keyform keyform] [--tls-engine-kpass-sha1]]
                       [--tls-use-os-certs]
                     [--psk hex-key --psk-identity identity [--ciphers ciphers]]
                     [--proxy socks-url]
                     [-D command identifier value]
       mosquitto_sub --help

 -A : bind the outgoing socket to this host/ip address. Use to control which interface
      the client communicates over.
 -c : disable clean session/enable persistent client mode
      When this argument is used, the broker will be instructed not to clean existing sessions
      for the same client id when the client connects, and sessions will never expire when the
      client disconnects. MQTT v5 clients can change their session expiry interval with the -x
      argument.
 -C : disconnect and exit after receiving the 'msg_count' messages.
 -d : enable debug messages.
 -D : Define MQTT v5 properties. See the documentation for more details.
 -E : Exit once all subscriptions have been acknowledged by the broker.
 -F : output format.
 -h : mqtt host to connect to. Defaults to localhost.
 -i : id to use for this client. Defaults to mosquitto_sub_ appended with the process id.
 -I : define the client id as id_prefix appended with the process id. Useful for when the
      broker is using the clientid_prefixes option.
 -k : keep alive in seconds for this client. Defaults to 60.
 -L : specify user, password, hostname, port and topic as a URL in the form:
      mqtt(s)://[username[:password]@]host[:port]/topic
 -N : do not add an end of line character when printing the payload.
 -p : network port to connect to. Defaults to 1883 for plain MQTT and 8883 for MQTT over TLS.
 -P : provide a password
 -q : quality of service level to use for the subscription. Defaults to 0.
 -R : do not print stale messages (those with retain set).
 -t : mqtt topic to subscribe to. May be repeated multiple times.
 -T : topic string to filter out of results. May be repeated.
 -u : provide a username
 -U : unsubscribe from a topic. May be repeated.
 -v : print published messages verbosely.
 -V : specify the version of the MQTT protocol to use when connecting.
      Can be mqttv5, mqttv311 or mqttv31. Defaults to mqttv311.
 -W : Specifies a timeout in seconds how long to process incoming MQTT messages.
 -x : Set the session-expiry-interval property on the CONNECT packet. Applies to MQTT v5
      clients only. Set to 0-4294967294 to specify the session will expire in that many
      seconds after the client disconnects, or use -1, 4294967295, or ∞ for a session
      that does not expire. Defaults to -1 if -c is also given, or 0 if -c not given.
 --help : display this message.
 --nodelay : disable Nagle's algorithm, to reduce socket sending latency at the possible
             expense of more packets being sent.
 --pretty : print formatted output rather than minimised output when using the
            JSON output format option.
 --quiet : don't print error messages.
 --random-filter : only print a percentage of received messages. Set to 100 to have all
                   messages printed, 50.0 to have half of the messages received on average
                   printed, and so on.
 --retained-only : only handle messages with the retained flag set, and exit when the
                   first non-retained message is received.
 --remove-retained : send a message to the server to clear any received retained messages
                     Use -T to filter out messages you do not want to be cleared.
 --unix : connect to a broker through a unix domain socket instead of a TCP socket,
          e.g. /tmp/mosquitto.sock
 --will-payload : payload for the client Will, which is sent by the broker in case of
                  unexpected disconnection. If not given and will-topic is set, a zero
                  length message will be sent.
 --will-qos : QoS level for the client Will.
 --will-retain : if given, make the client Will retained.
 --will-topic : the topic on which to publish the client Will.
 --cafile : path to a file containing trusted CA certificates to enable encrypted
            certificate based communication.
 --capath : path to a directory containing trusted CA certificates to enable encrypted
            communication.
 --cert : client certificate for authentication, if required by server.
 --key : client private key for authentication, if required by server.
 --keyform : keyfile type, can be either "pem" or "engine".
 --ciphers : openssl compatible list of TLS ciphers to support.
 --tls-version : TLS protocol version, can be one of tlsv1.3 tlsv1.2 or tlsv1.1.
                 Defaults to tlsv1.2 if available.
 --insecure : do not check that the server certificate hostname matches the remote
              hostname. Using this option means that you cannot be sure that the
              remote host is the server you wish to connect to and so is insecure.
              Do not use this option in a production environment.
 --tls-engine : If set, enables the use of a SSL engine device.
 --tls-engine-kpass-sha1 : SHA1 of the key password to be used with the selected SSL engine.
 --tls-use-os-certs : Load and trust OS provided CA certificates.
 --psk : pre-shared-key in hexadecimal (no leading 0x) to enable TLS-PSK mode.
 --psk-identity : client identity string for TLS-PSK mode.
 --proxy : SOCKS5 proxy URL of the form:
           socks5h://[username[:password]@]hostname[:port]
           Only "none" and "username" authentication is supported.

See https://mosquitto.org/ for more information.
~~~

## mosquitto_pub 命令参考
~~~sh
mosquitto_pub --help
mosquitto_pub is a simple mqtt client that will publish a message on a single topic and exit.
mosquitto_pub version 2.0.18 running on libmosquitto 2.0.18.

Usage: mosquitto_pub {[-h host] [--unix path] [-p port] [-u username] [-P password] -t topic | -L URL}
                     {-f file | -l | -n | -m message}
                     [-c] [-k keepalive] [-q qos] [-r] [--repeat N] [--repeat-delay time] [-x session-expiry]
                     [-A bind_address] [--nodelay]
                     [-i id] [-I id_prefix]
                     [-d] [--quiet]
                     [-M max_inflight]
                     [-u username [-P password]]
                     [--will-topic [--will-payload payload] [--will-qos qos] [--will-retain]]
                     [{--cafile file | --capath dir} [--cert file] [--key file]
                       [--ciphers ciphers] [--insecure]
                       [--tls-alpn protocol]
                       [--tls-engine engine] [--keyform keyform] [--tls-engine-kpass-sha1]]
                       [--tls-use-os-certs]
                     [--psk hex-key --psk-identity identity [--ciphers ciphers]]
                     [--proxy socks-url]
                     [--property command identifier value]
                     [-D command identifier value]
       mosquitto_pub --help

 -A : bind the outgoing socket to this host/ip address. Use to control which interface
      the client communicates over.
 -d : enable debug messages.
 -c : disable clean session/enable persistent client mode
      When this argument is used, the broker will be instructed not to clean existing sessions
      for the same client id when the client connects, and sessions will never expire when the
      client disconnects. MQTT v5 clients can change their session expiry interval with the -x
      argument.
 -D : Define MQTT v5 properties. See the documentation for more details.
 -f : send the contents of a file as the message.
 -h : mqtt host to connect to. Defaults to localhost.
 -i : id to use for this client. Defaults to mosquitto_pub_ appended with the process id.
 -I : define the client id as id_prefix appended with the process id. Useful for when the
      broker is using the clientid_prefixes option.
 -k : keep alive in seconds for this client. Defaults to 60.
 -L : specify user, password, hostname, port and topic as a URL in the form:
      mqtt(s)://[username[:password]@]host[:port]/topic
 -l : read messages from stdin, sending a separate message for each line.
 -m : message payload to send.
 -M : the maximum inflight messages for QoS 1/2..
 -n : send a null (zero length) message.
 -p : network port to connect to. Defaults to 1883 for plain MQTT and 8883 for MQTT over TLS.
 -P : provide a password
 -q : quality of service level to use for all messages. Defaults to 0.
 -r : message should be retained.
 -s : read message from stdin, sending the entire input as a message.
 -t : mqtt topic to publish to.
 -u : provide a username
 -V : specify the version of the MQTT protocol to use when connecting.
      Can be mqttv5, mqttv311 or mqttv31. Defaults to mqttv311.
 -x : Set the session-expiry-interval property on the CONNECT packet. Applies to MQTT v5
      clients only. Set to 0-4294967294 to specify the session will expire in that many
      seconds after the client disconnects, or use -1, 4294967295, or ∞ for a session
      that does not expire. Defaults to -1 if -c is also given, or 0 if -c not given.
 --help : display this message.
 --nodelay : disable Nagle's algorithm, to reduce socket sending latency at the possible
             expense of more packets being sent.
 --quiet : don't print error messages.
 --repeat : if publish mode is -f, -m, or -s, then repeat the publish N times.
 --repeat-delay : if using --repeat, wait time seconds between publishes. Defaults to 0.
 --unix : connect to a broker through a unix domain socket instead of a TCP socket,
          e.g. /tmp/mosquitto.sock
 --will-payload : payload for the client Will, which is sent by the broker in case of
                  unexpected disconnection. If not given and will-topic is set, a zero
                  length message will be sent.
 --will-qos : QoS level for the client Will.
 --will-retain : if given, make the client Will retained.
 --will-topic : the topic on which to publish the client Will.
 --cafile : path to a file containing trusted CA certificates to enable encrypted
            communication.
 --capath : path to a directory containing trusted CA certificates to enable encrypted
            communication.
 --cert : client certificate for authentication, if required by server.
 --key : client private key for authentication, if required by server.
 --keyform : keyfile type, can be either "pem" or "engine".
 --ciphers : openssl compatible list of TLS ciphers to support.
 --tls-version : TLS protocol version, can be one of tlsv1.3 tlsv1.2 or tlsv1.1.
                 Defaults to tlsv1.2 if available.
 --insecure : do not check that the server certificate hostname matches the remote
              hostname. Using this option means that you cannot be sure that the
              remote host is the server you wish to connect to and so is insecure.
              Do not use this option in a production environment.
 --tls-engine : If set, enables the use of a TLS engine device.
 --tls-engine-kpass-sha1 : SHA1 of the key password to be used with the selected SSL engine.
 --tls-use-os-certs : Load and trust OS provided CA certificates.
 --psk : pre-shared-key in hexadecimal (no leading 0x) to enable TLS-PSK mode.
 --psk-identity : client identity string for TLS-PSK mode.
 --proxy : SOCKS5 proxy URL of the form:
           socks5h://[username[:password]@]hostname[:port]
           Only "none" and "username" authentication is supported.

See https://mosquitto.org/ for more information.

~~~



## go项目创建和调试工具安装
~~~sh
go mod init tunnel-server
go mod install github.com/eclipse/paho.mqtt.golang
go mod tidy

#go开发调试环境

go install -v github.com/go-delve/delve/cmd/dlv@latest

~~~




