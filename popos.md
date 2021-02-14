# Pop!_OS を ThinkPad X230 Tablet / ThinkPad T495s に入れてみる

Pop Shellという名のタイル型ウィンドウマネージャがよくできているというので使ってみる。Ubuntuベースでカスタマイズを加えたものらしい。今回は 20.04 LTS を使ってみる。

## インストール

Pop!_OS 20.04 LTSのインストーラをダウンロードしてUSBメモリにRufasで書き込む。
少し時間がかかって途中途中で軽くフリーズしたようにも見えたがなんとか終了。
キーボードレイアウトは米国 (もしくはUS?) を選択。いろんなUSがあるが、何も修飾語がついていないもの(もしくは既定?)があるのでそれを選ぶこと。

## Capslock を Ctrl に

~~`/etc/default/keyboard` に以下を追記して再起動。~~

xkeysnail でやるので下記は不要。

```bash
XKBOPTIONS="ctrl:nocaps"
```

## xkeysnail

pip でインストール可能。

```bash
sudo apt install python3-pip
sudo pip3 install xkeysnail
```

デフォルトを参考に、以下のような `config.py` を作成し、`.config/xkeysnail/config.py` として保存。(**dotfiles化した**)

```py
# -*- coding: utf-8 -*-

import re
from xkeysnail.transform import *

# define timeout for multipurpose_modmap
define_timeout(1)

# [Global modemap] Change modifier keys as in xmodmap
define_modmap({
    Key.CAPSLOCK: Key.RIGHT_CTRL
})

# [Conditional modmap] Change modifier keys in certain applications
define_conditional_modmap(re.compile(r'Emacs'), {
    Key.RIGHT_CTRL: Key.ESC,
})

# [Multipurpose modmap] Give a key two meanings. A normal key when pressed and
# released, and a modifier key when held down with another key. See Xcape,
# Carabiner and caps2esc for ideas and concept.
define_multipurpose_modmap(
    # Enter is enter when pressed and released. Control when held down.
    {Key.ENTER: [Key.ENTER, Key.RIGHT_CTRL]}

    # Capslock is escape when pressed and released. Control when held down.
    # {Key.CAPSLOCK: [Key.ESC, Key.LEFT_CTRL]
    # To use this example, you can't remap capslock with define_modmap.
)

define_multipurpose_modmap({
    Key.MUHENKAN: [Key.MUHENKAN, Key.LEFT_SHIFT],
    Key.HENKAN: [Key.HENKAN, Key.RIGHT_SHIFT]
})

# ... 略 ...

sdefine_keymap(lambda wm_class: wm_class not in ("Emacs", "URxvt", "Gnome-terminal"), {

# ... 略 ...
```

オリジナルとの違いは以下の通り:
- CAPSLOCK を LEFT_CTRL ではなく RIGHT_CTRL にマップ
- 変換/無変換の機能追加
- Emacs バインディングを適用しない対象に `Gnome-terminal` を追加 (そうしないと `C-r` でのヒストリ検索ができない)

その後、

```
sudo xkeysnail .xkeysnail/config.py
```

してみると次のようなエラーが。

```
Xlib.error.DisplayConnectionError: Can't connect to display ":1": b'No protocol specified
```

[xkeysnailでキーリマップする - Qiita](https://qiita.com/miy4/items/dd0e2aec388138f803c5)によると

```
xhost +SI:localuser:root
```

することで xhost のアクセス制御に root が加わるので xhost が実行可能になるとのこと。たしかにその通りだった。

## xkeysnail の自動起動 (サービス化)

[Linuxで思い通りのキーマップを設定する【xkeysnail】 | Oh My Enter!](https://ohmyenter.com/how-to-install-and-autostart-xkeysnail/)

> うまく設定できたらsystemdに登録して自動起動するようにしておきます。xkeysnailはrootで実行する必要がありますが、どっちみち先程のxhostコマンドは一般ユーザーで実行する必要があるので、一般ユーザーで起動するようにしたほうがラクです。というわけで一般ユーザーに必要な権限だけ与えて、user環境のsystemdに登録することにします。
> 
> まずはグループを作成します。uinputというグループ名にします。
> 
> ```
> $ getent group uinput \# 一応すでに存在しないか見ておく
> $ sudo groupadd uinput
> ```
> 
> できましたら、inputと先程作成したuinputグループにユーザーを追加します。
> 
> ```
> $ sudo usermod -aG input,uinput あなたのユーザー名
> $ getent group input \# 確認
> $ getent group uinput \# 確認
> ```
> 
> あなたのユーザーがinput, uinputグループに追加されていることを確認しました。
> 続いてudev ruleを作成します。作成場所は/etc/udev/rules.dの中です。
> 
> ###### ◆/etc/udev/rules.d/70-input.rules
> 
> ```
> KERNEL=="event*", NAME="input/%k", MODE="660", GROUP="input"
> ```
> 
> ###### ◆/etc/udev/rules.d/70-uinput.rules
> 
> ```
> KERNEL=="uinput", GROUP="uinput"
> ```
> 
> 上記２ファイルを配置したら、一旦PCを再起動します。
> 
> 続いて、以下のファイルを`~/.config/systemd/user`に作成します。ディレクトリがなかったら作ってね。`あなたのユーザー名`は適宜変更してください。
> 
> ###### ◆~/.config/systemd/user/xkeysnail.service
> 
> ```
> [Unit]
> Description=xkeysnail
> 
> [Service]
> KillMode=process
> ExecStartPre=/usr/bin/xhost +SI:localuser:root
> ExecStart=/usr/local/bin/xkeysnail /home/あなたのユーザー名/.config/xkeysnail/config.py
> Type=simple
> Restart=always
> RestartSec=10s
> 
> \# たぶん :0 で問題ないと思いますが環境にもよります。\`echo $DISPLAY\`の値を設定してください
> Environment=DISPLAY=:0
> 
> [Install]
> WantedBy=default.target
> ```
> 
> 軽くポイントを説明しておくと、
> 
> -   ExecStartが実行コマンドです
> -   ExecStartPreが実行コマンド前に実行するコマンドです（さっきのxhostコマンド）
> -   落ちたりしたら自動的にリスタートするようにしていますが、そのリトライはRestartSecで10秒後を指定してます
> 
> といった感じです。

X230t でやったときは DISPLAY=:1 だった。(:0ではなかった)

> では次にキーマップ設定であるconfig.pyを`~/.config/xkeysnail/config.py`に配置してください。
> 
> ではサービスとして登録しましょう。
> 
> ```
> $ systemctl --user enable xkeysnail
> ```
> 
> 登録できたら起動してみます。
> 
> ```
> $ systemctl --user start xkeysnail
> ```
> 
> ちゃんと起動しているかステータスも見ておきます。
> 
> ```
> $ systemctl --user status xkeysnail
> ```
> 
> ● xkeysnail.service - xkeysnail
> 
> Loaded: loaded (/home/あなたのユーザー名/.config/systemd/user/xkeysnail.service; enabled; vendor preset: enabled)
> 
> Active: active (running) since Sat 2020\-04\-25 14:24:57 JST; 3h 6min ago
> 
> Process: 6409 ExecStartPre=/usr/bin/xhost +SI:localuser:root (code=exited, status=0/SUCCESS)
> 
> Main PID: 6410 (xkeysnail)
> 
> CGroup: /user.slice/user-1000.slice/user@1000.service/xkeysnail.service
> 
> └─6410 /usr/bin/python3 /usr/local/bin/xkeysnail /home/あなたのユーザー名/.config/xkeysnail/config.py
> 
> ● xkeysnail.service - xkeysnail Loaded: loaded (/home/あなたのユーザー名/.config/systemd/user/xkeysnail.service; enabled; vendor preset: enabled) Active: active (running) since Sat 2020-04-25 14:24:57 JST; 3h 6min ago Process: 6409 ExecStartPre=/usr/bin/xhost +SI:localuser:root (code=exited, status=0/SUCCESS) Main PID: 6410 (xkeysnail) CGroup: /user.slice/user-1000.slice/user@1000.service/xkeysnail.service └─6410 /usr/bin/python3 /usr/local/bin/xkeysnail /home/あなたのユーザー名/.config/xkeysnail/config.py
> 
> ● xkeysnail.service - xkeysnail
>      Loaded: loaded (/home/あなたのユーザー名/.config/systemd/user/xkeysnail.service; enabled; vendor preset: enabled)
>      Active: active (running) since Sat 2020-04-25 14:24:57 JST; 3h 6min ago
>     Process: 6409 ExecStartPre=/usr/bin/xhost +SI:localuser:root (code=exited, status=0/SUCCESS)
>    Main PID: 6410 (xkeysnail)
>      CGroup: /user.slice/user-1000.slice/user@1000.service/xkeysnail.service
>              └─6410 /usr/bin/python3 /usr/local/bin/xkeysnail /home/あなたのユーザー名/.config/xkeysnail/config.py
> 
> Active: active (running) になっていれば成功です。設定したキーマップを試してみましょう。
> 
> というわけで快適なLinux生活になりました。作者には強いありがたみ感じます。

## vscode インストール

`Pop!_Shop` でインストール可能だった。Flatpak版はローカルファイルの読み書きに制限があるようでコンソールがshだったりローカルのxclipが見つからずにPaste Image拡張が使えなかったりで不便だったので、deb版を使うことに。

## firefox で M-v などしたときにメニューにフォーカスがあたってしまう問題

`about:config` から `ui.key.menuAccessKeyFocuses` を `false` にしておく。さもないと M-v で page up しようとしたときにメニューバーにフォーカスがあたってしまって不都合。

参考: [https://apribase.net/2020/02/25/keybindings/](https://apribase.net/2020/02/25/keybindings/)

## vscode で M-v などしたときにメニューにフォーカスがあたってしまう問題

vscode で Alt キーがメニュー表示に取られてしまうのをなんとかしたい。vscode の設定を以下のように見直すことで解決。(参考: https://apribase.net/2020/02/25/keybindings/)

```json
"window.titleBarStyle": "custom",
"window.customMenuBarAltFocus": false
```

## workspace 切り替えを Super + Num で (**dotfiles化した**)

i3wm 的な動きにする。以下を一つひとつターミナル上で実行すると可能になった。

```shell-session
$ gsettings set org.gnome.mutter dynamic-workspaces false
$ gsettings set org.gnome.desktop.wm.preferences num-workspaces 10

$ gsettings set org.gnome.shell.keybindings switch-to-application-1 []
$ gsettings set org.gnome.shell.keybindings switch-to-application-2 []
$ gsettings set org.gnome.shell.keybindings switch-to-application-3 []
# ...
$ gsettings set org.gnome.shell.keybindings switch-to-application-9 []

$ gsettings set org.gnome.desktop.wm.keybindings switch-to-workspace-1  "['<Super>1']"
# ...
$ gsettings set org.gnome.desktop.wm.keybindings switch-to-workspace-10  "['<Super>0']"

$ gsettings set org.gnome.desktop.wm.keybindings move-to-workspace-1 "['<Super><Shift>1']"
# ...
$ gsettings set org.gnome.desktop.wm.keybindings move-to-workspace-10  "\['<Super><Shift>0'\]"
```

参考: 
- [Pop!\_OS Switch Workspaces with Super + Number – Stephen Cross](https://stephencross.com/2020/09/20/pop_os-switch-workspaces-with-super-number/)
- [Change workspaces with Super + number · Issue #142 · pop-os/shell · GitHub](https://github.com/pop-os/shell/issues/142)  
- [特に jabirali のコメント](https://github.com/pop-os/shell/issues/142#issuecomment-663079699)

## ホームディレクトリ内のサブディレクトリ名を英語に (**dotfiles化した**)

[Ubuntu でホームディレクトリ内のディレクトリ名を英語表記に | 雑廉堂の雑記帳](https://www.rough-and-cheap.jp/linux/ubuntu-change-xdg-directory-name/)

```shell-session
LANG=C xdg-user-dirs-gtk-update
```

ウィンドウが開いたら、`Don't ask me this again` にチェックを入れた上で、[Update Names] をクリック。

すでに使われているディレクトリは日本語名のものも残したまま新たに英字ディレクトリが作られるっぽい。例えば firefox がすでに `ダウンロード` フォルダを使っていたのでそうなった。この場合は firefox のダウンロード先設定も新しい英字フォルダ `Downloads` に変えてあげないといけないので注意。

## IME 設定

### 変換/無変換でオン/オフ

まず mozc の設定ツールを導入する。`アクティビティ` から `設定` で `言語サポート` を開くと、ご丁寧に mozc ツールが入っていないので導入を促されるのでそれに従う。
もしくは `sudo apt install mozc-utils-gui`。

するとアプリケーションメニューの中に `Mozc の設定` が現れる。これを押せばあとは直感的に分かる GUI (Google 日本語入力 と同じ) でキー設定すれば OK。(**dotfiles化した**)

### 英字や数字は常に半角

同じ GUI ツールにて、`入力補助` タブから設定可能。`変換前文字列` も半角にできるのが mozc というか Google 日本語入力の良いところだ。

## フォント設定

HackGenNerd と Spica Neue をダウンロード。
展開したフォント一式を `/usr/share/fonts/truetype/hackgen` と `/usr/share/fonts/truetype/spicaneue` にそれぞれコピーして

```shell-session
fc-cache -fv
```

すれば利用可能になる。

- VScode は `HackGen` にしてみた。あまり変わらないような。元が Noto Sans だから？
- Firefox は `Spica Neue P` にしてみた。文字間の幅が自然になって読みやすくなった感じ。

## システムフォントの変更

まず `Pop!_Shop` で `GNOME Tweaks` をインストールする。
そして `フォント`の欄で以下を変更:

| 項目 | Before | After |
|-|-|-|
| インターフェースのテキスト | Fira Sans Book (10) | |
| ドキュメントのテキスト | Roboto Slab Regular (11) | |
| 等幅テキスト | Fira Mono Regular (11) | |
| レガシーなウィンドウタイトル | Fira Sans SemiBold (10) | |

## 残課題

- [x] ubuntu のホームディレクトリを英語名に
- [x] IME の変換/無変換でのオン/オフ
- [x] フォント
	- [x] HackGenNerd
	- [x] Spica Neue
- [x] xkeysnail の config.py の `C-`を `RC-` に置換 @done(2021-02-07 09:41)
- [x] Obsidian @done(2021-02-13 23:30)
- [x] Xournal++ @done(2021-02-07 09:41)
- [x] トラポキャップ変えてみる @done(2021-02-07 15:04)
- [ ] テーマ
- [ ] 半透明

## T495s における予期せぬフリーズ

T495sでLinuxを使っていると、時折なんの前触れもなく突然のフリーズが発生する。キーもマウスも一切効かなくなる。唯一Fn+Spaceでキーボードのバックライトをつけたり消したりできる程度。`sudo radeontop`でGPUの状態表示をさせると10秒ほどで必ず再現できる。のでおそらくGPUの問題に間違いない。これは今回のPop!_OSに限った話ではなく、過去にもUbuntu20.04, Ubuntu20.10, Manjaro, Regolithでも同様だった。

以下で緩和された。完全に無くなったわけではないが、フリーズの頻度がだいぶ低くなった。

```shell-session
$ sudo su -
# echo "manual" > /sys/class/drm/card0/device/power_dpm_force_performance_level
```

