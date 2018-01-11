---
title: "メモリに乗らない量のselect結果をcsvへ書き出す(python)"
date: 2018-01-10T18:27:24+09:00
tags: ["python"]
categories: ["tips"]
---

# 数百GBデータでもcsvに書き出したい！
業務用のツールを作成する際、 **DBのselect結果をcsvへ書き出したい**  
という要望は結構あるかと思います。  
でもって適当にcur作ってfetchallしてcsv.writerってやるんですがselect結果がうん千万件とかになるとfetchallした瞬間にメモリを食い潰していきます。  
気がついたらインスタンスが死んでたということもしばしば(当社比)  

## どーしたか
コード

``` python
def write_sql_result(sql, day, fname):
    conn = d.con_pos(connect)
    chunks = psql.read_sql(sql=sql,
                           con=conn,
                           chunksize=5000)
    with open(fname, 'w') as f:
        for r in chunks:
            r.to_csv(f, sep=',',
                     header=False,
                     na_rep='',
                     index=False)
```

結局、pandas.io.sqlで **chunksize** を設定し、 *TextFileReader* をforで回しつつcsvへ書き出すようにした。
その分ディスクIOが断続的に起きますがまぁそこはもうしょうがない  
chunksizeは知見だったので、メモ
