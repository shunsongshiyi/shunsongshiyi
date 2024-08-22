---
layout:     post 
title:      "go实现SSH访问远程机器"
subtitle:   "go如何实现ssh远程登录详解"
date:       2024-08-21
author:     "赵时宜"
URL: "/2024/08/21/hello-world/"
image:      "/img/post-bg-coffee.jpg"
---
<!-- 

---
layout:     post
title:      "go语言实现SSH公钥访问远程机器"
subtitle:   "go如何实现ssh远程登录详解"
description: "Istio是来自Google，IBM和Lyft的一个Service Mesh（服务网格）开源项目，是Google继Kubernetes之后的又一大作,本文将演示如何从裸机开始从零搭建Istio及Bookinfo示例程序。"
excerpt: "Istio是来自Google，IBM和Lyft的一个Service Mesh（服务网格）开源项目，是Google继Kubernetes之后的又一大作,本文将演示如何从裸机开始从零搭建Istio及Bookinfo示例程序。"
date:     2024-08-22T12:00:00
author:      "赵时宜"
image: "/img/post-bg-coffee.png"
tags:
    - SSH
    - Go
URL: "/2024/08/22/ssh/"
categories: [ Tech ]
--- -->


### 摘要
SSH全称叫 Secure Shell（安全外壳协议，简称SSH），其实是一种加密的网络传输协议，能够在不安全的网络中为计算机之间的网络服务提供安全的传输环境。

SSH最常见的用途是在本地远程登录服务器，或者远程使用命令行界面执行远程命令。

### ssh登录
ssh 登录的两种方式

1. 密码登录
   （1）远程主机收到用户的登录请求，把自己的公钥发给用户。
   （2）用户使用这个公钥，将登录密码加密后，发送回来。
   （3）远程主机用自己的私钥，解密登录密码，如果密码正确，就同意用户登录。

   

2. 公钥登录

   使用密码登录，每次都必须输入密码，非常麻烦。好在SSH还提供了公钥登录，可以省去输入密码的步骤。

   所谓"公钥登录"，原理很简单，就是用户将自己的公钥储存在远程主机上。登录的时候，远程主机会向用户发送一段随机字符串，用户用自己的私钥加密后，再发回来。远程主机用事先储存的公钥进行解密，如果成功，就证明用户是可信的，直接允许登录shell，不再要求密码。

3. 公钥登录准备工作：

   1. 用户生成密钥 默认存放在路径下$HOME/.ssh, 会新生成两个文件id_rsa.pub和id_rsa, 也就是公钥和私钥。 

      `ssh-keygen`

   2. 以下命令将公钥发送至远程主机host

      ssh-copy-id 会将刚创建的公钥内容追加写入至远程机器 的$HOME/.ssh/authorized_keys 文件内,等同于命令 
      ```ssh user@host 'mkdir -p .ssh && cat >> .ssh/authorized_keys' < ~/.ssh/id_rsa.pub```
   
      ```ssh-copy-id user@host```

   3. 设置远程主机允许公钥登录，打开

      /etc/ssh/sshd_config这个文件，将以下几行注释掉，去掉#注释符号

      `　RSAAuthentication yes PubkeyAuthentication yes AuthorizedKeysFile .ssh/authorized_keys`

   4. 命令行公钥登录验证, ssh登录远程host成功 

      `ssh user@host`

   5. golang 通过ssh公钥访问远程server, 并执行命令

      ```go
      type SSH struct {
      	IP      string
      	User    string
      	Cert    string //key file path
      	Port    int
      	session *ssh.Session
      	client  *ssh.Client
      }
      
      func (S *SSH) readPublicKeyFile(file string) ssh.AuthMethod {
      	buffer, err := os.ReadFile(file)
      	if err != nil {
      		return nil
      	}
      
      	key, err := ssh.ParsePrivateKey(buffer)
      	if err != nil {
      		return nil
      	}
      	return ssh.PublicKeys(key)
      }
      
      // Connect the SSH Server
      func (S *SSH) Connect() {
      	var sshConfig *ssh.ClientConfig
      	var auth = []ssh.AuthMethod{
      		S.readPublicKeyFile(S.Cert),
      	}
      	sshConfig = &ssh.ClientConfig{
      		User: S.User,
      		Auth: auth,
      		HostKeyCallback: func(hostname string, remote net.Addr, key ssh.PublicKey) error {
      			return nil
      		},
      		Timeout: time.Second * 3,
      	}
      
      	client, err := ssh.Dial("tcp", fmt.Sprintf("%s:%d", S.IP, S.Port), sshConfig)
      	if err != nil {
      		fmt.Println(err)
      		return
      	}
      
      	session, err := client.NewSession()
      	if err != nil {
      		fmt.Println(err)
      		client.Close()
      		return
      	}
      
      	S.session = session
      	S.client = client
      }
      
      func (S *SSH) Close() {
      	S.session.Close()
      	S.client.Close()
      }
      
      // RunCmd to SSH Server
      func (S *SSH) RunCmd(cmd string) {
      	out, err := S.session.CombinedOutput(cmd)
      	if err != nil {
      		fmt.Println(err)
      	}
      	fmt.Println(string(out))
      }
      func TestSSH(t *testing.T) {
      
      	client := &SSH{
      		IP:   "ip",
      		User: "jenkins",
      		Port: 22,
      		Cert: "/Users/mrzhao/.ssh/id_rsa",
      	}
      	client.Connect() 
      	client.RunCmd("ls -al")
      	client.Close()
      }
      ```

      

### go语言实现
  go语言实现通过跳板机创建ssh连接访问VM, 前置条件可参照上述说明在跳板机和VM中配置访问机器的ssh公钥

```go
func ConnectVMThroughBastion() {

	bastionHost := "bastionIP:22"
	bastionUser := "bastionUser"
	cert := "/Users/mrzhao/.ssh/id_rsa"

	vmHost := "vmip:22"
	vmUser := "vmUser"
	vmCert := "/Users/mrzhao/.ssh/id_rsa"
	sshConfig := &ssh.ClientConfig{
		User: bastionUser,
		Auth: []ssh.AuthMethod{
			readPublicKeyFile(cert),
		},
		HostKeyCallback: func(hostname string, remote net.Addr, key ssh.PublicKey) error {
			return nil
		},
		Timeout: time.Second * 3,
	}
	vmSshConfig := &ssh.ClientConfig{
		User: vmUser,
		Auth: []ssh.AuthMethod{
			readPublicKeyFile(vmCert),
		},
		HostKeyCallback: func(hostname string, remote net.Addr, key ssh.PublicKey) error {
			return nil
		},
		Timeout: time.Second * 3,
	}

	//connect to he bastion host
	client, err := ssh.Dial("tcp", bastionHost, sshConfig)
	if err != nil {
		fmt.Println(err)
		return
	}
	defer client.Close()

	// Dial a connection to the service host, from the bastion
	conn, err := client.Dial("tcp", vmHost)
	if err != nil {
		fmt.Println(err)
		return
	}
	//establishes an authenticated SSH connection
	authConn, chans, reqs, err := ssh.NewClientConn(conn, vmHost, vmSshConfig)
	if err != nil {
		fmt.Println(err)
		return
	}
	//NewClient creates a Client on top of the given connection.
	//vmClient is an ssh client connected to the service host, through the bastion host.
	vmClient := ssh.NewClient(authConn, chans, reqs)
	defer vmClient.Close()
	session, err := vmClient.NewSession()
	if err != nil {
		fmt.Println(err)
		return
	}

	defer session.Close()
	//execute cmd
	out, err := session.CombinedOutput("ls -al")
	if err != nil {
		fmt.Println(err)
	}
	fmt.Println(string(out))
}
```
