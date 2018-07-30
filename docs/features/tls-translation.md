# 通过TLS保护pouch
如果客户端仅从同一服务器连接到pouchd，则只应使用unix socket，这意味着使用参数-l unix:///var/run/pouchd.sock启动pouchd.如果通过远程服务器上的http客户端连接pouchd，则pouchd应该监听tcp端口，例如：-l 0.0.0.0:4243，在这种情况下，如果我们不尽兴TLS保护的话，pouchd将不会对远程身份加以校验从而接受所有的连接，这在生产环境中绝对是非常危险的。

为了校验客户端身份，我们对pouchd和客户端进行CA认证。

## create a CA
如下所示，使用通用名字pouch_test来创建一个CA申请
openssl req -new -newkey rsa:2048 -days 3650 -nodes -x509 -subj "/C=CN/ST=ZheJiang/L=HangZhou/O=Company/OU=Department/CN=pouch_test" -keyout ca-key.pem -out ca.pem
应用上述命令后，在当前目录文件有两个名叫ca.pem和ca-key.pem的文件，就是CA。切记一定要保密。
## create a certificate for daemon
为pouchd创建一个证书，并且这个证书只能作为服务器证书使用，HOSTNAME是运行pouchd的机器对变量名。

name=$HOSTNAME
mkdir -p ${name}
# create a key
/usr/bin/openssl genrsa -out ${name}/key.pem 2048
# create a csr
/usr/bin/openssl req -subj "/C=CN/ST=ZheJiang/L=HangZhou/O=Company/OU=Department/CN=${name}" -new -key ${name}/key.pem -out ${name}/$name.csr
# generate a certificate
/usr/bin/openssl x509 -req -days 3650 -in ${name}/${name}.csr -CA ca.pem -CAkey ca-key.pem -CAcreateserial -out ${name}/cert.pem -extfile <(echo "extendedKeyUsage = serverAuth")
cp ca.pem ${name}/ca.pem

完成上面的任务之后，会有一个包含了所有我们需要设置pouchd的TLS protection文件，一般用以下命令来设置启动pouch（变量用正确的值替换掉即可）：

--tlsverify --tlscacert=${name}/ca.pem --tlscert=${name}/cert.pem --tlskey=${name}/key.pem

以上命令执行完成后，当启动pouchd，它监听的tcp地址只能与使用相同CA证书的客户端建立连接。

## create a client certificate
name=a_client
mkdir -p ${name}
# create a key
/usr/bin/openssl genrsa -out ${name}/key.pem 2048
# create a csr
/usr/bin/openssl req -subj "/C=CN/ST=ZheJiang/L=HangZhou/O=Company/OU=Department/CN=${name}" -new -key ${name}/key.pem -out ${name}/$name.csr
# generate a certificate
/usr/bin/openssl x509 -req -days 3650 -in ${name}/${name}.csr -CA ca.pem -CAkey ca-key.pem -CAcreateserial -out ${name}/cert.pem -extfile <(echo "extendedKeyUsage = clientAuth")
cp ca.pem ${name}/ca.pem

完成上面的任务之后，目录里将出现一个包含了所有我们需要设置pouchd的TLS protection文件，一般用以下命令来设置启动pouch（变量用正确的值替换掉即可）：

--tlsverify --tlscacert=${name}/ca.pem --tlscert=${name}/cert.pem --tlskey=${name}/key.pem

以上命令执行完成后，目录里将出现一个包含了所有我们需要使用的PouchContainer客户端，例如：可使用此证书用作身份验证来获取pouchd service的版本。

./pouch -H ${server_hostname}:4243 --tlsverify --tlscacert=${path}/ca.pem --tlscert=${path}/cert.pem --tlskey=${path}/key.pem version

当一个没有证书的客户端或者一个不是由同一CA发布的证书的客户端想要尝试连接具有TLS保护的pouchd时，连接将被拒绝。



