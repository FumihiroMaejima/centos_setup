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

```shell-session
# vi /etc/ssh/sshd_config
```

ssh接続の変更内容の読み込み(再起動)

```shell-session
# systemctl restart sshd
# systemctl status sshd -l
```


## header2

```shell-session

```
