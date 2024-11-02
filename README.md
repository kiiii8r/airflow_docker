# Airflow_Docker について
AirflowをDockerで簡単に構築するためのリポジトリです。

### Airflowのバージョン
Apache Airflow 2.1.2

### Dockerイメージ
Dockerイメージは、Docker Hubのapache/airflowを使用しています。

### Apache Airflow
Apache Airflowは、ワークフローのスケジューリングとモニタリングを行うためのプラットフォームです。Airflowは、Pythonで記述されたワークフローをDAG（Directed Acyclic Graph）として定義し、スケジュールに従って実行します。

### 前提条件
このリポジトリを使用するには、以下のソフトウェアがインストールされている必要があります。

- Docker
- Docker Compose

DockerとDocker Composeがインストールされていない場合は、公式サイトを参照してインストールしてください。

- Dockerのインストール
https://docs.docker.com/get-docker/

- Docker Composeのインストール
https://docs.docker.com/compose/install/


### Airflowの実行手順
以下の手順で、AirflowをDockerで構築します。

1. リポジトリのクローン
このリポジトリをクローンします。

2. Airflowの初期化（データベースやユーザー設定の初期化）
'''
docker-compose run airflow-webserver airflow db init
'''

3. 初期ユーザー（adminユーザー）を作成
docker-compose run airflow-webserver airflow users create \
    --username airflow \
    --firstname Airflow \
    --lastname Admin \
    --role Admin \
    --email admin@example.com \
    --password airflow

4. Airflowの起動
'''
docker-compose up -d
'''

5. AirflowのWebインターフェースにアクセス
Airflowが起動したら、ブラウザからhttp://localhost:8080にアクセスします。
上記で作成したユーザー名とパスワード（airflow / airflow）でログインできます。

6. 停止と再起動
Airflowを停止する場合は、以下のコマンドを実行します。
'''
docker-compose down
'''

再起動したい場合は、以下のコマンドを使用します。
'''
docker-compose up -d
'''

7. DAGの追加
DAG（Directed Acyclic Graph）は、Airflowでワークフローを定義するためのPythonスクリプトです。DAGを追加するには、dagsディレクトリにPythonスクリプトを配置します。


### 各フォルダとファイルついて
AirflowをDockerで起動する際の基本的なフォルダ構成は以下のようになります。

airflow-docker/
├── dags/                   # DAGファイルを保存するフォルダ
│   └── example_dag.py      # 例として配置するDAGファイル
├── logs/                   # Airflowのログを保存するフォルダ
├── plugins/                # カスタムプラグインを保存するフォルダ
├── .env                    # 環境変数を設定するファイル
└── docker-compose.yml      # Docker Composeファイル

dags/:
AirflowのDAGファイル（Pythonスクリプト）を保存するフォルダです。
Airflowはこのフォルダ内のDAGを自動的に認識して、スケジュールに従って実行します。
例としてexample_dag.pyを作成し、DAGのサンプルコードを記述できます。

logs/:
各タスクの実行ログを保存するためのフォルダです。
DockerのAirflowコンテナがこのフォルダにログを出力するように設定されているため、ログの管理に利用します。

plugins/:
カスタムのAirflowプラグイン（独自のオペレーター、フックなど）を保存するフォルダです。
必要に応じて独自のプラグインを作成して配置することで、Airflowに追加機能を提供できます。

.env:
環境変数ファイルで、AirflowやDocker Composeで必要な変数（例：ユーザーIDなど）を設定します。
AIRFLOW_UID=50000などの設定を行い、コンテナとローカルの権限問題を防ぐために使用します。

docker-compose.yml:
Docker Composeの設定ファイルで、AirflowやPostgres、Redisのコンテナ設定を管理します。
コンテナの起動順序や依存関係、ポート、ボリュームの設定などが記載されています。


### Airflowの構築手順

1. 準備
AirflowをDocker上で実行するためには、DockerとDocker Composeがインストールされている必要があります。

2. ディレクトリとファイルの準備
任意のディレクトリ（例: airflow-docker）を作成し、その中に必要なファイルを準備します。
以下の3つのファイルを作成します。

airflow-docker/
├── dags/                   # DAGファイルを保存するフォルダ
├── .env                    # 環境変数を設定するファイル
└── docker-compose.yml      # Docker Composeファイル


3. docker-compose.ymlの内容
以下の内容でdocker-compose.ymlファイルを作成します。
このファイルでは、Postgres、Redis、Airflow Webserver、Airflow Schedulerの4つのサービスを定義しています。

'''
version: '3'
services:
  postgres:
    image: postgres:13
    environment:
      POSTGRES_USER: airflow
      POSTGRES_PASSWORD: airflow
      POSTGRES_DB: airflow
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:latest

  airflow-webserver:
    image: apache/airflow:2.7.1
    depends_on:
      - postgres
      - redis
    environment:
      AIRFLOW__CORE__EXECUTOR: CeleryExecutor
      AIRFLOW__CORE__SQL_ALCHEMY_CONN: postgresql+psycopg2://airflow:airflow@postgres/airflow
      AIRFLOW__CELERY__BROKER_URL: redis://redis:6379/0
      AIRFLOW__CELERY__RESULT_BACKEND: db+postgresql://airflow:airflow@postgres/airflow
      AIRFLOW__CORE__FERNET_KEY: ''
      AIRFLOW__CORE__DAGS_ARE_PAUSED_AT_CREATION: 'true'
      AIRFLOW__CORE__LOAD_EXAMPLES: 'false'
      _AIRFLOW_WWW_USER_USERNAME: airflow
      _AIRFLOW_WWW_USER_PASSWORD: airflow
    volumes:
      - ./dags:/opt/airflow/dags
    ports:
      - "8080:8080"
    command: webserver

  airflow-scheduler:
    image: apache/airflow:2.7.1
    depends_on:
      - postgres
      - redis
    environment:
      AIRFLOW__CORE__EXECUTOR: CeleryExecutor
      AIRFLOW__CORE__SQL_ALCHEMY_CONN: postgresql+psycopg2://airflow:airflow@postgres/airflow
      AIRFLOW__CELERY__BROKER_URL: redis://redis:6379/0
      AIRFLOW__CELERY__RESULT_BACKEND: db+postgresql://airflow:airflow@postgres/airflow
    volumes:
      - ./dags:/opt/airflow/dags
    command: scheduler

volumes:
  postgres_data:
'''


4. 環境変数ファイル（.env）
.envファイルにはAirflowの環境変数を設定します。以下のように作成します。
'''
AIRFLOW_UID=50000
'''

※ Airflow UIDは、Airflowコンテナ内のユーザーIDです。ローカルユーザーと一致させることで、ファイルのアクセス権限に問題が発生しにくくなります。50000は一般的に使用されるUIDですが、環境に応じて変更してください。

5. DAGファイルの配置
AirflowのDAGファイルを配置するためのdagsフォルダを作成します。このフォルダにDAGファイル（Pythonスクリプト）を配置すると、Airflowが自動的に認識して実行可能になります。

6. Airflowの初期化と起動
以下のコマンドを使って、Airflow環境を初期化し、起動します。
'''
### ディレクトリをairflow-dockerに移動
cd airflow-docker
'''

### Airflowの初期化（データベースやユーザー設定の初期化）
'''
docker-compose run airflow-webserver airflow db init
'''

### 初期ユーザー（adminユーザー）を作成
'''
docker-compose run airflow-webserver airflow users create \
    --username airflow \
    --firstname Airflow \
    --lastname Admin \
    --role Admin \
    --email admin@example.com \
    --password airflow
'''

### Airflowの起動
'''
docker-compose up -d
'''

7. AirflowのWebインターフェースにアクセス
Airflowが起動したら、ブラウザからhttp://localhost:8080にアクセスします。
上記で作成したユーザー名とパスワード（airflow / airflow）でログインできます。

8. 停止と再起動
Airflowを停止する場合は、以下のコマンドを実行します。

'''
docker-compose down
'''