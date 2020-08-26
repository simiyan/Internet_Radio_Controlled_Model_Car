# Internet_Radio_Controlled_Model_Car
インターネットラジコンプロジェクト

# raspi_remote_car@rebuild

## 1.準備物

### 1.1 前提条件

- 操作端末はWindows 10

### 1.2 ハードウェア

#### 1.2.1 Raspberry pi 関連

- microSD Reader
- Raspberry pi 4 本体
- USB - AC電源変換
- USBtypeCケーブル
- microSD 32GB

### 1.3 OS

- Raspbian Buster

### 1.4 ソフトウェア

1. DD for Windows

## 2.環境の準備

### 2.1 imageファイルの準備

- 公式サイトからimageファイルをダウンロードする  
[Raspbian](https://www.raspberrypi.org/downloads/raspbian/)  
Raspbian Buster Liteの「Download ZIP」からDownloadする

### 2.2 Raspbianのインストール

目標：OSインストールしてsshで遠隔操作可能なところまで

1. microSD ReaderでmicroSDを読み込む
2. DD for Windowsを管理者権限で起動
3. 「ディスクを選択」で1.でloadしたドライブを選択
4. 「ファイルを選択」で2.1でダウンロードしたファイルを選択  
    ※ 読込むファイル形式をAll Filesに変更する
5. 「書込」ボタンでmicroSDにimageの書込みを開始する  
    ※ 数度の警告は出るが全てOKでよい
6. 書込みが終われば、読込まれるディスクのboot直下に下記のファイルを作成する
    - ssh ※空ファイル/拡張子なし
    - wpa_supplicant.conf  
        ※内容は以下  
        ※2.4Ghz/5GHz帯のどちらでもOK  
        ※設定値は操作端末と同じAPの設定値であること

    ``` wpa_supplicant.conf
    [wpa_supplicant.conf]

    country=JP
    ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
    update_config=1
    network={
        ssid="SSID名"
        psk="暗号化キー"
    }
    ```

7. microSDをRaspberry pi に差す。
8. USBtypeCケーブルとUSB - AC電源変換を使って起動する。
9. Raspberrypiに振られたIPアドレスを探す  
    - コマンドプロンプトで `arp -a` を実行して下記を探す。

    ``` cmd
    rem 192.168.0.1 - 255 までpingを打つ
    rem networkが違う場合、適宜変更する
    for /l %i in (0,1,255) do ping -w 1 -n 1 192.168.0.%i
    rem arpの結果からraspberrypiのOUIを探す
    arp -a | findstr b8-27-eb
    arp -a | findstr dc-a6-32
    ````

    ⇒ 対応するIPアドレスがraspberry piのIPアドレスとなる。

10. TeraTermでわかったIPアドレスにsshで接続する
    - ログイン情報：pi/raspberry

### 2.3 共通OS準備と確認

目標：dockerマシンの起動ができるところまで

1. インターネット接続を確認
   - `ping google.com`で確認  
    ⇒ 応答があればOK
2. OSとyumのupdate
   - `sudo apt update && sudo apt -y dist-upgrade && sudo apt -y autoremove && sudo apt autoclean`
3. 便利toolをinstall  
    - `sudo apt install vim tmux tcpdump git ufw`  
4. ストレージの拡張
    - `sudo raspi-config  --expand-rootfs`
5. dockerのinstall  
    ※homeディレクトリで  
    `mkdir -p install/docker`  
    `cd install/docker`  
    curl -fsSL https://get.docker.com -o get-docker.sh  
    sudo sh get-docker.sh  
    sudo usermod -aG docker pi  
    sudo systemctl enable docker  
    git clone https://github.com/docker/compose.git  
    cd compose  
    git checkout 1.25.3  
    sudo ./script/build/linux  
    cd dist  
    ./docker-compose-Linux-armv7l version  
    sudo cp docker-compose-Linux-armv7l /usr/local/bin/docker-compose  
    sudo chown root:root /usr/local/bin/docker-compose  
    sudo chmod 755 /usr/local/bin/docker-compose

6. streamning serverの導入
   - docker pull openhorizon/mjpg-streamer-pi3:latest
      ※ streaming server導入済みのdocker imageをpull
   - docker image ls  
      → "openhorizon/mjpg-streamer-pi3"があればよい

<!-- 
1. dockerにdebianイメージの導入  \
   docker pull debian  \
   docker images  )
-->

### 2.4 mjpeg-streamer

目標:camera moduleを通してstreamingできること

1. camera moduleの有効化  
    - `sudo raspi-config`
    - `5.Interfacing Options`
    - `P1 Camera`⇒yes  
    ⇒ rebootが走る

2. streaming serviceの起動と自動起動設定
   - docker container run  -it --privileged --name "car_streaming" -d -p 9000:9000 openhorizon/mjpg-streamer-pi3 ./mjpg_streamer -i "./input_raspicam.so -fps 15 -q 50 -x 640 -y 480" -o "./output_http.so -p 9000 -w ./www" /bin/bash
        - car_streaming：任意のcontainer名
        - /bin/bash：bashを使って起動
        - mjpg_streamer以下：raspicamの入力をWEBサーバの9000portに640x480で出力
        - hostの9000をdocker 9000 にport fowardする
   - docker update --restart=always car_streaming  
        - バックグラウンドで起動

<!-- 
1. docker baseへattach  
   `docker container attach remote_car_base`  

2. mjpg-streamerのinstall  
   - `apt install git make cmake -y`
   - `cd`
   - `mkdir work`
   - `cd work`
   - `git clone https://github.com/jacksonliam/mjpg-streamer.git mjpg-streamer`
   - `cd mjpg-streamer/mjpg-streamer-experimental/`
   - `mkdir /var/www`
   - `make`
   - `cp -r mjpg-streamer/mjpg-streamer-experimental/ /var/www/mjpg-streamer`
   - `cd /var/www/mjpg-streamer/`
   - a
-->
99参考サイト

- [RaspberryPi Raspbian ヘッドレスインストール（Buster編）](https://qiita.com/nori-dev-akg/items/38c2dfb108edb0d73908)
- [Raspberry Piにdockerとdocker-composeを入れてみた](https://qiita.com/hoshi621/items/7906274326ef3013a73d)  
- [いまさらだけどDockerに入門したので分かりやすくまとめてみた](https://qiita.com/gold-kou/items/44860fbda1a34a001fc1)
- [コンテナに外部からアクセス（ポートフォワード）](https://qiita.com/tifa2chan/items/a58e34019d4f10097a4d)