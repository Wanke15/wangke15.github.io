### 1. 基于TF-IDF
```sql

with query_table as (
select concat_ws('', biz_date, cast(biz_hour as string), user_id) as sess_id, query 
from search_algo.search_log
where biz_date >= '2024-11-01'
--- impala中一个中文字符length为3，这里其实是限制了query长度为2~5
and length(query) > 3 and length(query) <= 15
),


documents as (
select a.sess_id as sess_a, a.query as doc_id, b.sess_id as sess_b, b.query as word
from query_table a
left join query_table b
on a.sess_id = b.sess_id and a.query != b.query
where b.sess_id is not null
),

tf AS (
SELECT
    doc_id,
    word,
    count(1) AS tf
FROM
    documents
GROUP BY doc_id, word
),

doc_word_count AS (
SELECT
    doc_id,
    SUM(tf) AS total_count
FROM
    tf
GROUP BY
    doc_id
),


normalized_tf AS (
SELECT
    d.doc_id,
    d.word,
    d.tf / dwc.total_count AS normalized_tf
FROM
    tf d
JOIN
    doc_word_count dwc
ON
    d.doc_id = dwc.doc_id
),


df AS (
SELECT
    word,
    COUNT(DISTINCT doc_id) AS doc_freq
FROM
    documents
GROUP BY
    word
),


total_docs AS (
SELECT
    COUNT(DISTINCT doc_id) AS total_doc_count
FROM
    documents
),


idf AS (
SELECT
    df.word,
    log10(t.total_doc_count / df.doc_freq) AS idf
FROM
    df
CROSS JOIN
    total_docs t
),


tf_idf AS (
SELECT
    ntf.doc_id as base_query,
    ntf.word as rec_query,
    ntf.normalized_tf,
    idf.idf,
    ntf.normalized_tf * idf.idf AS tf_idf
FROM
    normalized_tf ntf
JOIN
    idf
ON
    ntf.word = idf.word
)


insert overwrite search_algo.query_recommend
PARTITION (biz_date = from_timestamp(days_sub(now(), 1), 'yyyy-MM-dd_HH'))
select
row_number() over(partition by base_query order by tf_idf desc) as rn,
*
from tf_idf;
```


### 2. 规则筛选与可视化
```python

from sqlalchemy import create_engine
import pandas as pd
import streamlit as st

# @st.cache_data
def query_impala(sql):
    URL = 'impala://xxxx:xxxx'
    impala_engine = create_engine(URL)
    df = pd.read_sql(sql, impala_engine)
    return df


def edit_distance(word1, word2):
    m, n = len(word1), len(word2)
    dp = [[0] * (n + 1) for _ in range(m + 1)]

    for i in range(m + 1):
        dp[i][0] = i
    for j in range(n + 1):
        dp[0][j] = j

    for i in range(1, m + 1):
        for j in range(1, n + 1):
            if word1[i - 1] == word2[j - 1]:
                dp[i][j] = dp[i - 1][j - 1]
            else:
                dp[i][j] = 1 + min(dp[i - 1][j], dp[i][j - 1], dp[i - 1][j - 1])

    return dp[m][n]


query = st.text_input("搜索词", "感冒")
biz_date = st.text_input("数据日期", "2024-12-03_14")


sql = """
select * from search_algo.query_recommend where base_query = '{}' and biz_date = '{}' limit 20
""".format(query, biz_date)


df = query_impala(sql)


def filter(df):
    valid_query = []
    records = []
    edit_threshold = 1
    for idx, row in df.iterrows():
        b, r = row["base_query"], row["rec_query"]
        if b in r or r in b:
            continue
        else:
            flag = sum([1 if (w in r or r in w) else 0 for w in valid_query])
            if flag > 0:
                continue
            else:
                word = ""
                if "(" in r and ")" in r:
                    splits = r.split("(")[-1]
                    word = splits[:-1]
                else:
                    word = r

                if not valid_query:
                    valid_query.append(word)

                # 检查编辑距离
                edit_flag = 0
                for w in valid_query:
                    if edit_distance(w, word) <= edit_threshold:
                        print(w, word)
                        edit_flag = 1
                        break

                if edit_flag == 0:
                    valid_query.append(word)

    print("*****", valid_query)

    # 1. 字形（语义todo）相似去除
    df = df[df["rec_query"].isin(valid_query)]

    # 2. 得分阈值
    df = df[df["tf_idf"] > 0.01]
    df = df[df['rn'] <= 10]

    # 3. 去除太热门的query
    filter_querys = ["延时"]
    df = df[~df["rec_query"].isin(filter_querys)]

    return df

df = filter(df)

st.write("推荐词：")
st.table(df)


```