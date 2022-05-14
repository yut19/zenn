---
title: "isucon事はじめ" # 記事のタイトル
emoji: "💺" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["isucon"] # タグ。["markdown", "rust", "aws"]のように指定する
published: false # 公開設定（falseにすると下書き）
---

# isucon 事はじめ

未回収のキーワード

- LimitNoFile
- LimitNPROC

1. ベンチマークを実行してスコアを確認する
2. アプリケーションのチューニングする
3. 1-2 の繰り返し

各課題ごとにベンチマーカーの採点基準が異なる。
採点基準に関しては各マニュアルを熟読する。

以下の記事をトレースしてみる。
[ISUCON11 予選問題実践攻略法](https://isucon.net/archives/56082639.html)

- 環境構築
  [公開されている AMI](https://ap-northeast-1.console.aws.amazon.com/ec2/home?region=ap-northeast-1#ImageDetails:imageId=ami-0796be4f4814fc3d5)をもとに EC2 インスタンスに環境を構築する。
  その他の過去問の環境構築用の AMI も[matsuu](https://github.com/matsuu)/[aws-isucon](https://github.com/matsuu/aws-isucon)で公開されている。

ディレクトリ構成

```
.
├── bench
│   ├── Makefile
│   ├── README.md
│   ├── bench
│   ├── data
│   ├── gen
│   ├── go.mod
│   ├── go.sum
│   ├── images
│   ├── key
│   ├── logger
│   ├── main.go
│   ├── model
│   ├── random
│   ├── scenario
│   └── service
├── env.sh(アプリに渡す環境変数)
├── go
│   └── pkg
├── local
│   ├── go
│   ├── node
│   ├── perl
│   ├── php
│   ├── python
│   └── ruby
└── webapp(アプリの各言語の実装)
    ├── NoImage.jpg
    ├── ec256-public.pem
    ├── frontend
    ├── go
    ├── nodejs
    ├── perl
    ├── php
    ├── public(静的コンテンツ)
    ├── python
    ├── ruby
    ├── rust
    └── sql(SQLの初期化ファイルなど)
```

DB の確認

```
$ mysql --version
mysql  Ver 15.1 Distrib 10.3.34-MariaDB, for debian-linux-gnu (x86_64) using readline 5.2
```

env.sh に設定されている環境変数の情報が格納されている。

```env.sh
MYSQL_HOST="127.0.0.1"
MYSQL_USER=isucon
://washufes-osaka03.peatix.com/event/3238632/view?utm_campaign=follow-organizer&utm_medium=email&utm_source=event%3A3238632&kme=reco-clk&km_reco-str=PodMembe://washufes-osaka03.peatix.com/event/3238632/view?utm_campaign=follow-organizer&utm_medium=email&utm_source=event%3A3238632&kme=reco-clk&km_reco-str=PodMember
MYSQL_DBNAME=isucondition
MYSQL_PASS=isucon
POST_ISUCONDITION_TARGET_BASE_URL="https://isucondition-1.t.isucon.dev"
```

アクセスログ解析ツール alp を install する
asdf でも簡単にインストールできる。
https://github.com/tkuchiki/alp

記事内で紹介されている解析・モニタリングツール

- top, htop, dstat
- alp, kataribe
- slow-query-log
- stackprof, pprof
- NewRelic, Splunk

nginx のアクセルログファイルを確認する。
nginx のログファイル: /var/log/nginx/access.log
nginx の設定ファイル: /etc/nginx/nginx.conf

アクセスログを alp で解析できるように ltsv 形式でログを書き出すようにする。

```nginx.conf
http {
	include       /etc/nginx/mime.types;
	default_type  application/octet-stream;

	log_format ltsv '$remote_addr - $remote_user [$time_local] "$request" '
									'$status $body_bytes_sent "$http_referer" '
									'"$http_user_agent" "$http_x_forwarded_for"';

	access_log  /var/log/nginx/access.log ltsv;

	sendfile        on;
	#tcp_nopush     on;

	keepalive_timeout  65;

	#gzip  on;

	include /etc/nginx/conf.d/*.conf;
	include /etc/nginx/sites-enabled/*.conf;
}
```

ベンチマークの実行

```
sudo -i -u isucon
cd bench
# 本番同様にnginx(https)へアクセスを向けたい場合
./bench -all-addresses 127.0.0.11 -target 127.0.0.11:443 -tls -jia-service-url http://127.0.0.1:4999
# isucondition(3000)へ直接アクセスを向けたい場合
./bench -all-addresses 127.0.0.11 -target 127.0.0.11:3000 -jia-service-url http://127.0.0.1:4999
```

alp でアクセスログを見てみる

```
$ sudo cat /var/log/nginx/access.log | alp ltsv
+-------+-----+------+-----+-----+-----+--------+-----------------------------------------------------+-------+-------+--------+-------+-------+-------+-------+--------+-----------+-----------+-----------+-----------+
| COUNT | 1XX | 2XX  | 3XX | 4XX | 5XX | METHOD |                         URI                         |  MIN  |  MAX  |  SUM   |  AVG  |  P90  |  P95  |  P99  | STDDEV | MIN(BODY) | MAX(BODY) | SUM(BODY) | AVG(BODY) |
+-------+-----+------+-----+-----+-----+--------+-----------------------------------------------------+-------+-------+--------+-------+-------+-------+-------+--------+-----------+-----------+-----------+-----------+
|   808 |   0 |  715 |   0 |  93 |   0 | POST   | /api/condition/76634cc4-be22-49a8-b80a-f31983f4cb24 | 0.004 | 0.120 | 42.444 | 0.053 | 0.100 | 0.100 | 0.104 |  0.026 |     0.000 |     0.000 |     0.000 |     0.000 |
|   907 |   0 |  792 |   0 | 115 |   0 | POST   | /api/condition/d0f40529-ab5a-471c-a722-31ebbde5dbf0 | 0.008 | 0.104 | 47.620 | 0.053 | 0.100 | 0.100 | 0.104 |  0.025 |     0.000 |     0.000 |     0.000 |     0.000 |
|   908 |   0 |  801 |   0 | 107 |   0 | POST   | /api/condition/0cd10b50-d3d1-461e-960f-3bed587a349c | 0.004 | 0.136 | 47.208 | 0.052 | 0.100 | 0.100 | 0.104 |  0.025 |     0.000 |     0.000 |     0.000 |     0.000 |
|   971 |   0 |  865 |   0 | 106 |   0 | POST   | /api/condition/df9746b8-6285-4644-a860-95b7a535d0f9 | 0.004 | 0.120 | 49.224 | 0.051 | 0.096 | 0.100 | 0.104 |  0.025 |     0.000 |     0.000 |     0.000 |     0.000 |
|   999 |   0 |  873 |   0 | 126 |   0 | POST   | /api/condition/e62e3281-4bda-4da3-ab44-b5cf2790a4b7 | 0.000 | 0.108 | 51.352 | 0.051 | 0.100 | 0.100 | 0.104 |  0.025 |     0.000 |     0.000 |     0.000 |     0.000 |
|  1023 |   0 |  907 |   0 | 116 |   0 | POST   | /api/condition/74a65e9a-dc33-45dd-b807-dfee9d35a2b1 | 0.008 | 0.116 | 51.064 | 0.050 | 0.100 | 0.100 | 0.104 |  0.025 |     0.000 |     0.000 |     0.000 |     0.000 |
|  1028 |   0 |  902 |   0 | 126 |   0 | POST   | /api/condition/9f2d23e5-0d4d-43c3-bbff-a6eb0d914018 | 0.008 | 0.108 | 51.980 | 0.051 | 0.100 | 0.100 | 0.104 |  0.025 |     0.000 |     0.000 |     0.000 |     0.000 |
|  1029 |   0 |  913 |   0 | 116 |   0 | POST   | /api/condition/807a1792-7081-4d3a-85d0-f22eb547fbbf | 0.004 | 0.116 | 51.708 | 0.050 | 0.100 | 0.100 | 0.104 |  0.025 |     0.000 |     0.000 |     0.000 |     0.000 |
|  1052 |   0 |  920 |   0 | 132 |   0 | POST   | /api/condition/80311e24-0be9-45d9-b512-1a6ba87999b3 | 0.004 | 0.120 | 52.392 | 0.050 | 0.100 | 0.100 | 0.104 |  0.027 |     0.000 |    14.000 |    14.000 |     0.013 |
|  1054 |   0 |  939 |   0 | 115 |   0 | POST   | /api/condition/9492fa67-9571-448f-bdc9-8b963277acbb | 0.012 | 0.108 | 52.244 | 0.050 | 0.096 | 0.100 | 0.104 |  0.025 |     0.000 |     0.000 |     0.000 |     0.000 |
|  1057 |   0 |  924 |   0 | 133 |   0 | POST   | /api/condition/594cd00f-b8f7-4f1d-b01c-2eb54b398843 | 0.004 | 0.116 | 53.500 | 0.051 | 0.100 | 0.100 | 0.104 |  0.026 |     0.000 |     0.000 |     0.000 |     0.000 |
|  1060 |   0 |  926 |   0 | 134 |   0 | POST   | /api/condition/49eaefa6-e057-429d-939f-ee8f34569d93 | 0.012 | 0.112 | 53.736 | 0.051 | 0.100 | 0.100 | 0.104 |  0.026 |     0.000 |     0.000 |     0.000 |     0.000 |
|  1063 |   0 |  942 |   0 | 121 |   0 | POST   | /api/condition/373e098e-1c77-4a45-bc62-b8807c79e902 | 0.008 | 0.112 | 52.764 | 0.050 | 0.096 | 0.100 | 0.104 |  0.025 |     0.000 |     0.000 |     0.000 |     0.000 |
|  1070 |   0 |  930 |   0 | 140 |   0 | POST   | /api/condition/3fd3e4de-7d22-45ca-bbc3-6419dc7ebbad | 0.016 | 0.120 | 53.064 | 0.050 | 0.100 | 0.100 | 0.104 |  0.027 |     0.000 |     0.000 |     0.000 |     0.000 |
|  1071 |   0 |  944 |   0 | 127 |   0 | POST   | /api/condition/418f4cc7-eb5d-44d2-a88a-dbf4da9521d3 | 0.008 | 0.112 | 53.192 | 0.050 | 0.100 | 0.100 | 0.104 |  0.026 |     0.000 |     0.000 |     0.000 |     0.000 |
|  1071 |   0 |  932 |   0 | 139 |   0 | POST   | /api/condition/ec9ebdda-c3c4-4bd4-85d9-ac67ec57a281 | 0.016 | 0.116 | 53.588 | 0.050 | 0.100 | 0.100 | 0.104 |  0.027 |     0.000 |     0.000 |     0.000 |     0.000 |
|  1072 |   0 |  949 |   0 | 123 |   0 | POST   | /api/condition/84ff011c-5164-4718-a4cd-9f717dac9691 | 0.004 | 0.116 | 52.664 | 0.049 | 0.096 | 0.100 | 0.104 |  0.026 |     0.000 |     0.000 |     0.000 |     0.000 |
|  1073 |   0 |  945 |   0 | 128 |   0 | POST   | /api/condition/f39d9d70-63be-4bc9-b7e9-a12283b98c83 | 0.004 | 0.116 | 52.840 | 0.049 | 0.100 | 0.100 | 0.104 |  0.026 |     0.000 |     0.000 |     0.000 |     0.000 |
|  1075 |   0 |  953 |   0 | 122 |   0 | POST   | /api/condition/e215e992-e36b-42d6-961e-8427f5ef8104 | 0.008 | 0.116 | 53.272 | 0.050 | 0.100 | 0.100 | 0.104 |  0.025 |     0.000 |     0.000 |     0.000 |     0.000 |
|  1076 |   0 |  968 |   0 | 108 |   0 | POST   | /api/condition/3ab0dfbb-8a3e-458c-b083-d0d8dcdcf51c | 0.008 | 0.136 | 52.844 | 0.049 | 0.096 | 0.100 | 0.104 |  0.025 |     0.000 |     0.000 |     0.000 |     0.000 |
|  1077 |   0 |  961 |   0 | 116 |   0 | POST   | /api/condition/0698d691-0b9e-4321-a0e7-34b612b004b8 | 0.004 | 0.112 | 52.848 | 0.049 | 0.096 | 0.100 | 0.104 |  0.025 |     0.000 |     0.000 |     0.000 |     0.000 |
|  1079 |   0 |  956 |   0 | 123 |   0 | POST   | /api/condition/cf48157d-0849-4935-91c8-455cbcfa02dc | 0.004 | 0.112 | 52.924 | 0.049 | 0.096 | 0.100 | 0.104 |  0.026 |     0.000 |     0.000 |     0.000 |     0.000 |
|  1081 |   0 |  946 |   0 | 135 |   0 | POST   | /api/condition/2842a894-3e2f-4fcf-aa0e-5deb3e145ffc | 0.012 | 0.112 | 53.192 | 0.049 | 0.100 | 0.100 | 0.104 |  0.027 |     0.000 |     0.000 |     0.000 |     0.000 |
|  1082 |   0 |  972 |   0 | 110 |   0 | POST   | /api/condition/6ea18c94-28af-4256-b95f-b5fca7107978 | 0.004 | 0.112 | 53.192 | 0.049 | 0.096 | 0.100 | 0.104 |  0.025 |     0.000 |     0.000 |     0.000 |     0.000 |
|  1083 |   0 |  950 |   0 | 133 |   0 | POST   | /api/condition/318f41bf-4f86-4e62-89b1-af44c94ceccc | 0.008 | 0.128 | 53.156 | 0.049 | 0.100 | 0.100 | 0.104 |  0.026 |     0.000 |     0.000 |     0.000 |     0.000 |
|  1086 |   0 |  966 |   0 | 120 |   0 | POST   | /api/condition/35e54e02-3b40-4cdd-9460-646ec83a7f8f | 0.012 | 0.132 | 53.320 | 0.049 | 0.100 | 0.100 | 0.104 |  0.026 |     0.000 |     0.000 |     0.000 |     0.000 |
|  1086 |   0 |  981 |   0 | 105 |   0 | POST   | /api/condition/b1bd5ce9-57ea-4227-bcac-0e94cfbb067a | 0.004 | 0.104 | 52.060 | 0.048 | 0.092 | 0.100 | 0.104 |  0.025 |     0.000 |     0.000 |     0.000 |     0.000 |
|  1088 |   0 |  958 |   0 | 130 |   0 | POST   | /api/condition/62befe75-3c7d-4ab9-9ab5-c2ee98358910 | 0.012 | 0.112 | 53.244 | 0.049 | 0.100 | 0.100 | 0.104 |  0.026 |     0.000 |     0.000 |     0.000 |     0.000 |
|  1088 |   0 |  969 |   0 | 119 |   0 | POST   | /api/condition/0d4ceadf-6ea8-4855-8b66-f9b54bf0520c | 0.012 | 0.120 | 53.284 | 0.049 | 0.100 | 0.100 | 0.104 |  0.026 |     0.000 |     0.000 |     0.000 |     0.000 |
|  1088 |   0 |  961 |   0 | 127 |   0 | POST   | /api/condition/679d9bd7-dba2-4c1b-9123-11502d15f49a | 0.012 | 0.120 | 52.924 | 0.049 | 0.100 | 0.100 | 0.104 |  0.026 |     0.000 |     0.000 |     0.000 |     0.000 |
|  1090 |   0 |  975 |   0 | 115 |   0 | POST   | /api/condition/d8787664-30b0-4e18-a7cd-fcf20483c5d9 | 0.012 | 0.120 | 53.116 | 0.049 | 0.096 | 0.100 | 0.104 |  0.026 |     0.000 |     0.000 |     0.000 |     0.000 |
|  1090 |   0 |  970 |   0 | 120 |   0 | POST   | /api/condition/63f97f45-c65e-4202-bbb1-23fc8d5a37ba | 0.020 | 0.132 | 52.868 | 0.049 | 0.100 | 0.100 | 0.104 |  0.026 |     0.000 |     0.000 |     0.000 |     0.000 |
|  1090 |   0 |  955 |   0 | 135 |   0 | POST   | /api/condition/c422a60e-8c9f-4bd9-8e9a-9cbe82127ec0 | 0.012 | 0.120 | 53.900 | 0.049 | 0.100 | 0.100 | 0.104 |  0.026 |     0.000 |     0.000 |     0.000 |     0.000 |
|  1090 |   0 |  971 |   0 | 119 |   0 | POST   | /api/condition/8b982297-e4b3-49b5-8988-f2a52744c2f6 | 0.004 | 0.108 | 52.800 | 0.048 | 0.100 | 0.100 | 0.104 |  0.026 |     0.000 |     0.000 |     0.000 |     0.000 |
|  1091 |   0 |  965 |   0 | 126 |   0 | POST   | /api/condition/6c474a3f-1cd5-47e1-82bc-8bd735e89851 | 0.008 | 0.108 | 52.704 | 0.048 | 0.100 | 0.100 | 0.104 |  0.026 |     0.000 |     0.000 |     0.000 |     0.000 |
|  1092 |   0 |  969 |   0 | 123 |   0 | POST   | /api/condition/403df6a0-f40a-4fef-914e-06784efe39e0 | 0.020 | 0.120 | 53.072 | 0.049 | 0.096 | 0.100 | 0.104 |  0.026 |     0.000 |     0.000 |     0.000 |     0.000 |
|  1092 |   0 |  982 |   0 | 110 |   0 | POST   | /api/condition/58adf9bc-7b05-4a67-bd7e-0943fee45dfc | 0.004 | 0.120 | 52.508 | 0.048 | 0.096 | 0.100 | 0.104 |  0.026 |     0.000 |     0.000 |     0.000 |     0.000 |
|  1093 |   0 |  972 |   0 | 121 |   0 | POST   | /api/condition/65c8496b-9cd5-493c-a3a7-3e22d0b2893b | 0.016 | 0.116 | 53.164 | 0.049 | 0.096 | 0.100 | 0.104 |  0.026 |     0.000 |     0.000 |     0.000 |     0.000 |
|  1094 |   0 |  961 |   0 | 133 |   0 | POST   | /api/condition/dedab46a-167c-415b-b5a2-b8ddf5671407 | 0.008 | 0.116 | 52.960 | 0.048 | 0.100 | 0.100 | 0.104 |  0.027 |     0.000 |     0.000 |     0.000 |     0.000 |
|  1094 |   0 |  969 |   0 | 125 |   0 | POST   | /api/condition/b9e1bf8f-7d2d-4485-93ce-24e57696f193 | 0.004 | 0.116 | 53.140 | 0.049 | 0.100 | 0.100 | 0.104 |  0.026 |     0.000 |     0.000 |     0.000 |     0.000 |
|  1094 |   0 |  965 |   0 | 129 |   0 | POST   | /api/condition/60be616c-3bc5-454a-9b5a-e21d6c07c44a | 0.004 | 0.108 | 53.328 | 0.049 | 0.100 | 0.100 | 0.104 |  0.027 |     0.000 |     0.000 |     0.000 |     0.000 |
|  1098 |   0 |  983 |   0 | 115 |   0 | POST   | /api/condition/f11792f4-7591-4f8d-b7e5-c717458efc69 | 0.004 | 0.108 | 53.092 | 0.048 | 0.096 | 0.100 | 0.104 |  0.026 |     0.000 |     0.000 |     0.000 |     0.000 |
|  1102 |   0 |  986 |   0 | 116 |   0 | POST   | /api/condition/d51a5fde-ed3d-430b-a922-7979949507dd | 0.004 | 0.128 | 53.912 | 0.049 | 0.096 | 0.100 | 0.104 |  0.026 |     0.000 |     0.000 |     0.000 |     0.000 |
|  1104 |   0 |  998 |   0 | 106 |   0 | POST   | /api/condition/34051482-91e5-4fcb-859a-0cd3af3ad254 | 0.004 | 0.108 | 53.076 | 0.048 | 0.096 | 0.100 | 0.104 |  0.025 |     0.000 |     0.000 |     0.000 |     0.000 |
|  1105 |   0 |  988 |   0 | 117 |   0 | POST   | /api/condition/86be6789-9dbc-4d9e-bf6b-9f967bf5ca28 | 0.004 | 0.108 | 53.080 | 0.048 | 0.096 | 0.100 | 0.104 |  0.026 |     0.000 |     0.000 |     0.000 |     0.000 |
|  1106 |   0 | 1003 |   0 | 103 |   0 | POST   | /api/condition/f1c760c6-1c98-4228-82a0-c6322b0ac724 | 0.004 | 0.108 | 52.756 | 0.048 | 0.084 | 0.100 | 0.104 |  0.024 |     0.000 |     0.000 |     0.000 |     0.000 |
+-------+-----+------+-----+-----+-----+--------+-----------------------------------------------------+-------+-------+--------+-------+-------+-------+-------+--------+-----------+-----------+-----------+-----------+
```