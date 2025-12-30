---
title: "SSHでパスワード+ワンタイムパスコードで認証する方法(Fedora)"
tags: []
date: 2025-04-08T10:17:54+09:00
---

<https://blog.apar.jp/linux/12502/>とか<https://security.sios.jp/security/ssh_google_authenticator/>とか<https://soji256.hatenablog.jp/entry/2020/05/17/150250#fn-5bcc649a>とかあったけど、どれもうまくいかなかったのでメモ。SSHのバージョンが新しいからかな？

Version: Fedora 41

/etc/ssh/sshd_config:  
以下の2行を置き換え。

```
PasswordAuthentication yes
#PermitEmptyPasswords no

# Change to no to disable s/key passwords
KbdInteractiveAuthentication yes
```

/etc/pam.d/sshd:  
ここで他のガイドに書いてあるのと全く内容が違ったのでなにを書いていいか詰まった。幸い、`forward_pass`と`use_first_pass`を使うと、最初に両方のパスを入力して片方が下流に流されるみたいなことができるらしいので、そのように改造した。また、SELinuxをオンにしていると、sshdは`.ssh`にしか見に行けないので、そこにauthenticatorのファイルを配置することに。   
最初の3行を以下のように置き換え   
```
#%PAM-1.0
auth required pam_google_authenticator.so secret=/home/${USER}/.ssh/.google_authenticator forward_pass nullok
auth       substack     password-auth use_first_pass
auth       include      postlogin
account    required     pam_sepermit.so
account    required     pam_nologin.so
account    include      password-auth
password   include      password-auth
# pam_selinux.so close should be the first session rule
session    required     pam_selinux.so close
session    required     pam_loginuid.so
# pam_selinux.so open should only be followed by sessions to be executed in the user context
session    required     pam_selinux.so open env_params
session    required     pam_namespace.so
session    optional     pam_keyinit.so force revoke
session    optional     pam_motd.so
session    include      password-auth
session    include      postlogin
```

その後、google-authenticatorコマンドを実行して、`~/.google_authenticator`ファイルを生成する。

SELinuxをオンにしていると、sshdが`~/.google_authenticator`にアクセスできない。
かといって`~/.google_authenticator`のファイルコンテキストを`ssh_home_t`にしてもうまくいかない(なんと同一階層に一時ファイルが生成される仕様！)。このファイルが入っているフォルダごとコンテキストをいじらなければならないが、`~/`をいじるのはどう考えても無理なので、結局このファイルは`~/.ssh`にいれることになる。

```
mv ~/.google_authenticator ~/.ssh
sudo restorecon -R ~/.ssh
```

パスワードは、ユーザーパスワード+6桁のワンタイムパスワードを連続して入力してログインできる。

Sudoとかでも同じようなことをやりたいなら、同様にディレクトリを用意して、semanage fcontext で適切にコンテキストを設定する必要あり？(sudoはuid transitionはするけどcontext transitionはなさそう? sudoersと同じコンテキストにすれば大丈夫かな？)  
ただまあ当然、ファイルの場所は`~/`じゃなくて`/etc/sudo_otp/$USER`とかのほうがいいか。そうすればコンテキスト設定しなくてもいいしね
