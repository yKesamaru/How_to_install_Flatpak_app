# Flatpakインストール周りの整理・まとめ

追加・更新：2024年5月23日

## はじめに
Ubuntu 22.04では標準で`Ubuntu software`というアプリケーションセンター的な仕組みが存在します。
カノニカルの方針で、Snapアプリケーションとdebアプリケーションはインストールできますが、Flatpakはインストールできません。
ここでは、Flatpakがすぐにインストールできるように、その構築法を整理し、再インストール時の再現性を確保します。
再インストール時のスクリプト活用については以下の投稿をご参照ください。

- [システム再インストールを楽にする手順: 専用BashScriptのすすめ](https://zenn.dev/ykesamaru/articles/1ab1297354d3c2)

あわせて、Flatpakなどの前提知識として、以下の投稿をご参照ください。

- [Flatpak vs. Snap. 違いと特性](https://zenn.dev/ykesamaru/articles/a9586cc52a376e)
- [日本語入力周りの整理・まとめ](https://zenn.dev/ykesamaru/articles/95ad7355c9bbba)

![](https://raw.githubusercontent.com/yKesamaru/How_to_install_Flatpak_app/master/assets/eye-catch.webp)

1. [Flatpakインストール周りの整理・まとめ](#flatpakインストール周りの整理まとめ)
   1. [はじめに](#はじめに)
   2. [環境](#環境)
   3. [前提条件](#前提条件)
      1. [補足：ユーザーレベルとシステムレベル](#補足ユーザーレベルとシステムレベル)
         1. [メリット](#メリット)
         2. [デメリット](#デメリット)
      2. [補足：レベル別のリモートリポジトリ追加](#補足レベル別のリモートリポジトリ追加)
   4. [セキュリティ基準を指定する](#セキュリティ基準を指定する)
      1. [サブセットの指定方法](#サブセットの指定方法)
         1. [補足](#補足)
   5. [ランタイムについて](#ランタイムについて)
   6. [CUIでのFlatpakアプリケーションインストール](#cuiでのflatpakアプリケーションインストール)
      1. [前処理](#前処理)
      2. [Flatpakアプリケーションインストール例](#flatpakアプリケーションインストール例)
         1. [アプリケーションIDを調べる方法](#アプリケーションidを調べる方法)
   7. [GUIでのシステムレベルインストール](#guiでのシステムレベルインストール)
   8. [パーミッション（権限）の管理](#パーミッション権限の管理)
      1. [CUI](#cui)
         1. [権限の確認](#権限の確認)
         2. [権限の操作](#権限の操作)
      2. [GUI](#gui)
   9. [参考文献](#参考文献)


## 環境
```bash
inxi -S
System:
  Host: **user** Kernel: 6.5.0-35-generic x86_64 bits: 64 Desktop: Unity
    Distro: Ubuntu 22.04.4 LTS (Jammy Jellyfish)
```

## 前提条件
1. リモートリポジトリは[Flathub](https://flathub.org/)とします。
2. Flatpakアプリケーションは基本的にユーザーレベルでのインストールとします
   1. 例外として`Boxes`などシステムレベルでのパーミッションがないと正常に動作しないものはこの限りではありません。
### 補足：ユーザーレベルとシステムレベル
システムの不安定化を最小限に留めるため、ユーザーレベルでのインストールで問題ないFlatpakアプリケーションに関しては、全てユーザーレベルでのインストールとしています。
このマイルールでは以下のメリットとデメリットが存在します。
#### メリット
- システム全体の不安定化を最小限にできる
- システム全体に関わるセキュリティリスクを最小限にできる
- システムの再インストール時、バックアップデバイスからコピーするだけでFlatpakアプリケーションを再構築できる（未検証）
#### デメリット
- ユーザーレベルでのセキュリティリスクは小さくならない
- Flatpakアプリケーションの管理がとても煩雑
- `gnome-software`などでGUIからFlatpakアプリケーションのインストールができない（すべてシステムレベルでインストールされてしまう）

これらのメリット・デメリットを鑑みて、インストールするFlatpakアプリケーションを全てシステムレベルにするのか、それともユーザーレベルでOKなFlatpakアプリケーションに関してはユーザーレベルでOKにするのか、前もって方針を決めておくことをおすすめします。

### 補足：レベル別のリモートリポジトリ追加
ユーザーレベルとシステムレベルで別々にFlatpakアプリケーションをインストールしたいのなら、それぞれのレベルでリモートリポジトリを追加しなくてはいけません。
以下は、それぞれのレベルでのリモートリポジトリ追加のコマンドラインです。（`verified`などは省略しています）
```bash
# システムレベルでのリモートリポジトリ追加
sudo flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo

# ユーザーレベルでのリモートリポジトリ追加
flatpak remote-add --user --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo
```

## セキュリティ基準を指定する
参考：[Installation: Flathub](https://docs.flathub.org/docs/for-users/installation)
使いたいFlatpakアプリケーションをインストールする前に、インストールするアプリケーションのフィルタリングを指定することができます。（前処理）
このフィルタリングを**サブセット設定**と呼びます。
サブセットを設定すると、そのサブセットの条件に一致するFlatpakアプリケーションのみがインストール可能となります。
例えば`verified`サブセットを設定した場合、検証済みのFlatpakアプリケーションのみがインストール可能となります。

なお、この操作は以後もCUIでFlatpakアプリケーションのインストールなどを行う場合限定となります。
### サブセットの指定方法
```bash
# 検証済みのFlatpakアプリケーションのみ(verified)を使用したい場合＆リモートリポジトリを追加する
flatpak remote-add --user --if-not-exists --subset=verified flathub-verified https://flathub.org/repo/flathub.flatpakrepo
```

既にFlathubリモートが追加されている場合は、次のコマンドでサブセットを変更できます。
```bash
flatpak remote-modify --user --subset=verified flathub
```
サブセットには以下の区分があります。
- `verified`: 検証済みなアプリケーションのみ
- `floss`: OSSアプリケーションのみ
- `verified_floss`: 検証済み＆OSSなアプリケーションのみ
上記以外だと「検証されていない」か「OSSではない」という条件になります。
`verified`サブセット以外のアプリケーションはFlathub管理者によって検証されていません。（といっても、検証されていないアプリケーションの方が圧倒的に多い状況です）

#### 補足
`--if-not-exists`というのは、「もし`flathub.org`をリモートリポジトリとしてまだ登録していなければ」の意味になります。
もしこの引数(`--if-not-exists`)を指定し忘れたらどうなるでしょうか？この場合、少しややこしいことになります。
例えばインストールスクリプト内で既にリモートリポジトリを指定していたとします。この時サブセットは指定していないとします。
そして今回、リモートリポジトリをサブセット指定して追加しました。
この状態はシステムには

1. サブセットを指定していないリモートリポジトリ
2. サブセットを指定したリモートリポジトリ

の２種類が登録されていることになります。
この状態は**予期しない挙動が発生するリスク**を生じます。
ですので、引数(`--if-not-exists`)は必ず指定する必要があります。
あるいは`flatpak remote-list`を端末で確認すると良いと思います。
```bash
flatpak remote-list
Name    Options
flathub system
flathub user
```
わたしの環境では、インストール時のスクリプトで既にFlathub.orgをリモートリポジトリとして登録しています。このような場合、それを忘れてリモートリポジトリを「追加で」「異なるサブセットで」指定するリスクがあります。
また、なにか不具合が起こった時、`flatpak remote-list`を打ち込んでみれば、その原因が分かるかも知れません。ちなみに`Options`には先程の`verified`などのサブセット名が入ります。

サブセットの削除などについても[Installation: Flathub](https://docs.flathub.org/docs/for-users/installation)に載っています。

![](https://raw.githubusercontent.com/yKesamaru/How_to_install_Flatpak_app/master/assets/2024-05-23-10-11-26.png)

## ランタイムについて
参照：[Flatpak vs. Snap. 違いと特性](https://zenn.dev/ykesamaru/articles/a9586cc52a376e#%E3%83%A9%E3%83%B3%E3%82%BF%E3%82%A4%E3%83%A0)
詳しくは上記をご参照ください。
ここでは簡単に紹介します。
Flatpakにおけるランタイムとは、各Flatpakアプリケーションが共通で参照するライブラリ群を指します。
ただし、特定のFlatpakアプリケーションが独自のライブラリを必要とする場合、それらアプリケーションがそれ用のランタイムをインストールすることがあります。

さて、全てのFlatpakアプリケーションをユーザーレベルでインストールするルールがあると仮定します。
この場合、Flatpakアプリケーションとそれら共通のランタイムは以下のディレクトリにインストールされます。
- Flatpakアプリケーション本体：`~/.local/share/flatpak/app/`
- ランタイム：`~/.local/share/flatpak/runtime/`

Flatpakアプリケーションを一つもインストールしていない状態では以下のようなディレクトリ構造になっています。
```bash
~/.local/share/flatpak$ tree ./
./
├── db
│   ├── background
│   ├── desktop-used-apps
│   ├── notifications
│   └── webextensions
├── overrides
│   └── org.mattbas.Glaxnimate
└── repo
    ├── config
    ├── extensions
    ├── objects
    ├── refs
    │   ├── heads
    │   ├── mirrors
    │   └── remotes
    ├── state
    └── tmp
        └── cache

12 directories, 6 files
```
ここでひとつFlatpakアプリケーションをインストールしてみましょう。
`flatpak install --user flathub com.github.tchx84.Flatseal`

```bash
tree -L 1
.
├── app
├── db
├── exports
├── overrides
├── repo
└── runtime
```
この様に`app`, `runtime`ディレクトリが追加されました。

## CUIでのFlatpakアプリケーションインストール
### 前処理
後述するGUIでのFlatpakアプリケーションインストールに比べ、CUIでのインストール操作は自由度が高くなります。具体的にはユーザーレベルとシステムレベルを分けてインストール可能です。そのメリット・デメリットは先述したとおりです。

それではFlatpakアプリケーションをインストールするための前処理を行いましょう。
```bash
# 最低限必要なパッケージ
sudo apt install -y flatpak
# qtを使ったパッケージをインストールする予定なら。
sudo apt install -y qt5-flatpak-platformtheme
```
`qt5-flatpak-platformtheme`は`Qt`を用いたFlatpakアプリケーションをインストールする予定なら入れておいたほうが良いでしょう。（現状の環境がGNOMEだと仮定しています）
### Flatpakアプリケーションインストール例
```bash
flatpak install --user flathub com.github.jeromerobert.pdfarranger
```
#### アプリケーションIDを調べる方法
[`Flathub`の公式サイト](https://flathub.org/)を参照するのが一番手っ取り早いですが、他の方法も存在します。
```bash
flatpak search <app_name>
# 例: pdfarrangerのアプリケーションIDを調べる
flatpak search pdfarranger
Name             Description                                                    Application ID                         Version    Branch    Remotes
PDF Arranger     PDF Merging, Rearranging, Splitting, Rotating and Cropping     com.github.jeromerobert.pdfarranger    1.10.1     stable    flathub

```
`apt search`と同じフォーマットで覚えやすいですね。

Flathubを参照する場合、インストールコマンドラインには気をつけてください。というのは**Flathubはシステムレベルでのインストールを前提としている**からです。ですのでコマンドラインの中に`--user`引数が記述されていません。単純にコピペすると本来システムレベルでのインストールになります。ですが現状、`--user`をつけ忘れた場合、ユーザーレベルなのかシステムレベルなのかをシステムが聞いてきてくれます。
```bash
flatpak install flathub org.kde.kdenlive
Looking for matches…
Remote ‘flathub’ found in multiple installations:

   1) system
   2) user

Which do you want to use (0 to abort)? [0-2]: 
```

## GUIでのシステムレベルインストール
FlatpakアプリケーションをGUIからインストールしたい時、`gnome-software`が利用できます。
先述の通り、標準でインストールされている`Ubuntu Software`はFlatpakを扱えません。
**ただし、システムレベルでのインストールしか選択肢はありません。**
**もしユーザーレベルでのインストールを希望する場合はCUIからのみとなります。**

それを踏まえてGUIでのインストールの準備を紹介します。
```bash
sudo apt install -y flatpak
sudo apt install -y gnome-software gnome-software-plugin-flatpak
sudo apt install -y qt5-flatpak-platformtheme
```
Snapパッケージも合わせてインストールしたい場合は、`gnome-software-plugin-snap`も追加してください。



## パーミッション（権限）の管理
### CUI
GUIで確認したほうが圧倒的に簡単で見やすいとは思いますが、CUIでの権限周りの操作を確認しましょう。
#### 権限の確認
```bash
flatpak info --show-permissions <app-ID>

# 例
flatpak info --show-permissions org.kde.kdenlive
[Context]
shared=network;ipc;
sockets=x11;wayland;pulseaudio;fallback-x11;
devices=dri;all;
filesystems=xdg-run/pipewire-0;xdg-config/kdeglobals:ro;host;

[Session Bus Policy]
com.canonical.AppMenu.Registrar=talk
org.kde.kconfig.notify=talk
org.kde.KGlobalSettings=talk

[System Bus Policy]
org.freedesktop.UDisks2=talk

[Environment]
QT_ENABLE_HIGHDPI_SCALING=1
TMPDIR=/var/tmp
FREI0R_PATH=/app/lib/frei0r-1
XDG_CURRENT_DESKTOP=kde
PACKAGE_TYPE=flatpak
LADSPA_PATH=/app/extensions/Plugins/ladspa:/app/lib/ladspa

```

主な引数は以下のとおりです。
```bash
flatpak info -h
Usage:
  flatpak info [OPTION…] NAME [BRANCH] - Get info about an installed app or runtime

Help Options:
  -h, --help                 Show help options

Application Options:
  --arch=ARCH                Arch to use
  --user                     Show user installations
  --system                   Show system-wide installations
  --installation=NAME        Show specific system-wide installations
  -r, --show-ref             Show ref
  -c, --show-commit          Show commit
  -o, --show-origin          Show origin
  -s, --show-size            Show size
  -m, --show-metadata        Show metadata
  --show-runtime             Show runtime
  --show-sdk                 Show sdk
  -M, --show-permissions     Show permissions
  --file-access=PATH         Query file access
  -e, --show-extensions      Show extensions
  -l, --show-location        Show location
  -v, --verbose              Show debug information, -vv for more detail
  --ostree-verbose           Show OSTree debug information
```

#### 権限の操作
`flatpak override`コマンドを用います。
主な引数は以下のとおりとなります。
```bash
flatpak override --help
Usage:
  flatpak override [OPTION…] [APP] - Override settings [for application]

Help Options:
  -h, --help                              Show help options
  --help-all                              Show all help options

Application Options:
  --user                                  Work on the user installation
  --system                                Work on the system-wide installation (default)
  --installation=NAME                     Work on a non-default system-wide installation
  --reset                                 Remove existing overrides
  --show                                  Show existing overrides
  -v, --verbose                           Show debug information, -vv for more detail
  --ostree-verbose                        Show OSTree debug information
  --share=SHARE                           Share with host
  --unshare=SHARE                         Unshare with host
  --socket=SOCKET                         Expose socket to app
  --nosocket=SOCKET                       Don't expose socket to app
  --device=DEVICE                         Expose device to app
  --nodevice=DEVICE                       Don't expose device to app
  --allow=FEATURE                         Allow feature
  --disallow=FEATURE                      Don't allow feature
  --filesystem=FILESYSTEM[:ro]            Expose filesystem to app (:ro for read-only)
  --nofilesystem=FILESYSTEM               Don't expose filesystem to app
  --env=VAR=VALUE                         Set environment variable
  --env-fd=FD                             Read environment variables in env -0 format from FD
  --unset-env=VAR                         Remove variable from environment
  --own-name=DBUS_NAME                    Allow app to own name on the session bus
  --talk-name=DBUS_NAME                   Allow app to talk to name on the session bus
  --no-talk-name=DBUS_NAME                Don't allow app to talk to name on the session bus
  --system-own-name=DBUS_NAME             Allow app to own name on the system bus
  --system-talk-name=DBUS_NAME            Allow app to talk to name on the system bus
  --system-no-talk-name=DBUS_NAME         Don't allow app to talk to name on the system bus
  --add-policy=SUBSYSTEM.KEY=VALUE        Add generic policy option
  --remove-policy=SUBSYSTEM.KEY=VALUE     Remove generic policy option
  --persist=FILENAME                      Persist home directory subpath
```

### GUI
[`Flatseal`というFlatpakアプリケーション](https://flathub.org/apps/com.github.tchx84.Flatseal)が使いやすいでしょう。
![](https://raw.githubusercontent.com/yKesamaru/How_to_install_Flatpak_app/master/assets/2024-05-23-14-12-34.png)
どこを変更したのか一目瞭然な上に、リセットする機能もついていておすすめです。
`Flatseal`をユーザーレベルでインストールした場合でもシステムレベルでインストールしたFlatpakアプリケーションの権限を変更できます（未検証）

以上です。ありがとうございました。

## 参考文献
- [Installation: Flathub](https://docs.flathub.org/docs/for-users/installation)
- [User vs system install: Flathub](https://docs.flathub.org/docs/for-users/user-vs-system-install)
- [Uninstallation: Flathub](https://docs.flathub.org/docs/for-users/uninstallation)