# CentOS Setup Environmental Version7

CentOS7の環境構築方法をまとめる
*ちなみに「さくらのVPS」のCentOS7の環境を参考にしている。

## OSのバージョンのチェック

```shell-session
# cat /etc/system-release
```

## rootパスワードの変更

```shell-session
# passwd
```

## ユーザーの作成

```shell-session
# useradd test_user
```

## 作成したユーザーのパスワードの追加

```shell-session
# passwd test_user
```

## sudoの設定

ユーザーをwheelグループに追加

```shell-session
# usermod -G wheel test_user
```

wheelグループの権限を設定

```shell-session
# visudo
## Allows people in group wheel to run all commands
%wheel  ALL=(ALL)       ALL

管理者権限を付与する場合
## Allow root to run any commands anywhere
root    ALL=(ALL)   ALL
test_user  ALL=(ALL)  ALL

```

## 公開鍵認証の設定

ローカルのPCからssh接続出来る様にする為に専用の鍵を作る

```shell-session
$ ssh-keygen -t rsa -b 2048 -C your_email@example.com -f id_rsa_centos7
```

ssh_configの設定

```shell-session
Host centos_host
  HostName 127.0.0.1
  User test_user
  port 22
  IdentityFile ~/.ssh/id_rsa_centos7
  IdentitiesOnly yes
```

公開鍵をサーバーに転送

```shell-session
$ scp ~/id_rsa_centos7.pub test_user@127.0.0.1s:/home/test_user/id_rsa_centos7.pub
```

サーバーに移動して公開鍵認証の設定を行う

```shell-session
$ mkdir .ssh
$ chmod 700 .ssh
$ cat id_rsa_centos7.pub > .ssh/authorized_keys
$ chmod 600 .ssh/authorized_keys
```

## ssh接続の設定変更

sshd_configを開き、下記の対応を記述する
・portを任意の値にする
・PermitRootLoginをnoに変更する
・PasswordAuthenticationをnoに変更する
・PermitEmptyPasswordsをnoに変更する

```shell-session
# vi /etc/ssh/sshd_config
```

ssh可能ユーザーを限定する

AllowUsersはwhiltelist方式
DenyUsersでblacklist方式

デプロイ自動化する時は注意が必要

```
AllowUsers test_user git
# DenyUsers another_user
```

```shell-session
LoginGraceTime 30
MaxAuthTries 3
```

認証するまでの猶予時間LoginGraceTimeと、試行回数MaxAuthTriesを設定する

```shell-session
LoginGraceTime 30
MaxAuthTries 3
```

ssh接続の変更内容の読み込み(再起動)

```shell-session
# systemctl restart sshd
# systemctl status sshd -l
```


## firewalldの設定

Firewallで、sshでアクセスするポートを開ける

```shell-session
$ sudo firewall-cmd --permanent --add-port=10222/tcp
```

ポートが開いているか確認

```shell-session
$ sudo firewall-cmd --list-ports
10222/tcp
```

## 秘密鍵を使ってのSSH接続

```shell-session
ssh -p 10222 -i ~/.ssh/id_rsa_centos7 test_user@127.0.0.1
```

## サーバーをシャットダウンする時

```shell-session
$ sudo shutdown now
```

## SELinuxの停止

SELinuxをdisabledにする。

```shell-session
#  vi  etc/sysconfig/selinux
SELINUX=disabled
```

## ssh.xmlの設定

```shell-session
# cp /usr/lib/firewalld/services/ssh.xml /etc/firewalld/services/ssh.xml
# vi /etc/firewalld/services/ssh.xml
<port protocol="tcp" port="22"/>
⇨下記の通りに変更
<port protocol="tcp" port="10222"/>
```

```shell-session
$sudo firewall-cmd --reload
```

## ポートスキャン対策

```shell-session
# firewall-cmd --permanent --direct --add-chain ipv4 filter port-scan
# firewall-cmd --permanent --direct --add-rule ipv4 filter INPUT 400 -i eno16777736 -p tcp --tcp-flags SYN,ACK,FIN,RST SYN -j port-scan
# firewall-cmd --permanent --direct --add-rule ipv4 filter port-scan 450 -m limit --limit 1/s --limit-burst 4 -j RETURN
# firewall-cmd --permanent --direct --add-rule ipv4 filter port-scan 451 -j LOG --log-prefix "IPTABLES PORT-SCAN:"
# firewall-cmd --permanent --direct --add-rule ipv4 filter port-scan 452 -j DROP

```

⇨「eno16777736」は「eth0」でいい気がする
*eno16777736=eth0
ネットワークインターフェイスのこと。

```shell-session
firewall-cmd --permanent --direct --add-rule ipv4 filter INPUT 400 -i eth0 -p tcp --tcp-flags SYN,ACK,FIN,RST SYN -j port-scan
```

## ポートスキャン対策 ルール追加

limitモジュールを使用して1秒間にSYNパケットが5以上連続するとログに書き出して以降はSYNパケットをドロップさせる｡

```shell-session
# firewall-cmd --permanent --direct --add-chain ipv4 filter port-scan
# firewall-cmd --permanent --direct --add-rule ipv4 filter INPUT 400 -i eth0 -p tcp --tcp-flags SYN,ACK,FIN,RST SYN -j port-scan
# firewall-cmd --permanent --direct --add-rule ipv4 filter port-scan 450 -m limit --limit 1/s --limit-burst 4 -j RETURN
 firewall-cmd --permanent --direct --add-rule ipv4 filter port-scan 451 -j LOG --log-prefix "IPTABLES PORT-SCAN:"
# firewall-cmd --permanent --direct --add-rule ipv4 filter port-scan 452 -j DROP
```

```shell-session
# firewall-cmd --reload
```

```shell-session
# firewall-cmd --direct --get-all-chains
ipv4 filter port-scan

# firewall-cmd --direct --get-all-rules
ipv4 filter port-scan 450 -m limit --limit 1/s --limit-burst 4 -j RETURN
ipv4 filter port-scan 451 -j LOG --log-prefix 'IPTABLES PORT-SCAN:'
ipv4 filter port-scan 452 -j DROP
ipv4 filter INPUT 400 -i eth0 -p tcp --tcp-flags SYN,ACK,FIN,RST SYN -j port-scan
ipv4 filter INPUT 0 -p tcp -m multiport --dports ssh -m set --match-set fail2ban-sshd src -j REJECT --reject-with icmp-port-unreachable
```


```shell-session
# firewall-cmd --reload
```


```shell-session
# iptables -L -v
⇨下記の設定が追加されている。

Chain port-scan (1 references)
 pkts bytes target     prot opt in     out     source               destination
   12   504 RETURN     all  --  any    any     anywhere             anywhere             limit: avg 1/sec burst 4
    0     0 LOG        all  --  any    any     anywhere             anywhere             LOG level warning prefix "IPTABLES PORT-SCAN:"
    0     0 DROP       all  --  any    any     anywhere             anywhere
```

## IP Spoofing(なりすまし) の対応

外部からやってくるローカルIPは破棄

```shell-session
# firewall-cmd --permanent --direct --add-rule ipv4 filter INPUT 0 -i enp0s3 -s 127.0.0.1/8 -j DROP
# firewall-cmd --permanent --direct --add-rule ipv4 filter INPUT 0 -i enp0s3 -s 10.0.0.0/8 -j DROP #開発環境では許可
# firewall-cmd --permanent --direct --add-rule ipv4 filter INPUT 0 -i enp0s3 -s 172.16.0.0/12 -j DROP
# # firewall-cmd --permanent --direct --add-rule ipv4 filter INPUT 0 -i enp0s3 -s 192.168.0.0/16 -j DROP  #開発環境では許可
# # firewall-cmd --permanent --direct --add-rule ipv4 filter INPUT 0 -i enp0s3 -s 192.168.0.0/24  -j DROP #開発環境では許可
```





## header2

```shell-session

```
