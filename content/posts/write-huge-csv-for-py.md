---
title: "メモリに乗らない量のselect結果をcsvへ書き出す(python)"
date: 2018-01-10T18:27:24+09:00
tags: ["python", "aws", "redshift"]
categories: ["tips"]
draft: False
---

# 数百GBデータでもcsvに書き出したい！
業務用のツールを作成する際、 **DBのselect結果をcsvへ書き出したい**  
という要望は結構あるかと思います。  
でもって適当にcur作ってfetchallしてcsv.writerってやるんですがselect結果がうん千万件とかになるとfetchallした瞬間にメモリを食い潰していきます。  
気がついたらインスタンスが死んでたということもしばしば(当社比)  

## どーしたか
※2018-10-07 update
~~結局、pandas.io.sqlで **chunksize** を設定し、 *TextFileReader* をforで回しつつcsvへ書き出すようにした。~~  
~~その分ディスクIOが断続的に起きますがまぁそこはもうしょうがない~~  
~~chunksizeは知見だったので、メモ~~

その後実際にうん千万件のデータで試した所、`pandas.io.sql`の**chunksize**は一旦ローカルに全てfetchするという動作でした。(メモリ食い潰してec2が死にました)  
https://github.com/pandas-dev/pandas/issues/12265  

なので、サーバサイドカーソルを使用してデータの書き出しを行いました。

```python
def write_sql_result(sql f):
    with get_connection() as conn:
        with conn.cursor('query1') as cur:
            cur.itersize = 1000
            cur.execute(sql)
            with gzip.open(f, 'wt') as f:
                w = csv.writer(f, lineterminator='\n', quoting=csv.QUOTE_ALL)
                for i in cur:
                    w.writerow(i)
```

とりあえず、これで大丈夫そう
