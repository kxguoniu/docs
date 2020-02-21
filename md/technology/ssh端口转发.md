# ssh 端口转发

## 应用
```
ssh -i .ssh/id_rsa -o StrictHostKeyChecking=no -o TCPKeepAlive=yes -o ServerAliveInterval=30 -fCNR forword_port:server_ip:server_port username@forword_ip
```

## 选项介绍
- -f 后台启动
- -N 不执行远程命令
- -C 压缩数据
- -R 远程转发
- -o StrictHostKeyChecking=no 自动添加秘钥，不需要手动确认。
- -o TCPKeepAlive=yes  TCP keepalive
- -o ServerAliveInterval=30 无通信30s请求服务器响应
- forword_port 远程转发端口
- server_ip 数据转发的ip地址
- server_port 数据转发的端口
- username 远程主机用户名
- forword_ip 远程主机ip地址

## 示例
ssh 22 端口转发
- A  A_name  A_ip
- B  B_name  B_ip
- C  C_name  C)ip

A 可以与 B 通信
B 可以与 C 通信
现在要目的要是A ssh 到 C

C 主机：
```
ssh -i .ssh/id_rsa -o StrictHostKeyChecking=no -o TCPKeepAlive=yes -o ServerAliveInterval=30 -fCNR 223:localhost:22 B_name@B_ip
```

A 主机执行
```
ssh -i .ssh/id_rsa C_name@B_ip -p 223
```
就可以 ssh 到 C 主机

## 配置
当我们实际操作的时候可能会发现在 A 主机上执行命令那一步可能会连接不上。
这时我们去 B 主机上查看可以发现 B 监听的地址是 127.0.0.1:223。
解决这个问题需要我们在配置文件 /etc/ssh/sshd_config 添加 GateWayPorts yes
然后执行  servie sshd restart，这样B服务器上的端口就会监听在 0.0.0.0地址上了。
