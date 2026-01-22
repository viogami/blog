---
title: 【PYQT】赛马娘抽卡模拟器-基于Bwiki的抽卡模拟(下)
slug: "uma_drawcard_2"
subtitle: 
summary:
date: 2023-02-05
cardimage: 
featureimage: 
caption: 
authors:
  - viogami: author.png
---
## 前言

本贴将之前没有连接数据库的抽卡模拟器全部完善，最终已经放在了Github上

<!--more-->

用的mysql构建的数据库，导出sql文件，用SQlite3再读一遍，生成一个db文件，创建了四个表

| 表          | 属性                                 |
| ----------- | ------------------------------------ |
| 卡池表      | 序号，卡池名，up角色                 |
| 卡牌表      | 卡牌序号，卡牌名，卡牌星级，卡牌种类 |
| 卡池&卡牌表 | 序号，卡池名，卡牌名                 |
| 抽卡记录表  | 记录序号，卡池名，卡牌名             |

简单易用就好，其实后两个表应该用卡池和卡牌表的主键(序号)来做外码的，但查询语句多写一段有点麻烦，就不绕弯子，直接用名字查了。

## 预览

功能实现

- [X] 单抽+十连的结果图片显示
- [X] 十连保底出现SR
- [X] 重做界面，删除无用的登陆界面
- [X] 可以抽取马娘池，也可以抽取支援卡池子
- [X] 对抽卡结果的统计
- [X] 可以自主选择卡池
- [X] 选择不同卡池出现不同的标志性图片
- [X] 显示公告
- [X] 建立抽卡模拟器的数据库，建立表：卡池表，卡牌表，卡池&卡牌表，抽取结果表
- [X] 连接数据库
- [X] 抽卡后输出抽卡日志

## 说明

抽卡逻辑为：

- 先得到本次抽卡卡的稀有度
- 在数据库中查询该稀有度的所有卡牌，随机返回一张
- 十连保底，但up的概率并未添加

对于日志中的None，表示的是卡名称查询不到。这是因为卡牌数据库构建的不完备，数据太少，有的品质的卡没有具体写入，修改data文件夹中的数据可以完善。data文件夹就是数据库中构建的三个表，而第四个表--抽卡记录表在程序运行后会以txt格式输出到根目录。

## 具体修改

为了简明，我把数据库相关的函数写在了conn2sql.py文件中，其中包含数据库的创建，添加元素，查找卡牌以及将数据库的某表输出为txt
源码：

```python
import sqlite3
import random

# 创建数据库
def create_database_from_sql(sql_file, db_name):
    with open(sql_file, 'r') as file:
        sql_statements = file.read()

    conn = sqlite3.connect(db_name)
    cursor = conn.cursor()
    cursor.executescript(sql_statements)
    conn.commit()
    conn.close()

# 数据写入数据库
#添加卡池信息
def insert_cardpool(text_file, db_name):
    conn = sqlite3.connect(db_name)
    cursor = conn.cursor()
    with open(text_file, 'r') as file:
        lines = file.readlines()
        for line in lines:
            pool_no, pool_name,pool_up = line.strip().split(",")
            cursor.execute("INSERT INTO card_pool (pool_no,pool_name,pool_up) VALUES (?,?,?)",
                           (pool_no, pool_name,pool_up))   
    conn.commit()
    conn.close()
#添加卡牌信息
def insert_card(text_file, db_name):
    conn = sqlite3.connect(db_name)
    cursor = conn.cursor()

    with open(text_file, 'r') as file:
        lines = file.readlines()
        for line in lines:
            card_no, card_name,card_kind,card_star= line.strip().split(",")
            cursor.execute("INSERT INTO card (card_no, card_name,card_kind,card_star) VALUES (?,?,?,?)",
                           (card_no, card_name,card_kind,card_star))   
    conn.commit()
    conn.close()
#添加卡池和卡牌信息
def insert_pool_card(text_file, db_name):
    conn = sqlite3.connect(db_name)
    cursor = conn.cursor()
    with open(text_file, 'r') as file:
        lines = file.readlines()
        for line in lines:
            pool_name,card_name=line.strip().split(",")
            cursor.execute("INSERT INTO poolcard (pool_name, card_name) VALUES (?,?)",
                           (pool_name,card_name))   
    conn.commit()
    conn.close()

# 添加抽卡记录
def insert_draw_log(db_name,pool_name,card_name):
    conn = sqlite3.connect(db_name)
    cursor = conn.cursor()
    cursor.execute("INSERT INTO draw_log (pool_name,card_name) VALUES (?,?)", (pool_name,card_name))
  
    conn.commit()
    conn.close()

#返回卡牌名
def get_cardname(pool_name, star, db_name):
    conn = sqlite3.connect(db_name)
    cursor = conn.cursor()

    cursor.execute("SELECT card_name FROM poolcard WHERE pool_name = ?", (pool_name,))
    pool_cards = cursor.fetchall()
  
    pool_cards = [card[0] for card in pool_cards]

    matching_star_cards = [card for card in pool_cards if get_card_star(card, conn) == star]

    if matching_star_cards:
        random_card = random.choice(matching_star_cards)
    else:
        random_card = None
    conn.close()

    return random_card

def get_card_star(card_name, conn):
    cursor = conn.cursor()
    cursor.execute("SELECT card_star FROM card WHERE card_name = ?", (card_name,))
    result = cursor.fetchone()
    if result:
        return result[0]
    else:
        return None
  
#将数据库的表输出为txt
def export_table_to_txt(table_name, db_name, txt_file):
    conn = sqlite3.connect(db_name)
    cursor = conn.cursor()
    cursor.execute(f"SELECT * FROM {table_name}")
    table_data = cursor.fetchall()
    conn.close()
    #写入文件
    with open(txt_file, 'w') as file:
        for row in table_data:
            row_str = "\t".join(str(value) for value in row)
            file.write(row_str + "\n")
```

在启动器/主文件中引用即可。

除此之外，还有修改原先的start.py文件的部分内容。
为了使得抽卡记录每次都要存入数据库中，抽卡的单抽和十连函数要略加修改。
具体为

- 先添加控件信息的读取

```python
poolname=self.cardpool_Box.currentText()#卡池名
db_name = "card_database.db"#db文件名
```

- 再添加使用数据库查询的函数，使得每次抽卡获得具体的卡牌名

```python
 #得到抽卡结果
        cardname=conn2sql.get_cardname(pool_name=poolname,star=star,db_name=db_name)
        conn2sql.insert_draw_log(db_name,poolname,cardname) #存入抽卡记录的数据库
```

- 最后添加抽卡日志的输出

```python
#输出日志文件
        table_name="draw_log"
        txt_file="draw_log.txt"
        conn2sql.export_table_to_txt(table_name,db_name,txt_file) #抽卡记录表导出为txt
```

对主文件中的四个抽卡函数(`one,one2,ten,ten2`)均做类似操作即完成了数据库的连接。

公告内容可有可无，就是一个信息弹窗。

注意数据库不要重复创建，开始先判空。

## 最后

### 放在了 `<u>`[github](https://github.com/viogami/DrawCard_uma) `</u>`上，觉得有用可以Star☆ ，hh

本以为简单搞完的，结果还是搞到了半夜T^T

<script src="https://giscus.app/client.js"
        data-repo="viogami/blog"
        data-repo-id="R_kgDOORWDyA"
        data-category="Announcements"
        data-category-id="DIC_kwDOORWDyM4Conxc"
        data-mapping="pathname"
        data-strict="0"
        data-reactions-enabled="1"
        data-emit-metadata="0"
        data-input-position="top"
        data-theme="preferred_color_scheme"
        data-lang="zh-CN"
        crossorigin="anonymous"
        async>
</script>
