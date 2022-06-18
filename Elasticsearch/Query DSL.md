# Elasticsearch Query DSL

Elasticsearch 提供了 Query DSL (Domain Specific Language) 的搜尋語言來搜尋資料，書寫方式是使用 JSON 格式，查詢可分成兩種文法：

### **Leaf query clauses**

是針對特定的欄位查詢特定的值，底下又可分為兩類：

1. **[Term-level queries](https://www.elastic.co/guide/en/elasticsearch/reference/current/term-level-queries.html)**

用於查詢數據、結構化的資料，像是時間、IP 位址、價錢、ID 等等

是查詢精確的值(查詢完全符合內容的資料)

Term 是表達語意的最小單位，搜尋或是利用自然語言進行處理時都需要處理 term

不會做分詞(analyze)處理(分詞處理大意是指會將字拆開重組搜尋)

例： Term Query、Terms Query、Range Query、Exists Query、Prefix Query、Wildcard Query、Ids Query 、Fuzzy Query、Regexp Query 等

2. **[Full text queries](https://www.elastic.co/guide/en/elasticsearch/reference/current/full-text-queries.html)**

會做分詞處理，可搜尋像是 email 中的信件內容

例： Match Query、Match phrase Query、Query string Query 等

### **Compound query clauses**

複合查詢(也就是查詢條件有兩個以上時)，可以組合 leaf 或 compound 來查詢

例：Bool Query

接著依序介紹各種 Query 的用法，語法大致如下：

```json
POST(或 GET) 表格名稱/_search
{
  "query":{
    "查詢的種類": {
      // 可能會填上欄位名稱或是值
    }
  }
}
```

### Term Query

查詢**完全符合**的資料(是精確的查詢)，多一字少一字都不行

預設情況下需用全小寫才能查詢到資料，而中文無法查詢到資料

(官網建議不要用 Term query 查詢文字，理由是他不會做分詞處理，常常會查不到東西)

```json
POST house/_search
{
  "query":{
    "term": {
      "parkingLot": {
        "value": "n"
      }
    }
  }
}

// 也可這樣寫
POST house/_search
{
  "query": {
    "term": {
      "parkingLot": "n"
    }
  }
}
```

若要明確區分大小寫或是查詢中文，field 須加上 keyword 關鍵字

```json
POST house/_search
{
  "query":{
    "term": {
      "parkingLot.keyword": {
        "value": "N"
      }
    }
  }
}
```

### Terms Query

同  Term Query，但可在一個欄位查詢多個值

```json
POST house/_search
{
  "query":{
    "terms": {
      "parkingLot": ["y", "n"]
    }
  }
}
```

### Exists Query

用於查詢是否有此欄位

例：查詢有 city 欄位的資料

```json
POST house/_search
{
  "query": {
    "exists": {
      "field": "city"
    }
  }
}
```

### Range Query

gt：大於(greater than)

gte：大於等於(greater than or equal to)

lt：小於(less than)

lte：小於等於(less than or equal to)

例：查詢 unitPrice ≥20且≤40的資料

```json
POST house/_search
{
  "query": {
    "range": {
      "unitPrice": {
        "gte": 20,
        "lte": 40
      }
    }
  }
}
```

例：查詢 createTime 在前一天的(大於等於昨天日期，小於今天日期)

d 是 day，/d 是顯示到「日」就好，以此類推

```json
POST house/_search
{
  "query": {
    "range": {
      "createTime": {
        "gte":"now-1d/d",
        "lt":"now/d"
      }
    }
  }
}
```

### Ids Query

Elasticsearch 會自動給予每筆資料流水號(_id)，此即為查詢 _id 的 query

例：查詢 _id 為 2 的資料

```json
POST house/_search
{
  "query": {
    "ids": {
      "values": ["2"]
    }
  }
}
```

### Match Query

match 可想成是查詢有配對到的資料(亦即有「包含」即可)

會把要查詢的字拆開來去搜尋符合的資料

所以很類似模糊查詢

但 field 只要加上 keyword 關鍵字，一樣會變成精確查詢

例：查詢 city 欄位中包含「北市台」三字的資料

屬性 operator 是 or 時，代表查詢 city 欄位中有「台」或「北」或「市」的資料，operator 是 and，代表查詢 city 欄位中有「台」及「北」及「市」的資料

minimum_should_match 代表最少應符合幾字

```json
POST house/_search
{
  "query": {
    "match": {
      "city": "北市台"
    }
  }
}

POST house/_search
{
  "query": {
    "match": {
      "city": {
        "query": "北市台"
	      //"operator": "and",
        //"minimum_should_match": 3,
      }
    }
  }
}
```

預設的 operator 是 or，因此會找到多筆資料

搭配 operator 或 minimum_should_match，可以讓查詢結果更準確

### Match phrase Query

查詢 city 欄位中**包含**「新北」(連續字)的資料

```json
POST house/_search
{
  "query": {
    "match_phrase": {
      "city": "新北"
    }
  }
}
```

### Match all Query

沒有任何查詢條件，全部查詢(會返回所有的資料)

```json
POST house/_search
{
  "query": {
    "match_all": {}
  }
}
```

### Bool Query

當查詢條件不只一個時，就必須使用 Bool Query

講到 Bool Query，就必須提到：在 Elasticsearch 中，有 Query & Filter 兩種不同的 Context，Query Context 會對相關性進行算分，而 Filter Context 不會進行算分，但可利用 cache 來取得更好的效能

**Bool Query 的查詢修飾詞**

must：必須滿足的條件。或者說，只要滿足條件，就一定會出現在搜尋結果中，相當於 and，屬於 Query Context

must_not：不可出現在查詢結果中，相當於 not，屬於 Filter Context

should：至少要滿足一個條件，相當於 or，屬於 Query Context

filter：效果同 must，但不會算分，屬於 Filter Context

**使用方式**

先指名使用 bool query，再寫想要哪一種修飾詞，裡面再寫要查詢的條件(使用先前提到的 query )

**Must**

例：city 必須為台北市

```json
POST house/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "term": {
            "city.keyword": {
              "value": "台北市"
            }
          }
        }
      ]
    }
  }
}
```

**Must_not**

例：city 不可為新北市

```json
POST house/_search
{
  "query": {
    "bool": {
      "must_not": [
        {
          "term": {
            "city.keyword": {
              "value": "新北市"
            }
          }
        }
      ]
    }
  }
}
```

**Should**

例：查詢出 totalPrice 介於 1500 到 3000 之間，**或** city 為台北市者

```json
POST house/_search
{
  "query": {
    "bool": {
      "should": [
        {
          "range": {
            "totalPrice": {
              "gte": 1500,
              "lte": 3000
            }
          }
        },
        {
          "term": {
            "city.keyword": {
              "value": "台北市"
            }
          }
        }
      ]
    }
  }
}
```

**Filter**

例：unitPrice 必須介於 10 到 50 之間

```json
POST house/_search
{
  "query": {
    "bool": {
      "filter": [
        {
          "range": {
            "unitPrice": {
              "gte": 10,
              "lte": 50
            }
          }
        }
      ]
    }
  }
}
```

### 多層的 Bool Query

bool query 中還可以再包 bool query(因為可能同時會用到兩個修飾詞)

例：找出 city 不為新北市或 unitPrice 介於 40 到 100 之間者

```json
POST house/_search
{
  "query": {
    "bool": {
      "should": [
        {
          "bool": {
            "must_not": [
              {
                "term": {
                  "city.keyword": "新北市"
                }
              }
            ]
          }
        },
        {
          "range": {
            "unitPrice": {
              "gte": 40,
              "lte": 100
            }
          }
        }
      ]
    }
  }
}
```