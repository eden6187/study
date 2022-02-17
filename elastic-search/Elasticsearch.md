# Elastic Search 학습 내용 정리

## Architecture

- Cluster : Node들로 구성
- Node : 하나의 Elasticsearch server
- Index : Document를 저장
- Document : Index에 저장되는 데이터

![architecture](https://media.vlpt.us/images/jwpark06/post/da9ebf2d-a3d2-4f30-ae66-7a0ed85fc168/image.png)

출처 : https://velog.io/@jwpark06/Elasticsearch-%EC%8B%9C%EC%8A%A4%ED%85%9C-%EA%B5%AC%EC%A1%B0-%EC%95%8C%EC%95%84%EB%B3%B4%EA%B8%B0

### Sharding
- DB의 샤딩과 유사한 개념
- Index를 여러개의 Node에 분산해 놓는 것
- Sharding을 통해서 대용량의 데이터도 분산해서 저장 가능. (Scalability)
- 각 shard는 Apache Lucene Index
- Shard를 통해서 Index의 throuput 증가 가능 -> 성능 향상의 목적도 가지고 있음. 
(Performance)

### Replication
- Cluster 내부의 Node에 문제가 생긴다면, Node가 가지고 있는 Index들에도 문제가 발생하게 된다.
- Elasticsearch는 이러한 문제를 방지하기 위해서 Replication을 지원한다. (For fault tolerance)
- Elasticsearch에서는 replication이 기본적으로 지원된다.
- Replication도 Sharding과 마찬가지로 Index 단위로 지원되고, 각각의 shard들을 복제한다.
- 복제된 shard의 원본을 primary shard라고 한다.
- shard의 사본을 replica shard 라고 한다. 
- primary shard + replica shard -> replication group
- replica shard는 primary shard 와 같은 node에 저장되지 않는다.

## Node roles
### 1. Master-eligible
- Master node가 될 수 있음을 의미.
### 2. Data
- query 실행
### 3. Ingest
- node가 ingest pipeline을 수행하도록 할 수 있다
>  Ingesting : index에 document를 추가 하는 것.
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

- ARS(Adaptive Replica Selection)는 Replication Group 중에서 최적(가장 빨리 읽어 올 수 있는?)의 shard를 찾아준다.

# Analyzing and Mappings

## Analysis

- document의 text field는 indexing 과정에서 검색에 최적하된 형태로 저장된다.
- 그러한 과정을 Analysis라고 한다.

## Analyze API

- Custom Anayzer를 만들고, 실제로 Document에 적용해보기 전에 유용.

## Analyzer

- 3가지 구성요소로 이루어짐.
- 관련 API : Analyze API

### Character Filters

- original string을 받아서 add, remove, change characters.

### Tokenizer

- character filtering이 끝난 string을 tokenizing.

### Token filters

- token들을 받아서 수정.

## Inverted Index

- term을 기준으로, 해당 term을 가지고 있는 document들을 구분
- relavance scoring을 위한 정보들도 가지고 있다.
- Index의 각각의 Filed 별로 Inverted Index가 만들어진다.
- text field가 아닌 field들은 다른 자료구조를 쓴다. 

![inverted index](https://habrastorage.org/webt/5e/f9/f0/5ef9f0a04c3f6781438716.jpeg)

출처 : https://habrastorage.org/webt/5e/f9/f0/5ef9f0a04c3f6781438716.jpeg

# Mappings

ES에는 2가지 Mapping 전략이 있음.

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

### 3. Keyword Type

- exact matching을 위해서 사용한다.
- filtering, aggregations, sorting에서 사용한다.
- full-text search를 사용하고 싶다면, text type을 사용한다.
- keyword field는 keyord analyzer에 의해서 analyze된다.
- keyword analyzer는 origin string을 그대로 반환한다.

## Type coersion

- 일종의 자동 형변환?
- defalut로 활성화 되어 있지만, 안정적인 type만 받기를 원한다면, disable 하는 것도 방법이다. 

## Array Data Type in Elastic Search
- Elasticsearch에는 array가 존재하지 않는다.
대신, 입력받은 값들을 concatination해서 저장한다.
- array 내부에 있는 object들을 독립적으로 다루고자 한다면, nested type을 사용하자!

## Date Type
- epoch 이후 몇 millisecond가 지났는지를 기준으로 long 형태로 저장된다.
unix timestamp와 혼용해서 사용하지 않도록 주의해야한다.

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
- norms : 
    - relavance scoring에 사용되는 Normalization factor를 저장할 것인지를 결정.
    - relavance scoring에 사용되지 않을 field들에 대해서 사용하게 되면, disk-space를 줄일 수 있다.
- index:
    - 해당 field에 대한 indexing을 disable하기 위해서 사용한다.
- null_value:
    - ES에서는 원래 null value를 무시하시함, 따라서 null value를 searchable하게 만들기 위해서 사용.
- copy-to:
    - 여러개의 field 값을 복사하기 위해서 사용한다.

## Updating existing mappings

- ES에서는 field mapping을 변경 할 수 없다.
- 따라서, 새로운 index로 reindexing 해주는 방법을 사용해야 한다.
- reindex : Reindex API

## Multi Field Mappings

- 하나의 filed에 여러개의 type을 지정하는 것.
- 하나의 filed에 대해서 여러개의 anlayzing 방식을 사용 할 수 있도록한다.
- 유용한 기능이라고 한다...??


## Index Template

- 말그대로, Index를 만들기 위한 Template을 정의해 놓는 것.

## ECS - Elastic Common Schema

## Dynmaic Mappings

## Stemming and stop words

### Stemming

- reduces word to root form 
- ex) loved -> love / drinking -> drink
- Snowball token filter

### Stop words

- text 분석 과정에서 의미 없는 단어들을 날리는 것
- 해당 단어들은 releveance scoring에 별다른 효과가 없다.

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

Term Frequency : document의 field에 term이 얼마나 나타났는가?

- Term이 많이 등장 할 수록, relavant score가 높다.

Inverse Document Frequency

- Term이 Index 내부의 다른 Document들에 얼마나 나타나는가?

Field-length norm

- filed가 얼마나 긴가?

## Debugging

-> Explain API

## Query Context vs Filter Context
- Filter Context -> Filtering dates, satus, ranges, etc / relavance scoring을 사용하지 않는다.
- Query Conetext -> Relavance score를 사용한다.

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
- [slop](https://kb.objectrocket.com/elasticsearch/how-to-use-slop-with-phrase-search-in-elasticsearch-6)에는 integer 값을 주고 그 값은 최대 편집 거리를 의미한다.

- [관련예제](https://marcobonzanini.com/2015/02/09/phrase-match-and-proximity-search-in-elasticsearch/)

## Fuzzy match query
- [fuzziness/levenstein distance(편집 거리)](https://en.wikipedia.org/wiki/Levenshtein_distance)
- fuzzy query : term level query (analyzing x)

## Synonym
- Synonym token filter를 적용할 때 순서를 주의!!
- Lowercase token filter보다 앞에 적용시키면, 대문자로 시작하는 term들이 synonym으로 치환이 제대로 안될 수 있다.
- Synonym token filter를 적용하고, Anlayzing이 의도한대로 작동하는지 확인해야 한다.
- file에서 Synonym 추가 가능.
- 새로운 synonym이 추가 되면, 검색이 제대로 되지 않는 경우가 있다.

ex) 
- elasticsearch => elk
- elasticsearch를 가지고 있는 document들은 검색안됨.
- query에 있는 "elasticsearch"라는 term들은 모두 "elk"로 바뀌기 때문이다.
- 해결챌 : [Update By Query API](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-update-by-query.html)


