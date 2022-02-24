# Elasticsearch 정리

## Elasticsearch(이하 ES)란?

- 검색 엔진
- 검색을 위해 단독으로 사용 가능
- ELK 스택의 한 부분으로서 사용 가능 

## Architecture

- Cluster : Node들로 구성
- Node : 하나의 Elasticsearch server
- Index : Document를 저장 (RDB의 Table과 유사)
- Document : Index에 저장되는 데이터 (RDB의 Row와 유사)

![architecture](https://media.vlpt.us/images/jwpark06/post/da9ebf2d-a3d2-4f30-ae66-7a0ed85fc168/image.png)

### Sharding
- DB의 샤딩과 유사한 개념
- Index를 여러개의 Node에 분산해 놓는 것
- Sharding을 통해서 대용량의 데이터도 분산해서 저장 가능. (**Scalability**)
- 각 shard는 Apache Lucene Index (루씬 = 색인과 검색 기능을 제공하는 정보 검색 라이브러리 / Elasticsearch는 Lucene 기반 검색 엔진) 
- Shard를 통해서 Index의 throuput 증가 가능 -> 성능 향상의 목적도 가지고 있음. 
(Performance)


### Replication
- Cluster 내부의 Node에 문제가 생긴다면, Node가 가지고 있는 Index들에도 문제가 발생하게 된다.
- Elasticsearch는 이러한 문제를 방지하기 위해서 Replication을 지원한다. (For fault tolerance)
- Replication도 Sharding과 마찬가지로 Index 단위로 지원되고, 다른 노드(즉, 다른 ES 서버)에 사본을 만들어 놓는다.
- 복제된 shard의 원본을 primary shard라고 한다.
- shard의 사본을 replica shard 라고 한다. 
- primary shard + replica shard -> replication group
- replica shard는 primary shard 와 같은 node에 저장되지 않는다.

- 참고 : [Shard의 갯수와 크기 정하기](https://jaemunbro.medium.com/elastic-search-%EC%83%A4%EB%93%9C-%EC%B5%9C%EC%A0%81%ED%99%94-68062271fb64)

## Node roles

- 각각의 노드들이 1가지 Role만 수행하는 것은 아니다.

### 1. Master-eligible
- Master node가 될 수 있음을 의미.

### 2. Data
- query 실행

> 하나의 Cluster에는 반드시 하나의 Master노드가 필요하다.

### 3. Ingest
- node가 ingest pipeline을 수행하도록 할 수 있다
> Ingesting : index에 document를 추가 하는 것.

### 4. Coordination
> Coordination : query를 분산시키고, 결과를 aggregate하는 것.

## Managing Documents
### Routing
- Doucument가 어떤 Shard에 저장되어야 할 지 결정하는 과정.
- Elasticsearch는 default Routing 전략을 기본적으로 제공하고, 필요하다면 Routing 전략을 커스터마이징 할 수 있다.
- default formula
> shard_num = hash(_routing) % num_primary_shards

- 위와 같이 index의 shard의 갯수가 routing formula에서 사용되기 때문에 index가 한번 생성되면, shard의 갯수를 바꿀 수 없는 것이다.

- **이러한 상황에서 새로운 Document를 Indexing 할 때는 문제가 없겠지만, 기존의 문서를 가져올 때 문제가 된다.**

- 이러한 default routing 전략은 document를 각 shard에 고르게 분배한다.

### How Elasticsearch reads data
- read request는 coordinating nodes에 의해서 관리됨.
- Routing 은 Document가 속한 Replication Group을 찾기 위해 사용된다.

# Analyzing and Mappings

## Analysis
- document의 text field는 indexing 과정에서 검색에 최적화된 형태로 저장된다.
- 그러한 과정을 Analysis라고 한다.

## Analyzer
- 3가지 구성요소로 이루어짐.
- 관련 API : Analyze API

### 1. Character Filters
- original string을 받아서 add, remove, change characters. (string -> string)

### 2. Tokenizer
- character filtering이 끝난 string을 tokenizing. (string -> tokens)

### 3. Token filters 
- token들을 받아서 수정. (tokens -> tokens)

![analysis](https://www.wikitechy.com/tutorials/elasticsearch/img/elasticsearch-images/elasticsearch-tokenizer-analysis.png)

## Inverted Index
- term(== token)을 기준으로, 해당 term을 가지고 있는 document들을 구분
- relavance scoring을 위한 정보들도 가지고 있다.
- Index의 각각의 Filed 별로 Inverted Index가 만들어진다.
- text field가 아닌 field들은 다른 자료구조를 쓴다. 
![inverted index](https://habrastorage.org/webt/5e/f9/f0/5ef9f0a04c3f6781438716.jpeg)

## Analyze API
- Custom Analyzer를 만들고, 실제로 Document에 적용해보기 전에 유용.

# Mappings

## Data Types

특별한 Data Type 3가지

### 1. Object Type

- Apache Lucene에서는 object 타입을 지원하지 않기 때문에 ES에서는 json object type을 flatten 한다.

json object
```json
{
    "name":"Coffe Maker",
    "price":64.2,
    "in_stock":10,
    "manufacturer":{
        "name":"Nespresso",
        "country":"Switzerland"
    }
}
```

flatten object
```
{
    "name":"Coffe Maker",
    "price":64.2,
    "in_stock":10,
    "manufacturer.name":"Nespresso",
    "manufacturer.country":"Switzerland"
}
```

### 2. Nested Type

- Object Type과 달리 [**배열에서의 객체의 관계를 유지**](https://www.elastic.co/guide/en/elasticsearch/reference/current/nested.html)하기 위한 목적을 가지고 있다.

```json
{
  "group" : "fans",
  "user" : [ 
    {
      "first" : "John",
      "last" :  "Smith"
    },
    {
      "first" : "Alice",
      "last" :  "White"
    }
  ]
}
```

```json
{
  "group" :      "fans",
  "user.first" : [ "alice", "john" ],
  "user.last" :  [ "smith", "white" ]
}
```

### 3. Keyword Type
- exact matching을 위해서 사용한다.
- filtering, aggregations, sorting에서 사용한다.
- full-text search를 사용하고 싶다면, text type을 사용한다.
- keyword field는 keyword analyzer에 의해서 analyze된다.
- keyword analyzer는 origin string을 **그대로 반환**한다.

## Type coersion
- 일종의 자동 형변환?
- defalut로 활성화 되어 있지만, 안정적인 type만 받기를 원한다면, disable 하는 것도 방법이다. 

## Array Data Type in Elastic Search
- Elasticsearch에는 array가 존재하지 않는다.
대신, 입력받은 값들을 concatination해서 저장한다.
- array 내부에 있는 object들을 독립적으로 다루고자 한다면, nested type을 사용하자!

## Date Type
- epoch 이후 몇 millisecond가 지났는지를 기준으로 long 형태로 저장된다.
unix timestamp와 혼용해서 사용하지 않도록 주의해야한다. 가독성을 위해서 formatting을 사용한다.

## How missing fields are handled
- RDB와는 다르게 ES는 field가 존재하지 않아도, indexing이 가능하다.
- 즉, document들 중에서 몇몇 field값이 아예 존재하지 않는 document들도 존재 할 수 있다.

## Important Mapping Parameters
- format : date field를 formatting하기 위해서 사용.
- coerce : type coersion enable/disable 설정.
- doc_values : 
    - doc_values를 false로 설정하면, disk-space를 절약 할 수 있다.
    - aggregations, sorting, scripting을 사용하지 않을 경우에만, doc value를 비활성화 한다.
    - size가 큰 index에는 효과적이지만, 그렇지 않을 경우에는 큰 의미는 없다.
    - 특별한 이유가 없다면, false로 설정하지 말자.
    - Doc values are the on-disk data structure, built at document index time, which makes this data access pattern possible. They store the same values as the _source but in a **column-oriented fashion that is way more efficient for sorting and aggregations.** 
- norms : 
    - relavance scoring에 사용되는 Normalization factor를 저장할 것인지를 결정.
    - relavance scoring에 사용되지 않을 field들에 대해서 사용하게 되면, disk-space를 줄일 수 있다.
- index:
    - 해당 field에 대한 indexing을 disable하기 위해서 사용한다.
- null_value:
    - ES에서는 원래 null value를 무시함, 따라서 null value를 searchable하게 만들기 위해서 사용.
- copy-to:
    - 여러개의 field 값을 복사하기 위해서 사용한다. 
    (first_name + last_name => full name)

## Updating existing mappings
- ES에서는 field mapping을 변경 할 수 없다.
- 따라서, 새로운 index로 reindexing 해주는 방법을 사용해야 한다.
- reindex : Reindex API

## Multi Field Mappings
- 하나의 filed에 여러개의 type을 지정하는 것.
- 하나의 filed에 대해서 여러개의 anlayzing 방식을 사용 할 수 있도록한다.
- 어느 나라 말로 들어올지 모르는 경우 -> analyzer를 2개를 태워서 가지고 있는다. (한국어(한국어를 위한 analyzer), 영어(영어를 위한 analyzer))

## Dynmaic Mappings
- index없이도 indexing 가능.
- To index a document, you don’t have to first create an index, define a mapping type, and define your fields 
- **you can just index a document and the index, type, and fields will display automatically**

## Index Template
- 말그대로, Index를 만들기 위한 Template을 정의해 놓는 것.
- log 날짜별로 rollup해서 index가 생성될 때, 동일한 Mapping을 사용하기 위함

## Stemming and stop words
### Stemming
- reduces word to root form 
- ex) loved -> love / drinking -> drink
- rule-based이기 때문에 맹신해서는 안된다.
- ex) amusing, amuses, amused -> amus

### Stop words
- text 분석 과정에서 의미 없는 단어들을 날리는 것
- 해당 단어들은 releveance scoring에 별다른 효과가 없다.

![](https://www.googleapis.com/download/storage/v1/b/kaggle-forum-message-attachments/o/inbox%2F4010658%2F47e45441f229db984b7b5cf2ab1c8264%2Fstopwords.jpg?generation=1600940415595842&alt=media)

## Analyzers and search queries

- Analyzing time, Search time에는 같은 analyzer가 쓰여야 한다.
- 따라서, Analyzing time과 Search time에 사용되는 analyzer를 커스터마이징 할 때는 주의를 기울여야 한다.

# Search
- Getting Deeper : [Search in Depth](https://www.elastic.co/guide/en/elasticsearch/guide/current/search-in-depth.html)
- Query DSL로 search 가능

## Search 과정
- coordinating node가 request를 받는다.
- request가 다른 node에 broadcasting한다.
- coordinating node가 결과를 취합해서 client에게 넘긴다.

## Relevance scores
- Relavance scores가 RDB와 가장 큰 차이점을 만든다.

### TF-IDF(Term Frequency / Inverse Document Frequency)

Term Frequency : document의 field에 term이 얼마나 나타났는가?
- Term이 많이 등장 할 수록, relavant score가 높다.

Inverse Document Frequency
- Term이 Index의 다른 Document들에 얼마나 나타나는가?
- Term이 다른 Document들에도 많이 나타난다. -> 검색에 있어서 그리 중요하진 않다.

Field-length norm
- filed가 얼마나 긴가?
- document의 길이가 더 짧은 곳에 나타나는 term의 가중치가 더 높다.

### BM25
- TF-IDF에서는 a, the 같은 그다지 중요하지 않은 term들이 많이 등장하게 되면, term-frequency가 높아진다. 따라서, 빈도수가 일정수준에 도달하면, term-frequency가 높아지지 않도록 한다.

![](https://opensourceconnections.com/wp-content/uploads/2020/05/TF1-2.png)

## Debugging

-> Explain API

## Query Context vs Filter Context
- Filter Context -> Filtering dates, satus, ranges, etc / relavance scoring을 사용하지 않는다.
- Query Context -> Relavance score를 사용한다.

## Full text queries vs Term level queries
- Term level quereis -> exact match ( 따라서, full-text search에서는 사용해서는 안된다. )
- Full text queries -> search query가 analyzing 된다. ( 해당 field에 define된 analyzer를 사용해서 )

## [Boolquery](https://medium.com/elasticsearch/introduction-to-elasticsearch-queries-b5ea254bf455)
query를 **조합**해서 사용
- must
- must_not
- [should](https://esbook.kimjmin.net/05-search/5.4-keyword) 
- filter

# Improving Search Result

## Proximity Search and Slop
- match_phrase -> 엄격한 검색
- match_phrase에서는 term들의 순서가 맞아야지 검색 대상에 포함된다.
- 이러한 제약조건을 완화해서 검색하는 것이 proximity search
- proximity search를 위해서 ES에서는 slop paramter를 제공한다.
- [slop](https://kb.objectrocket.com/elasticsearch/how-to-use-slop-with-phrase-search-in-elasticsearch-6)에는 integer 값을 주고 그 값은 **최대 편집 거리**를 의미한다.

- [관련예제](https://marcobonzanini.com/2015/02/09/phrase-match-and-proximity-search-in-elasticsearch/)

- 'quick fox'로 quick brown fox를 검색하고 싶다면 slop >= 1

```text
            Pos 1         Pos 2         Pos 3
-----------------------------------------------
Doc:        quick         brown         fox
-----------------------------------------------
Query:      quick         fox
Slop 1:     quick                 ↳     fox
```


- 'fox quick'으로 'quick brown fox'를 검색하고 싶다면 slop >= 3
```text
            Pos 1         Pos 2         Pos 3
-----------------------------------------------
Doc:        quick         brown         fox
-----------------------------------------------
Query:      fox           quick
Slop 1:     fox|quick  ↵  
Slop 2:     quick      ↳  fox
Slop 3:     quick                 ↳     fox
```

## Fuzzy match query
- [fuzziness/levenstein distance(편집 거리)](https://en.wikipedia.org/wiki/Levenshtein_distance)
- fuzzy query : term level query (analyzing x)
- 오타 방지
![](https://img-blog.csdnimg.cn/20191014111717533.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1VidW50dVRvdWNo,size_16,color_FFFFFF,t_70)

## Synonym
- Synonym token filter를 적용할 때 순서를 주의!!
- Lowercase token filter보다 앞에 적용시키면, 대문자로 시작하는 term들이 synonym으로 치환이 제대로 안될 수 있다.
- Synonym token filter를 적용하고, Anlayzing이 의도한대로 작동하는지 확인해야 한다.
- file에서 Synonym 추가 가능. (사전식)
- 새로운 synonym이 추가 되면, 검색이 제대로 되지 않는 경우가 있다.

ex) 
- elasticsearch => es인 상태에서
- elasticsearch, elk => es로 바뀌면
- elk를 가지고 있는 document들은 검색안됨. (elk도 es로 바뀌어 검색 되기 때문 - analyzing)

# ELK Stack

**E**lasticsearch, **L**ogstash, **K**ibana
## 1. Elasticsearch

## 2. Logstash
Logstash는 실시간 파이프라인 기능을 가진 오픈소스 데이터 수집 엔진이다.

## 3. Kibana
Kibana는 Elasticsearch에 색인된 데이터를 검색하고 시각화하는 오픈소스 프론트엔드 서비스이다.

**로그 관제 시스템**으로 많이 활용된다.

- why?
로그가 그냥 파일에 쌓여서 관리가 안 되고 용량은 계속 증가해서 나중에 서버에도 악영향을 끼칠 것이라는 문제 

ELK Stack을 활용하면 이 문제를 해결

## ELK Demo

- Kibana 실행 - bin에서 아래 명령어 실행
```bash
./kibana
```

- Logstash 실행 - bin에서 아래 명령어 실행
```bash
./logstash -f ../config/logstash.conf
```
