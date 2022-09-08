# Citus試してみる

手順は[公式チュートリアル](https://docs.citusdata.com/en/stable/get_started/tutorial_multi_tenant.html)

実行環境はWindows 10。wsl2がバックエンドのdockerで動かした。

```console
# dockerでcitusのイメージをダウンロードしてコンテナを起動
docker run -d --name citus -p 5432:5432 -e POSTGRES_PASSWORD=password citusdata/citus:11.0

# ダウンロードしたファイルをコンテナにコピー
docker cp .\companies.csv citus:.
docker cp .\campaigns.csv citus:.
docker cp .\ads.csv citus:.

# コンテナ内のpsqlコンソールを開く
docker exec -it citus psql -U postgres
```

普通にテーブル作成できる。

```sql
CREATE TABLE companies (
    id bigint NOT NULL,
    name text NOT NULL,
    image_url text,
    created_at timestamp without time zone NOT NULL,
    updated_at timestamp without time zone NOT NULL
);

CREATE TABLE campaigns (
    id bigint NOT NULL,
    company_id bigint NOT NULL,
    name text NOT NULL,
    cost_model text NOT NULL,
    state text NOT NULL,
    monthly_budget bigint,
    blacklisted_site_urls text[],
    created_at timestamp without time zone NOT NULL,
    updated_at timestamp without time zone NOT NULL
);

CREATE TABLE ads (
    id bigint NOT NULL,
    company_id bigint NOT NULL,
    campaign_id bigint NOT NULL,
    name text NOT NULL,
    image_url text,
    target_url text,
    impressions_count bigint DEFAULT 0,
    clicks_count bigint DEFAULT 0,
    created_at timestamp without time zone NOT NULL,
    updated_at timestamp without time zone NOT NULL
);
```

テーブルの情報を表示させるコマンド`\d companies`の結果は

```
                         Table "public.companies"
   Column   |            Type             | Collation | Nullable | Default
------------+-----------------------------+-----------+----------+---------
 id         | bigint                      |           | not null |
 name       | text                        |           | not null |
 image_url  | text                        |           |          |
 created_at | timestamp without time zone |           | not null |
 updated_at | timestamp without time zone |           | not null |
```

となっていて、たしかにテーブルの作成ができているみたい。

プライマリーキーを追加する。

```sql
ALTER TABLE companies ADD PRIMARY KEY (id);
ALTER TABLE campaigns ADD PRIMARY KEY (id, company_id);
ALTER TABLE ads ADD PRIMARY KEY (id, company_id);
```

追加できたか確認する。

```
# \d companies
                         Table "public.companies"
   Column   |            Type             | Collation | Nullable | Default
------------+-----------------------------+-----------+----------+---------
 id         | bigint                      |           | not null |
 name       | text                        |           | not null |
 image_url  | text                        |           |          |
 created_at | timestamp without time zone |           | not null |
 updated_at | timestamp without time zone |           | not null |
Indexes:
    "companies_pkey" PRIMARY KEY, btree (id)
```

お～、btreeだ。これインデックスの持たせ方もこだわれるのかも。

## データの分散（本題）

すべてのテーブルを、company_idでシャーディングさせる。たしかにどれもcompanyに紐づいてるからね。それはそうって感じだ。せっかくシャーディングしても、ノード間でjoinなんかが走ると時間がかかるというのは想像にたやすい。

```sql
SELECT create_distributed_table('companies', 'id');
```

出力は

```
 create_distributed_table
--------------------------

(1 row)
```

できていそう。残りのテーブルも走らせる。

```sql
SELECT create_distributed_table('campaigns', 'company_id');
SELECT create_distributed_table('ads', 'company_id');
```

ちなみにこの時点で`\d companies`や`\d ads`を走らせても出力に変化はなかった。透過性を感じる。

さっきダウンロードしておいたCSVをデータベースにぶち込む

```
\copy companies from 'companies.csv' with csv
\copy campaigns from 'campaigns.csv' with csv
\copy ads from 'ads.csv' with csv
```

一通りコマンドを試してみたが、どれも動く。分割されていると感じない。

```sql
SELECT DISTINCT com.name
FROM campaigns AS cam
INNER JOIN companies as com ON cam.company_id=com.id
WHERE cam.monthly_budget > 9900;
```

これは、monthly_budgetが9900を超えたキャンペーンがある企業名一覧を表示する。
こういうあるあるなやつも動く。えらい。

というか、psqlがそもそもpostgreSQLのインターフェースっぽい。citusは追加のインターフェースを提供しているだけということ？すごいなあ

## 片付け

```console
docker stop citus
```

おしまい
