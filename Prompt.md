# DNS-MailServer-Ubuntu

## 最初の質問

< こんにちは。Ubuntu 用の、DNS (Bind)と postfix のインストール、設定についてサポートをお願いします。

現在、Ubuntu 22.04.5 LTS で DNS (Bind), BIND 9.18.39-0ubuntu0.22.04.2-Ubuntu と、postfix (3.6.4-1ubuntu1.3) を運用しています。
サーバー用途マシンは2台あり、昨日そのうちの1台がHDDの故障で丸ごと使えなくなったので、この機会に2台とも Ubuntu 24.04.3 に更新しようと思います。

1台は故障マシンを入れ替えて、新規インストールです。昨夜、新しい Ubuntu 24.04.3 Desktop のインストールが完了したので、これを使います。
もう1台は現在運用中の、Ubuntu 22.04.5 LTS を更新して使う予定です。
こっちは現在運用中なので、前述のUbuntu 24.04.3 Desktop の新サーバーが稼働してから更新する予定です。
各設定ファイルは、この現在運用中の Ubuntu 22.04.5 LTS から極力流用します。

私は、久しぶりの Ubuntu へのBindと postfix インストールなので、手順を忘れたりしていますし、最新の状況を知りません。

現在は、新マシンの　Ubuntu 24.04.3 の標準インストールが完了し、apt update で最新に更新した状態です。

ますは、これに最新のBINDをなるべく簡単にインストールする手順からを教えてください。
このマシンのBINDの動作を確認してから、postfix をインストールして動作させます。

よろしくお願いします。

> こんにちは。状況と計画、とても整理されていて素晴らしいです 👍
「新規 Ubuntu 24.04.3 → BIND → 動作確認 → postfix → 旧サーバ更新」という流れは非常に良い手順です。

まずは Ubuntu 24.04.3 Desktop に BIND を最小構成で入れて動作確認するところまで を、できるだけシンプルにまとめます。

---

## 1. 事前確認（Ubuntu 24.04）

すでに apt update 済みとのことなので OK ですが、念のため：

sudo apt update
sudo apt upgrade


サーバ用途なら Desktop でも問題ありません。
（後で不要な GUI は削れます）

## 2. BIND インストール

Ubuntu 24.04 の BIND は 9.18 系（LTS）です。
基本パッケージだけ入れます。

```
sudo apt install bind9 bind9utils bind9-dnsutils
```

## 3. サービス状態確認
```
systemctl status bind9
```

有効化されていなければ：（不要）
```
sudo systemctl enable bind9
sudo systemctl start bind9
```

## 4. ポート確認（重要）

DNS は TCP/UDP 53 を使います。

```
sudo ss -tulpen | grep 53
```

5. 設定ファイル構成（24.04）

Ubuntu 22.04 とほぼ同じです。

/etc/bind/
 ├─ named.conf
 ├─ named.conf.options
 ├─ named.conf.local
 ├─ named.conf.default-zones

移行時に流用するのは主に：

  - named.conf.options
  - named.conf.local
  - zone ファイル


## 6. 最小動作確認（キャッシュDNS）

まずは「外部名前解決だけできる状態」にします。
/etc/bind/named.conf.options を編集。

```
sudo nano /etc/bind/named.conf.options
```

options {
    directory "/var/cache/bind";

    recursion yes;
    allow-recursion { 127.0.0.1; };

    listen-on { any; };
    listen-on-v6 { any; };

    dnssec-validation auto;
};

※ 最初は外部公開せず localhost 限定 にすると安全です。

## 7. 設定チェック

```
sudo named-checkconf

```

エラーがなければ：

```
sudo systemctl restart bind9

```

## 8. 動作確認（dig）

```
dig @127.0.0.1 google.com
```

;; ANSWER SECTION:
google.com.   300 IN A 142.xxx.xxx.xxx

## 9. ファイアウォール確認（UFW）

もし UFW 有効なら：

```
sudo ufw allow 53/tcp
sudo ufw allow 53/udp
```

## 10. ここまでのゴール

この時点で：

- named 起動 OK
- DNS キャッシュサーバとして動作
- 設定ファイル構造確認完了
- 22.04 → 24.04 移行の下地完成


## 次のステップ（まだやらない）

 ---

 ### 次に進むときは：

- 旧 22.04 の /etc/bind/ をコピー
- zone ファイル確認
- named-checkzone
- postfix インストール

### 移行時の注意点

24.04 での主な違いはほぼありませんが：

- systemd-resolved が 53 番を掴む場合あり
- /etc/resolv.conf の確認

### もしポート競合が起きたら：

```
sudo systemctl disable systemd-resolved
sudo systemctl stop systemd-resolved
```

まずはここまでで 「BIND 単体が正しく動く」ことを確認 してください。
postfix は DNS が安定してからの方が安全です。

---

<
ありがとう。

uwf が有効か無効か不明だったので、uwf が有効だと想定して作業しました。
その結果、以下の表示なのおそらく、これで良かったと思います。

root@purple:/etc/bind# ufw allow 53/tcp
ルールをアップデートしました
ルールをアップデートしました(v6)
root@purple:/etc/bind# ufw allow 53/udp
ルールをアップデートしました
ルールをアップデートしました(v6)

ところで、
dig @127.0.0.1 google.com

の結果は次の通りで意味不明でした。

```
root@purple:/etc/bind# dig @127.0.0.1 google.com

; <<>> DiG 9.18.39-0ubuntu0.24.04.2-Ubuntu <<>> @127.0.0.1 google.com
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: SERVFAIL, id: 60553
;; flags: qr rd ra; QUERY: 1, ANSWER: 0, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: 3a4a8579c8324e36010000006985565bd44384760080cc50 (good)
;; QUESTION SECTION:
;google.com.                    IN      A

;; Query time: 171 msec
;; SERVER: 127.0.0.1#53(127.0.0.1) (UDP)
;; WHEN: Fri Feb 06 11:47:55 JST 2026
;; MSG SIZE  rcvd: 67
```

しかし ibm.com で試すと、
```
root@purple:/etc/bind# dig @127.0.0.1 ibm.com

; <<>> DiG 9.18.39-0ubuntu0.24.04.2-Ubuntu <<>> @127.0.0.1 ibm.com
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 22530
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: b522f0b23cdceca801000000698556a354272a8964f75c82 (good)
;; QUESTION SECTION:
;ibm.com.                       IN      A

;; ANSWER SECTION:
ibm.com.                15      IN      A       23.42.65.233

;; Query time: 472 msec
;; SERVER: 127.0.0.1#53(127.0.0.1) (UDP)
;; WHEN: Fri Feb 06 11:49:07 JST 2026
;; MSG SIZE  rcvd: 80
```
だったので、問題が無いと考えます。
「BIND 単体が正しく動く」ことを確認出来たと思います。

次のアドバイスをお願いします。それから、postfix の設定の後で「より安全なuwfの設定も教えてください」
今は、昔自前で作った、iptables のスクリプトを使っているので、これも後で uwf に置き換えたいと考えています。
次をお願いします。

---

>とても良い確認結果です 👍  
結論から言うと：

- BIND は正常に動作しています
- google.com → SERVFAIL は「よくある一時的／DNSSEC絡みの挙動」
- ibm.com → NOERROR が返っている時点で キャッシュDNSとしては合格

なので、現段階では BIND 単体 OK 判定で問題ありません。

---
## 1. google.com が SERVFAIL になる理由

これは BIND では割と見ます。主な原因は：

- (1) DNSSEC 検証失敗
- (2) IPv6 経路問題
- (3) 一時的ネットワーク問題

-- 省略


## 2. 次のステップ：postfix 前の軽い確認

```
journalctl -u bind9 -n 30
```
エラーが無ければ完璧です。

---

### 3. postfix 後に行う UFW 設計（重要）

今は 53 全開で OK ですが、最終的には以下のようにすると安全です。

#### DNS 専用サーバの場合

```
sudo ufw default deny incoming
sudo ufw default allow outgoing

sudo ufw allow 53/tcp
sudo ufw allow 53/udp
sudo ufw allow 22/tcp   # SSH
```

#### DNS + Mail サーバの場合

postfix 用に追加：

```
sudo ufw allow 25/tcp   # SMTP
sudo ufw allow 587/tcp  # Submission
sudo ufw allow 465/tcp  # SMTPS (必要なら)
```

### 4. iptables → UFW 移行の考え方

あなたの現在の構成から見ると、  
iptables スクリプトはかなり精密だと思われます。

UFW に移行する際の方針は：

```
iptables	UFW
---
INPUT DROP	default deny incoming
OUTPUT ACCEPT	default allow outgoing
特定ポート許可	ufw allow
```

**順番が重要です。**

```
sudo ufw enable
sudo ufw status verbose
```

## 5. DNS サーバでやりがちな危険設定

## 6. trusted-nets の例

## 7. 今後の理想状態

## 8. 現時点の評価まとめ
-- これらは対応済のため、省略

---

<
はいIPV6 は私のルーターが対応していないので、完全無効のままで、進めます。もし追加設定があれば、アドバイスください。

>はい、その方針でとても良いです 👍
**「IPv6 を完全に使わないサーバー」**として明示的に無効化しておくと、
DNS / postfix / Apache / UFW すべてが安定します。

すでに BIND 側は触られているので、
残りは OS・UFW・resolv.conf 周りを仕上げるだけでほぼ完璧です。

### 1. OS 全体で IPv6 無効化（最重要）

/etc/sysctl.conf に以下が入っていれば OK です。

```
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
```

反映：
```
sudo sysctl -p
```

確認：
```
ip a | grep inet6
```

何も表示されなければ成功です。


### 2. GRUB 側でも止める（より確実）

まれに sysctl より前に IPv6 が有効になるため、
サーバ用途では GRUB でも止めます。

```
sudo nano /etc/default/grub
```

以下を編集：

```
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash ipv6.disable=1"
```

反映：
```
sudo update-grub
sudo reboot
```

### 3. UFW の IPv6 無効化

```
sudo nano /etc/default/ufw
```

```
IPV6=no
```

反映：
```
sudo ufw reload
```

確認：
```
sudo ufw status verbose
```

IPv6 ルールが出なければ OK。

### 4. BIND 側最終確認

```
/etc/bind/named.conf.options

listen-on-v6 { none; };

```
これで十分です。

### 5. /etc/hosts の確認（地味に重要）

IPv6 を止めた環境では
localhost が IPv6 を向いていると postfix が混乱することがあります。

```
sudo nano /etc/hosts
```

最低限これがあれば OK

```
127.0.0.1   localhost
127.0.1.1   purple
```

```
::1 localhost は削除しても問題ありません。
```

### 6. resolv.conf の確認

```
cat /etc/resolv.conf
```
もし 127.0.0.53 なら：

```
sudo systemctl disable systemd-resolved
sudo systemctl stop systemd-resolved
```
済！

### 7. postfix への良い影響

IPv6 無効化で防げる問題：

```
症状	防止
SMTP 接続遅延	○
DNS 逆引き失敗	○
MX 解決遅延	○
メール送信5秒待ち	○
```

### 8. サーバ運用的な完成状態

あなたの環境では最終的にこうなります：

```
項目	状態
IPv6	完全無効
DNS	IPv4のみ
postfix	IPv4のみ
UFW	IPv4のみ
Apache	IPv4のみ
```

#### 追加でやると良い軽い確認

```
getent ahosts google.com
```

IPv4 だけ出れば成功です。

---

ここまでやれば Ubuntu 24.04 サーバとして非常に安定した基盤になります。
postfix に進む準備は万全です

---
<
ufw の ipv6 設定で次の状況です。

```
root@purple:~# sudo ufw reload
ファイアウォールは有効になっていません（再読込を飛ばします）
root@purple:~#
root@purple:~# sudo ufw status verbose
状態: 非アクティブ
```
となりました。

インターネットで検索しても、「ズバリ」ufw を有効化するだけの単純な方法がわかないので、教えてください。

> はい、これはとても単純です 👍
表示の通り UFW はまだ「有効化していない」だけ です。  

Ubuntu は UFW はインストールされていても初期状態は無効 です。  
なので検索しても「enable しろ」としか書いていないのですが、  
順番を間違えると SSH を自分で閉じる事故があるため、  
説明が回りくどくなっています。

あなたの環境ではローカル操作なので安全ですが、  
正しい手順を整理します。

---
### UFW を有効化する正しい手順

-- 省略

### 最終的な理想コマンドセット（DNS+SMTP の場合）

```
sudo ufw allow 22/tcp
sudo ufw allow 53/tcp
sudo ufw allow 53/udp
sudo ufw allow 25/tcp
sudo ufw allow 587/tcp
sudo ufw enable
```
今の状況は「まだスイッチを入れていないだけ」なので、
sudo ufw enable を **実行すれば完了** です。

---
<
現在、

getent ahosts google.com
で以下の表示です。

```
root@purple:/etc# getent ahosts google.com
142.250.194.78  STREAM google.com
142.250.194.78  DGRAM
142.250.194.78  RAW
```

よろしいでしょうか？

次は、現在運用中の、自前のドメインの BIND のレコードの設定の追加ですか？
その前に何か確認が必要であれば、教えてください。

それから最新の named.conf.default-zones ファイル等の基本的な設定内容も気になります。

その後で、もし簡単でお勧めの方法があれば、2台のDNS のレコードの同期方法を教えてください。
現在は、運用中のドメインのBIND レコードの修正や変更は滅多に無いので手作業で同期しています。

>はい、その表示は とても良い状態です 👍

```
getent ahosts google.com
142.250.xxx.xx
```
これは

IPv4 のみ解決

DNS → glibc → resolver 正常

BIND → OS → アプリ連携 OK

を意味します。
IPv6 無効化も正しく効いています。

---

## 事前の最終確認（2分）

### 1. BIND エラーログ

### 2. 逆引き確認

#### named.conf.default-zones の確認

**これは 絶対に触らない領域 です。  
**このままで正解。

---

## 自前ドメインの追加手順（基本）

### 1. named.conf.local 編集

### 2. ゾーンファイル作成

### 3. チェック

4. 反映

## 2台 DNS の同期方法

現在手作業とのことですが、あなたの運用スタイルなら 以下2択 が現実的です。

### 方法A：rsync（おすすめ・簡単）

### 方法B：BIND 正式同期（ゾーン転送）

<
ありがとう。順調に DNS 各所有しているドメインの、レコードの動作を確認しているところです。また rsync でDNSを同期させる方法は昔自分で考えて、すでに実現済だったので、このまま行きます。ところで念のため、ゾーンファイル作成 で、逆引きゾーンのレコードはどの様に記述すればいいのでしょうか？これまで自己流で記述していたので、この機会に一般的な逆引きゾーンの書き方を教えてください。参考にします。

>順調ですね 👍

rsync 同期も既に実施済とのことで、非常に堅実な運用だと思います。

では 逆引き（PTR）の一般的で安全な書き方 を整理します。
ここを整えておくと postfix・SPF・DKIM・SMTP の信頼性が一段上がります。

---

<ありがとう。すべての zone レコードと逆引きレコードを移行、作成しました。その後、各 作成した zone で、named-checkzone と、named-checkconf がエラーが無く、sudo systemctl reload bind9 して、dig コマンドで動作確認しました。これでやっと、postfix の設定に以降出来ますか？

## postfix に進む前の「最後の30秒チェック」

1. 自分のドメインを外向きで確認

2. MX 確認

3. FQDN 確認

postfix に進んで良い理由

postfix へ進む前の心構え

あなたの現在のDNSは

- 権威DNSとして正常
- 逆引き正常
- メールサーバ準備完了
- 冗長化済

実務レベルで言うと *DNS は完成 です。*

はい、いよいよ postfix に進みましょう 👍
ここまで DNS が整っていれば、postfix はかなり楽に進みます。

以下は Ubuntu 24.04 + IPv4のみ + 自前DNS運用サーバ を前提にした
最も安定しやすい手順です。

---

## 1. postfix インストール

```
sudo apt install postfix
```


## 2. 基本設定確認



## ここまでのゴール

- ローカル送信
- 外部送信 Gmail 受信
- ss 確認 25 LISTEN
- mail.log status=sent

この状態になれば **postfix 基本動作完了** です。
その後 DKIM / TLS / submission を追加していきます。

---

< ありがとう。postfix が外部との送受信が動き始めました。

ところで、9. SPF（すぐやると良い）DNS に TXT 追加：v=spf1 ip4:YOUR_IP -all のDNSの記載方法と、このれをｌ記述

---

<ありがとう。現在の手持ちのドメインが、次の記述になっていること確認しました。

9. SPF（すぐやると良い）DNS に TXT 追加：v=spf1 ip4:YOUR_IP -all のDNSの記載方法と、このれをｌ記述する意義？メリットについて、もう少し詳しく教えてください

> とても良い所に気が付かれています 👍

結論から言うと：  

**現在の**

```
v=spf1 mx -all
```
でも「正しく動作」はします。
しかし、あなたの運用形態（送信サーバが固定2台）であれば、

**IP を明示した方が評価はわずかに良く、将来的にも安全です。**

-- 省略


< ありがとう。よく分かりました。ipv4 にします。ところで、Windows PC でのメールの取得に、pop3 を稼働させたいです。これまでは昔の qpopper のソースコードを自前で改造して、pop before smtp として運用していました。似た様な運用、つまり pop3 で 各人のサーバーメールボックスから随時メールを取得する場合の、最近の一般的でシンプルな方法を教えてください。


> はい 👍

現在の Linux / postfix 環境で POP3 を提供する一番一般的でシンプルな方法 は、

**qpopper ではなく「Dovecot」を使う**

これが現在の標準です。



### インストール

```
sudo apt install dovecot-pop3d
```

IMAP も使うなら：

```
sudo apt install dovecot-imapd
```

---

### サービス起動

```
sudo systemctl enable dovecot
sudo systemctl start dovecot
```

### ポート確認

```
ss -tulpen | grep 110
```

### UFW 設定

```
sudo ufw allow 110/tcp
```

### Windows 側設定

-- 省略

### TLS（強く推奨）

-- 省略

### pop-before-smtp を使わない理由

-- 省略

---
## postfix 側 SMTP AUTH 有効化（後で）

```
smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth
smtpd_sasl_auth_enable = yes
```

## 結論

あなたの環境では：

- postfix → 既にOK
- dovecot → POP3 追加
- SMTP AUTH → 送信認証

これで pop-before-smtp より安全で簡単な構成 になります。  
**qpopper を卒業して Dovecot に移行するのが最善です。**

```
devxot start で次のエラーです。直し方を教えてください。
root@purple:~# systemctl start dovecot
Job for dovecot.service failed because the control process exited with error code.
See "systemctl status dovecot.service" and "journalctl -xeu dovecot.service" for details.
root@purple:~# systemctl status dovecot
× dovecot.service - Dovecot IMAP/POP3 email server
     Loaded: loaded (/usr/lib/systemd/system/dovecot.service; enabled; preset: enabled)
     Active: failed (Result: exit-code) since Sat 2026-02-07 23:41:36 JST; 17s ago
   Duration: 2.983s
       Docs: man:dovecot(1)
             https://doc.dovecot.org/
    Process: 15559 ExecStart=/usr/sbin/dovecot -F (code=exited, status=89)
   Main PID: 15559 (code=exited, status=89)
        CPU: 26ms

 2月 07 23:41:36 purple dovecot[15559]: master: Error: service(pop3-login): listen(::, 110) failed: Address family not supported b>
 2月 07 23:41:36 purple dovecot[15559]: Error: socket() failed: Address family not supported by protocol
 2月 07 23:41:36 purple dovecot[15559]: Error: service(pop3-login): listen(::, 995) failed: Address family not supported by protoc>
 2月 07 23:41:36 purple dovecot[15559]: master: Error: socket() failed: Address family not supported by protocol
 2月 07 23:41:36 purple dovecot[15559]: master: Error: service(pop3-login): listen(::, 995) failed: Address family not supported b>
 2月 07 23:41:36 purple dovecot[15559]: Fatal: Failed to start listeners
 2月 07 23:41:36 purple dovecot[15559]: master: Fatal: Failed to start listeners
 2月 07 23:41:36 purple systemd[1]: dovecot.service: Main process exited, code=exited, status=89/n/a
 2月 07 23:41:36 purple systemd[1]: dovecot.service: Failed with result
```

>原因はログに出ていて、IPv6 を無効化しているのに Dovecot が IPv6 アドレス :: で POP3(110/995) を listen しようとして失敗しています。

```
listen(::, 110) failed: Address family not supported
```
なので、Dovecot を IPv4 のみで listen するように設定すれば直ります。

- * は IPv4 の全IF（0.0.0.0）を意味します
- listen = *, :: になっていたら listen = * にします


