# Azure IoT Edge の作り方 
Docker ベースの [Azure IoT Edge](https://docs.microsoft.com/ja-jp/azure/iot-edge/) 対応の Edge Module の開発方法のTipsです。 

## いちから新規でカスタムモジュールを開発する場合 
### Visual Studio 2019 で開発 
[こちら](https://docs.microsoft.com/ja-jp/azure/iot-edge/how-to-visual-studio-develop-module)を参照の事 

### Visual Studio Code で開発 
[こちら](https://docs.microsoft.com/ja-jp/azure/iot-edge/how-to-vs-code-develop-module) を参照の事 

## 誰かが公開したサンプルコードを Edge Module 化したい！ 
こちらの方が実際には多い気がするので、詳細解説。  
※ 実行環境はありふれたRaspberry Pi（Raspbian）とします。  
開発の手順は以下の通り  
1. Docker のインターラクティブシェルで動作確認 
2. インターラクティブシェルで行った処理を元に DockerFile 化し動作確認 
3. DockerFileの作成 
4. IoT Edge 対応に改造して出来上がり 

### 1. Docker のインターラクティブシェルで動作確認 
このフェーズでは、公開されているサンプルのコードが、Docker コンテナで動作するか、Azure IoT Device SDK との相性に問題が無いか、の二点を確認します。  
手順は、以下の通り。  
1. インターラクティブシェル用にDocker Imageを作成し、シェルを起動する 
```sh
$ sudo docker create --name custommodule -it balenalib/raspberrypi3:latest /bin/bash
$ sudo docker start -ia customeodule
```
2. 基本的なコマンド群をインストール・セットアップする  
開始されたシェル上で以下のコマンドを実行する
```sh
$ sudo apt-get update
$ apt-get upgrade -y
$ apt-get dist-upgrade -y
```
3. サンプルに記載されたサンプルコードを実行するためのライブラリーやツールをインストールする。  
例えば、[OMRONの2jciebu-usb-raspberrypi](https://github.com/omron-devhub/2jciebu-usb-raspberrypi)を例にすると 
```sh
$ sudo apt-get install build-essential tk-dev libncurses5-dev libncursesw5-dev libreadline6-dev libdb5.3-dev libgdbm-dev libsqlite3-dev libssl-dev libbz2-dev libexpat1-dev liblzma-dev zlib1g-dev git 
$ apt-get install wget
$ wget https://www.python.org/ftp/python/3.6.5/Python-3.6.5.tar.xz
$ tar xf Python-3.6.5.tar.xz
$ cd Python-3.6.5
$ ./configure
$ make
$ sudo make altinstall
$ python3 -m pip install pyserial
$ git clone https://github.com/omron-devhub/2jciebu-usb-raspberrypi.git
``` 
と、一通り環境構築をインターラクティブシェル上で実行していく。途中でエラー等発生した場合は、その原因を考える。一番ありそうなのは実行しようとしたコマンドがインストールされていないというケース。なければインストールして再チャレンジを繰り返す。必要かつ十分なコマンド実行手順を見つけ出す。  

4. サンプルコードを実行してみる  
オムロンセンサーの場合は、[サンプルコードの動かし方](https://github.com/omron-devhub/2jciebu-usb-raspberrypi#usage) に従って実行してみる。
```sh
$ sudo modprobe ftdi_sio
$ sudo chmod 777 /sys/bus/usb-serial/drivers/ftdi_sio/new_id
$ sudo echo 0590 00d4 > /sys/bus/usb-serial/drivers/ftdi_sio/new_id
$ sudo python3 sample_2jciebu.py
```
このケースの場合、デバイスにアクセスできないとか、システムのディレクトリにアクセスできないとか、エラーが出る。このケースでは、最初の三行は、Docker上では実行せずシステムの初期化時にやるべきものだろうと判断し、Docker Imageには含めないこととする。  
※ この3つのコマンド実行は、インターラクティブシェル実行前、コンテナ実行前に行う必要があるので注意が必要。  
サンプルアプリの中からUSBデバイスへのアクセスと、/sys/bus/usb-serialのディレクトリへのアクセスを行うので、Dockerのコンテナーからもそれらにアクセスできなければならない。インターラクティブシェルの場合、Docker Imageを作成する時点でそれらを指定しなければならないので。アプリ実行して、エラーメッセージを見て、何を指定すればよいかを確定する。確認が終わったら一旦作業していシェルからExitし、Docker Imageを一旦削除する。  
インターラクティブシェル上で実行するアプリから参照するデバイスやSystemのディレクトリはそれぞれ、
- デバイスの場合、 → --device=...  
- ディレクトリの場合    -v  

で作成時にコマンドラインのオプションで指定しなければならないので、以下のコマンドでDocker Imageを新たに作り直す。
```sh
$ sudo docker create --device=/dev/ttyUSB0:/dev/ttyUSB0 -v /lib/modules/:/lib/modules -v /sys/bus/usb-serial/:/sys/bus/usb-serial/ --name custommodule -it balenalib/raspberrypi3:latest /bin/bash
$ sudo docker start -ia customeodule
```
1～3で確立したセットアップを再度くりかえし、4のサンプルを実行し動作を確認。動かない場合は四苦八苦して問題を解決する。その過程でインターラクティブシェル上で実行したことはすべて記録しておくこと。 

5. Azure IoT Device SDK のサンプル実行確認  
動かしたいサンプルの言語に合わせて適切な言語用SDKを選択し、インターラクティブシェル上でcloneする。OMRONセンサーの場合は、[Python向けSDK](https://github.com/azure/azure-iot-sdk-python)をインターラクティブシェル上でcloneし、このSDK用の開発環境をセットアップ（こちらも何を実行したかすべて記録しておくこと）し、
```sh
RUN python3 -m pip install pyserial
RUN python3 -m pip install azure-iothub-device-client
RUN sudo apt-get install libboost-python-dev
```
※Azure IoT Device SDK for Pythonの場合  
[device/sample/iothub_client_sample.py](https://github.com/Azure/azure-iot-sdk-python/blob/master/device/samples/iothub_client_sample.py) を実行し、Azure IoT Hubにメッセージが送信されることを確認。 

これで、このフェーズはすべて完了です。  

### 3. DockerFileの作成  
インターラクティブシェルで実行した内容を元にDockerFileを作成し、Docker Imageの構築を自動化し、自動化してできたイメージの動作を確認する。  
DockerFile作成の基本は、以下の通り。  
- 環境構築用に実行したコマンドの頭に RUN をつける。  
例えば、インターラクティブシェルで実行した最初のコマンドは、 
```go
RUN sudo apt-get update
RUN apt-get upgrade -y
RUN apt-get dist-upgrade -y
```
- ディレクトリの変更を伴うような場合は、&& \\ でつなぐ。  
```go
RUN wget https://www.python.org/ftp/python/3.6.5/Python-3.6.5.tar.xz && \
    tar xf Python-3.6.5.tar.xz && \
    ./configure && \
    make && \
    sudo make altinstall
```
- 実際に動かしたいアプリは、ENTRYPOINT [ command, 引数… ] と書く。その前に実行に必要なファイル群もコピーしておく  
```go
WORKDIR /app
COPY . /app
ENTRYPOINT ["python3", "sample_2jciebu-iotedge.py"]
```
※実行に必要なファイルをDocker buildするディレクトリに置いておくこと。作成したファイルの名前をcustommodule.Dockerfileとした場合は、 
```sh
$ sudo docker build -f custommodule.Dockerfile -t custommodule .
```
で、Imageがビルドされ、
```sh
$ sudo docker run --device=/dev/ttyUSB0:/dev/ttyUSB0 -v /sys/bus/usb-serial/:/sys/bus/usb-serial/ -t custommodule
```
で、Docker化したサンプルが実行される。  
※ 上の、--device=や、-vの指定は、オムロンセンサーの場合  
IoT Edge Module化したいサンプルと、そのサンプルを実行する環境で動作するAzure IoT Device SDKのサンプルの両方を、ぞれぞれ、確認する事。  

### 4. IoT Edge 対応に改造して出来上がり  
IoT Edge Module 化したいサンプルを、Azure IoT Device SDK のModule サンプルに組込む。各ModuleサンプルのIoT Edge Runtimeとの接続後に、IoT Edge化したいロジックを組込む、Module TwinのDesired Propertyの変更ハンドラーで起動するように組込む、Method Invocation で起動するなど、幾つかの方法があり、目的や用途に応じて選択すること。 
この過程で、IoT Edge Runtime との入出力が確定する。（入力と出力にそれぞれ名前をつける） 
Pythonの場合は[オムロンセンサーのサンプル](https://github.com/ms-iotkithol-jp/2jciebu-usb-raspberrypi/blob/master/sample_2jciebu-iotedge.py
)を参考にするとよいだろう。  
※このケースの場合は、Runtimeへの接続確立後に元々のサンプルを埋め込んでいる。  
※ Pythonの場合は、[Python向けSDKのサンプル](https://github.com/Azure/azure-iot-sdk-python/tree/master/device/samples)のiothub_client_cert.pyとiothub_client_args.pyが必要なので、改造したスクリプトファイルと同じディレクトリに置いておくこと。そこで、Dockerビルドすれば、そのファイルも併せてDocker Imageにコピーされる。 
IoT Edge対応モジュールをビルドするDockerfileが出来上がったら、githubで公開するなりなんなりで公開すると世のため人の為になるはず。  
その際、そのIoT Edge対応モジュールの入力と出力の名前をどこかに明記する事。加えて、実際にIoT Edge対応デバイスで動かすときに必要なコンフィギュレーション（--device=や―vの指定）をJSONで明記する事。 例に使っているオムロンセンサーのサンプルの場合は、 
```javascript
{
    "HostConfig": {
        "Binds": [
            "/lib/modules/:/lib/modules",
            "/sys/bus/usb-serial/:/sys/bus/usb-serial/"
        ],
        "Privileged": true,
        "Devices": [
            {
                "PathOnHost": "/dev/ttyUSB0",
                "PathInContainer": "/dev/ttyUSB0",
                "CgroupPermissions": "mrw"
            }
        ]
    }
}
```
こんな風になる。ディレクトリのバインドは、Bindsに、デバイスは、Devicesに追加する。  
以上、こんな感じで、[オムロンセンサーのIoT Edge化サンプル](https://github.com/ms-iotkithol-jp/2jciebu-usb-raspberrypi/tree/master/DockerExtension)は作っている。
開発したIoT Edge Module の汎用性が高ければ、ビルド済みのイメージをDocker Hubにおいて公開するとなおよろし。 

