# Circle CI 2.0の基礎的な設定まとめてみた(GAE/Goのサンプル付き)
## 今回の記事について
[Circle CI2.0](https://circleci.com/?utm_source=Google&utm_medium=SEM&utm_campaign=Search%20Signup%20Branded&utm_content=circleci-search%20&gclid=EAIaIQobChMIy-XZgb7Y1gIVzI5oCh1rsgNjEAAYASAAEgIKB_D_BwE#2%20Branded)用の設定の基礎的な部分のメモ。

今回の記事は、単純な設定をCircle CI 2.0 で行うことを目的としているため、基礎的な設定のみを行い、[Workflow](https://circleci.com/docs/2.0/workflows/)等の設定は行わない。
また、サンプルとして、GAE/Goの設定を行う。
別にCircle CI1.0のことを知らなくても読めるはず。
Circle CI 1.0とCircle CI 2.0の機能的な違いなどについては、ここでは行わないので、以下を参考にするといいと思われる。
[CircleCI 2.0に移行して新機能を活用したらCIの実行時間が半分になった話 - クラウドワークス エンジニアブログ](http://engineer.crowdworks.jp/entry/2017/04/04/202719)

[CircleCI 2.0を使うようにするだけで、こんなに速くなるとは夢にも思わなかった！ | Tokyo Otaku Mode Blog](http://blog.otakumode.com/2017/06/09/cicle_ci_2/)

基本的な設定の項目はまとめるが、全部ではないので足りないところについては[公式ドキュメント](https://circleci.com/docs/2.0/configuration-reference/)を参照されたい。

[公式ドキュメント](https://circleci.com/docs/2.0/configuration-reference/)をかなり参考にさせていただいた。

## 実際の設定
まずは、実際の設定ファイルを見ていく方がわかりやすいと思うので、まずは設定ファイルを以下に記述する。

```yaml:circleci/.config.yml
version: 2 # バージョン2を指定する
jobs:
 build: # Goのbuildとテストを行う
  docker: # ベースとなるDocker imageを指定
   - image: circleci/golang:1.8 # Dockefileのパスを指定(Go1.8を指定)
  environment:
   TZ: /usr/share/zoneinfo/Asia/Tokyo # Time Zoneを指定
  working_directory: /home/circleci/go/src/project # コード実行場所 以下のstepsはworking_directoryで実行される
  steps: # ローカルでも必要なものはshell scriptと言う感じで行う
   - checkout # working_directoryにcheckout
   - run: # command lineのプログラムを発動させる
      name: Set PATH to .bashrc. # runには名前をつけることができる
      command: | # 実際のコマンド 複数行に場合は、 `|` をつける
       echo 'export PATH=$HOME/go/bin:$HOME/go_appengine:$PATH' >> $BASH_ENV  # $BASH_ENVはデフォルトで入っている
       source /home/circleci/.bashrc
   - run:
      name: Make GOPATH directory. # GOPATHを指定するディレクトリを作成
      command: mkdir -p $HOME/go/src
   - run:
      name: Set GOPATH to .bashrc. # .bashrcにGOPATHを追加
      command: |
       echo 'export GOPATH=$HOME/go' >> $BASH_ENV  # $BASH_ENVはデフォルトで入っている
       source /home/circleci/.bashrc
   - run:
      name: Install appengine sdk. # appengine SDKをインストール
      command: |
       wget https://storage.googleapis.com/appengine-sdks/featured/go_appengine_sdk_linux_amd64-1.9.58.zip
       unzip go_appengine_sdk_linux_amd64-1.9.58.zip -d $HOME
   - run:
      name: Execute setup. # ここでセットアップ用のshellを実行
      command: PATH/TO/setup.sh
   - run:
      name: Run Server Unit Tests. # ここでユニットテストを実行するshellを実行
      command: PATH/TO/test.sh
```

## Circle CI 2.0の各項目の説明

### 設定ファイル
Circle CI 2.0では、設定ファイルを従来の `circle.yml` ではなく、 `.circleci/config.yml` に記述することになっている。

### version
Circle CI 2.0を使用する場合は、versionと言うkeyを `.circleci/config.yml` の先頭に記述し、valueとして、2を指定する。

### jobs
mapで表現する。以降で説明する各jobの集合。

#### docker
dockerの設定を行う。
色々な項目があり、全部は記述しないので、その他の項目や詳しいことは[公式ドキュメント](https://circleci.com/docs/2.0/configuration-reference/#docker)を参照されたい。

##### image
Circle CIで使用するCustom Docker Image。
複数指定することが出来るが、最初に設定したDocker ImageがDocker executorを使用して、各jobを実行するのに使用される[Primary Container](https://circleci.com/docs/2.0/glossary/#primary-container)となる。

#### environment
各変数はここで宣言する。

##### TZ
Docker imageのTime Zoneは、environmentにTZと言う変数で指定する。

#### working_directory
後述するstepsを実行する場所を指定する。

#### branches
Git hubなどのbranchのルールを指定する。
onlyとignoreを指定することができる。
onlyとignoreが同一 `.circleci/config.yml` に存在する場合に関しては、ignoreのみが適用される。

##### only
onlyを宣言した後のbranch名のリストにあるbranchのみCircle CIが実行されるようになる。

##### ignore
ignoreを宣言した後のbranch名のリストにあるbranchを無視してCircle CIが実行されるようになる。

[Workflow](https://circleci.com/docs/2.0/workflows/)を使用する場合は、個別のjobにbranchを記述しない。

#### steps
実行されるstepのリスト。
key/valueのmapで表現する。
色々な項目があり、全部は記述しないので、その他の項目や詳しいことは[公式ドキュメント](https://circleci.com/docs/2.0/configuration-reference/#steps)を参照されたい。

##### checkout
設定されたpathにソースコードをcheck outする。
デフォルトでは、 `working_directory` がそのpathになる。

##### run
command lineプログラムを実行する。
以下のような方法が基本的な書き方。

```yaml
-run 
  command: コマンド
``` 
以下のように `name` としてcommandに名前をつけることもできる。

```yaml
-run 
　　　　name:
  command: コマンド
``` 

この場合は、CircleCI UIで表示される時にcommandがこの名前で表示される。
そうじゃない場合はフルのcommandが表示れるので、 `name` をつけた方がわかりやすくていいと思う。

複数行に渡る `command` は `|` をつけて、以下のように記述する。

```yaml
-run 
　　　　name:
  command: |
  コマンド1行目
  コマンド2行目
``` 

## その他の注意点
Circle CI 1.0では、 `environment` に指定すればCircle CIがよしなにやってくれていたが、Circle CI 2.0ではそうはいかない。
Circle CI 2.0では、自分で `command` で、`.bashrc` に設定する必要がある。
サンプルにある `BASH_ENV` というのはCircle CI 2.0がデフォルトでexportしているため、自前でenvironmentで宣言する必要はない。
以下を参考にさせていただいた。
[How to add a path to PATH in Circle 2.0? - CircleCI 2.0 / 2.0 Support - CircleCI Community Discussion](https://discuss.circleci.com/t/how-to-add-a-path-to-path-in-circle-2-0/11554)

## GAE/Goのサンプルの流れの説明
GAE/Goの流れは以下のようになっている。(stepsの部分のみ説明)
1. .bashrcにPATHを設定する
2. GOPATH用のディレクトリを作成する
3. 作成したGOPATH用のディレクトリを.bashrcにGOPATHとして設定
4. appengine SDKをインストールして、解凍
5. その他の設定用のshellを実行
6. テスト用のshellを実行

Circle CIから別の環境に乗り換えることも考慮して、appengine SDK以外の設定に関しては、shellで行うようにしている。

## 所感
書きやすくなった。

##　参考にさせていただいたサイト
[公式ドキュメント](https://circleci.com/docs/2.0/configuration-reference/)

[Circle CI2.0](https://circleci.com/?utm_source=Google&utm_medium=SEM&utm_campaign=Search%20Signup%20Branded&utm_content=circleci-search%20&gclid=EAIaIQobChMIy-XZgb7Y1gIVzI5oCh1rsgNjEAAYASAAEgIKB_D_BwE#2%20Branded)

[circleci2.0でおもにgoをCIする](https://gist.github.com/k-hoshina/3193afdbee67fef7faa7c5586c5311e8)

[CircleCI 2.0に移行して新機能を活用したらCIの実行時間が半分になった話 - クラウドワークス エンジニアブログ](http://engineer.crowdworks.jp/entry/2017/04/04/202719)

[How to add a path to PATH in Circle 2.0? - CircleCI 2.0 / 2.0 Support - CircleCI Community Discussion](https://discuss.circleci.com/t/how-to-add-a-path-to-path-in-circle-2-0/11554)

[CircleCI 2.0を使うようにするだけで、こんなに速くなるとは夢にも思わなかった！ | Tokyo Otaku Mode Blog](http://blog.otakumode.com/2017/06/09/cicle_ci_2/)