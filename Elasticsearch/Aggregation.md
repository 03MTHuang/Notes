# Aggregation 聚合

Elasticsearch 除了提供搜尋的功能外，也提供資料統計的功能，Aggregation 指的即是資料統計

和 query 是同層級的

聚合主要的功能有以下四個：

- Metric Aggregation (指標型聚合)
- Bucket Aggregation (桶型聚合)
- Pipeline Aggregation (管道型聚合)
- Matrix Aggregation (矩陣型聚合)

Aggregation 的基本寫法：

```json
// NAME：要幫這個統計取個名稱(看起來可用中文)
// AGG_TYPE：哪一種單值分析
// FIELD：統計哪一個欄位
GET(或POST) house/_search?size=0
{
  "aggs": {
    "NAME": {
      "AGG_TYPE": {
        "field": "FIELD"
      }
    }
  }
}
```

### ****Metric Aggregation (指標型聚合)****

簡單來說就是可以用來計算最大值、最小值、平均值、總和等等的功能

單值分析(single-value numeric metrics aggregation)可以使用的功能包含：

- min：最小值
- max：最大值
- avg：平均值
- sum：總合(預設無條件捨去?)
- value_count：指定欄位的個數
- cardinality：基數值，就是不重複的數值的個數(去除重複後的個數)，類似 SQL 的 distinct count

例：找出所有資料中 unitPrice 的最大值及最小值

```json
GET house/_search
{
  "aggs": {
    "max_unitPrice": {
      "max": {
        "field": "unitPrice"
      }
    },
    "min_unitPrice": {
      "min": {
        "field": "unitPrice"
      }
    }
  }
}
```

結果視窗中在最外層 hits 同層級的地方，有一個 aggregations 屬性，會回傳統計資料

```json
{
	...
	"aggregations" : {
	  "min_unitPrice" : {
	    "value" : 30.0
	  },
	  "max_unitPrice" : {
	    "value" : 100.0
	  }
	}
}
```

多值分析(multi-value numeric metrics aggregation)常用的功能包含 :

- stats : 列出一系列的數值型別統計
- extended_stats : stats 的擴充，可以列出更多統計資料，例如 : 標準差
- percentiles : 百分位數統計
- percentile_ranks : 百分等級統計
- top_hits : 搜尋結果的前幾個

**stats**

```json
GET house/_search
{
  "aggs": {
    "stats分群": {
      "stats": {
        "field": "unitPrice"
      }
    }
  }
}
```

```json
{
	...
	"aggregations" : {
    "stats分群" : {
      "count" : 3,
      "min" : 30.0,
      "max" : 100.0,
      "avg" : 54.333333333333336,
      "sum" : 163.0
    }
  }
}
```

### ****Bucket Aggregation (桶型聚合)****

類似 SQL 的 group by

- terms : 依照單詞進行分群
- range : 依照指定區間進行分群
- date_range : 依照日期進行分群
- histogram : 依照指定數值作為間隔區間分群
- date_histogram : 依照指定時間間隔分群
- filter：用查詢的方式分群(亦可做統計)

**terms**

terms 預設是回傳 10 筆分群結果，若分群會超過 10 筆以上，可在 size 屬性中設定回傳筆數

依照 city 分群

```json
GET house/_search
{
  "aggs": {
    "terms分群":{
      "terms": {
        "field": "city.keyword",
        "size": 10
      }
    }
  }
}
```

city 為台北市的資料有 2 筆，為新北市的有 1 筆

```json
{
	...
	"aggregations" : {
		"terms分群" : {
		  "doc_count_error_upper_bound" : 0,
		  "sum_other_doc_count" : 0,
		  "buckets" : [
		    {
		      "key" : "台北市",
		      "doc_count" : 2
		    },
		    {
		      "key" : "新北市",
		      "doc_count" : 1
		    }
		  ]
		}
	}
}
```

**range**

to：**小於**某數

from：**大於等於**某數

```json
GET house/_search
{
  "aggs": {
    "range分群":{
      "range": {
        "field": "unitPrice",
        "ranges": [
          { 
            "to": 50
          },
          {
            "from": 50,
            "to": 100
          },
          {
            "from": 100
          }
        ]
      }
    }
  }
}
```

```json
{
	...
	"aggregations" : {
	  "range分群" : {
	    "buckets" : [
	      {
	        "key" : "*-50.0",
	        "to" : 50.0,
	        "doc_count" : 2
	      },
	      {
	        "key" : "50.0-100.0",
	        "from" : 50.0,
	        "to" : 100.0,
	        "doc_count" : 0
	      },
	      {
	        "key" : "100.0-*",
	        "from" : 100.0,
	        "doc_count" : 1
	      }
	    ]
	  }
	}
}
```

**date_range**

用 range 也行

查詢 createTime 為兩年內的(即大於等於兩年前的時間點)(/d指時間顯示到最小的單位為 day)

```json
GET house/_search
{
  "aggs": {
    "date_range分群":{
      "date_range": {
        "field": "createTime",
        "ranges": [
          {
            "from": "now-2y/d",
						"to": "now/d"
          }
        ]
      }
    }
  }
}
```

```json
{
	...
	"aggregations" : {
    "date_range分群" : {
      "buckets" : [
        {
          "key" : "2020-06-06T00:00:00.000Z-2022-06-06T00:00:00.000Z",
          "from" : 1.5914016E12,
          "from_as_string" : "2020-06-06T00:00:00.000Z",
          "to" : 1.6544736E12,
          "to_as_string" : "2022-06-06T00:00:00.000Z",
          "doc_count" : 1
        }
      ]
    }
  }
}
```

**histogram**

依一定的間隔分群

例：以 10 為區間分群

```json
GET house/_search
{
  "aggs": {
    "histogram分群": {
      "histogram": {
        "field": "unitPrice",
        "interval": 10,
        "min_doc_count": 1
      }
    }
  }
}
```

30幾的有 2 筆(30、33.95)，100多的有 1 筆(100)

```json
{
	...
	"aggregations" : {
	  "histogram分群" : {
	    "buckets" : [
	      {
	        "key" : 30.0,
	        "doc_count" : 2
	      },
	      {
	        "key" : 100.0,
	        "doc_count" : 1
	      }
	    ]
	  }
	}
}
```

****date_histogram****

時間依一定的間隔分群

**子分析**

分群後再分群

**filter**

例：city 為台北市的資料為一群，並計算此群的 unitPrice 平均

```json
GET house/_search
{
  "aggs": {
    "filter分群":{
      "filter": {
        "term": {
          "city.keyword": "台北市"
        }
      },
      "aggs": {
        "台北市的平均單價": {
          "avg": {
            "field": "unitPrice"
          }
        }
      }
    }
  }
}
```

```json
{
	...
  "aggregations" : {
    "filter分群" : {
      "doc_count" : 2,
      "台北市的平均單價" : {
        "value" : 66.5
      }
    }
  }
}
```

### ****Pipeline Aggregation (管道型聚合)****

- Parent：在父聚合的結果上進行聚合分析並且可以計算出新的桶子或是將新的聚合結果加入到現有的桶子中。
- Sibling：在兄弟 (同級) 聚合的結果上進行聚合分析。計算出一個新的聚合結果，結果與兄弟聚合的結果同級。

**Sibling**

Sibiling 提供了以下這些常用的功能，這些功能大致與指標型聚合提供的是一樣的，差別在 Sibiling 是計算每一分群內的資料

- avg_bucket
- max_bucket
- min_bucket
- sum_bucket
- stats_bucket
- extended_stats_bucket
- percentiles_bucket

**sum_bucket**

在「依city分群」的 terms 分群下，又有一個指標型聚合，名稱是「分群中的unitPrice總和」，而另一個「兄弟分群」的 sum_bucket 分群下，則須在 buckets_path 屬性中指名要做總合的路徑

```json
GET house/_search
{
  "aggs": {
    "依city分群": {
      "terms": {
        "field": "city.keyword",
        "size": 10
      },
      "aggs": {
        "分群中的unitPrice總和": {
          "sum": {
            "field": "unitPrice"
          }
        }
      }
    },
    "兄弟分群": {
      "sum_bucket": {
        "buckets_path": "依city分群>分群中的unitPrice總和"
      }
    }
  }
}
```

```json
{
	...
	"aggregations" : {
	  "依city分群" : {
	    "doc_count_error_upper_bound" : 0,
	    "sum_other_doc_count" : 0,
	    "buckets" : [
	      {
	        "key" : "台北市",
	        "doc_count" : 2,
	        "分群中的unitPrice總和" : {
	          "value" : 133.0
	        }
	      },
	      {
	        "key" : "新北市",
	        "doc_count" : 1,
	        "分群中的unitPrice總和" : {
	          "value" : 30.0
	        }
	      }
	    ]
	  },
	  "兄弟分群" : {
	    "value" : 163.0 // sum_bucket 會算全部資料的總和
	  }
	}
}
```