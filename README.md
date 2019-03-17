---

ES2
===

---

<br><br>

1.Elasticsearch Internal
------------------------

-	**refresh_interval** 설정은 얼마나 자주 Lucene flush가 발생하는지에 대해 정의한다. 말인즉, 얼마나 자주 각각의 샤드가 in-memory buffer로 부터 document를 제거하는지 그리고 그것들을 포함하는 segment를 만드는지를 의미한다. search의 관점에서, document가 index operation후에 검색이 불가능한 최대 시간을 의미한다.

###### 3 priamry shards, 0 replica shard, refresh interval 1 hour를 가지고 **my_refresh_test** index를 만들어 보자

<details><summary> 정답 </summary>

```shell
PUT my_refresh_test
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 0,
    "refresh_interval": "1h"
  }
}
```

</details>

<br>

-	다음의 경로로 들어가보자

```shell
cd /var/lib/elasticsearch/nodes/0/indices
ls

# uuid로 나열된 index가 보인다.
```

<br>

###### kibana console에서 cat command를 사용해서 **my_refresh_test** index를 확인해보자

<details><summary> 정답 </summary>

```shell
GET _cat/indices/my_refresh_test?v

# output
health status index           uuid                   pri rep docs.count docs.deleted store.size pri.store.size
green  open   my_refresh_test FxzYyrITThitG8CdR-Pirg   3   0          0            0       783b           783b
```

</details>

<br><br>

-	첫번째 노드에서 uuid확인하여 **my_refresh_test** index의 경로로 들어가보자

```shell
# example
cd FxzYyrITThitG8CdR-Pirg/
ls

# output
2  _state
```

-	위의 output은 shard 2가 해당 노드에 위치해 있다는 것을 확인 할 수 있다.

-	kibana console에서 cat command를 사용해서 확인할 수 있다.

```shell
GET _cat/shards/my_refresh_test?v&s=node

# response
index           shard prirep state   docs store ip         node
my_refresh_test 2     p      STARTED    0  261b 10.142.0.4 itmare-hot01
my_refresh_test 0     p      STARTED    0  261b 10.142.0.5 itmare-hot02
my_refresh_test 1     p      STARTED    0  261b 10.142.0.6 itmare-hot03
```

<br>

###### **my_refresh_test** index의 replica 수를 1로 변경해보고, 첫노드의 shard folder에 있는 것들을 확인해보자.

<details><summary> 정답 </summary>

```shell
PUT my_refresh_test/_settings
{
    "number_of_replicas": 1
}
```

```shell
cd /var/lib/elasticsearch/nodes/0/indices
ls
```

<img src="./pictures/internal-shards.png">

-	각 노드별로 shard의 갯수가 늘어난 걸 확인 할 수 있다.

</details>

<br>

###### cat API를 통해 shard확인 해보자

<details><summary> 정답 </summary>

```shell
# node, shard 이름 순으로 정렬
GET _cat/shards/my_refresh_test?v&s=node,shard
```

</details>

<br>

###### 터미널에서 첫번쨰 노드의 shard 0d으로 들어가 보자

```shell
cd 0
ls
_state  index  translog
```

<br>

###### cd로 다시 index directory로 들어가보자

```shell
cd index
ls
segments_4  write.lock
```

-	해당 shard 경로에서 segment_x와 write.lock 파일을 볼 수 있다.

<br>

###### 데이터를 추가해보자

```
PUT my_refresh_test/_doc/_bulk
{ "index" : { "_id" : "1"}}
{ "level" : "test"}
{ "index" : { "_id" : "2"}}
{ "level" : "test"}
{ "index" : { "_id" : "8"}}
{ "level" : "test"}
```

<br>

###### search request를 실행했을때 몇개의 document가 리턴되고 왜 그럴까?

```shell
GET my_refresh_test/_search
```

<details><summary> 정답 </summary>

```shell
{
  "took" : 4,
  "timed_out" : false,
  "_shards" : {
    "total" : 3,
    "successful" : 3,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : 0,
    "max_score" : null,
    "hits" : [ ]
  }
}
```

-	0개가 리턴된다. (hits.total => 0), 현재 refresh_interval이 1시간으로 설정되어 있다. 그래서 Lucene flush가 발생하고 document가 저장될 segment가 생성 될때까지 최대 1시간이 걸린다.

</details>

<br>

###### document 1을 호출하기 위해 GET request를 보내보자.

```shell
GET my_refresh_test/_doc/1
```

<details><summary> 정답 </summary> docuemnt 1은 리턴된다. 기본적으로 GET API는 실시간이다. index의 refresh rate에 영향을 받지 않는다.</details>

<br>

###### force refresh를 해보고, 다시 검색을 해보자 결과는? (hit의 갯수는?)

```shell
POST my_refresh_test/_refresh
```

<details><summary> 정답 </summary>

```shell
{
  "took" : 13,
  "timed_out" : false,
  "_shards" : {
    "total" : 3,
    "successful" : 3,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : 3,
    "max_score" : 1.0,
    "hits" : [
      {
        "_index" : "my_refresh_test",
        "_type" : "_doc",
        "_id" : "2",
        "_score" : 1.0,
        "_source" : {
          "level" : "test"
        }
      },
      {
        "_index" : "my_refresh_test",
        "_type" : "_doc",
        "_id" : "8",
        "_score" : 1.0,
        "_source" : {
          "level" : "test"
        }
      },
      {
        "_index" : "my_refresh_test",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 1.0,
        "_source" : {
          "level" : "test"
        }
      }
    ]
  }
}
```

-	3개의 hit를 확인할 수 있다. `_refresh`는 in-memory buffer를 비우기 위해, 그리고 새로운 segment를 만들기 위한 lucene flush를 작동시킨다.

</details>

<br>

###### 다음과 같이 shard 경로안에 파일들이 생성된 것을 확인 할 수 있다.

<img src="./pictures/internal-force_refresh.png">[참고: Lucene Documentation](https://lucene.apache.org/core/2_9_4/fileformats.html#file-names)

<br>

###### 더 많은 데이터를 추가해보자

```
PUT my_refresh_test/_doc/_bulk
{ "index" : { "_id" : "3"}}
{ "level" : "test"}
{ "index" : { "_id" : "4"}}
{ "level" : "test"}
{ "index" : { "_id" : "14"}}
{ "level" : "test"}
```

-	shard 경로 확인 `_1.fdt`와 `_1.fdx`가 추가 되었다.

<img src="./pictures/internal-froce_refresh2.png">

<br><br>

###### 다시 refresh하고 shard경로를 다시 확인해 보자

```shell
POST my_refresh_test/_refresh
```

<img src="./pictures/internal-froce_refresh3.png">

<br><br>

###### 현재 2개의 segment를 가지고 있다. **my_refresh_test** index의 **_forcemerge** 를 실행해보고, shard 경로를 확인 해보자.

```shell
POST my_refresh_test/_forcemerge
```

-	아무런 옵션없는 forcemerge는 Lucene이 상황에 따라서 merge를 결정한다. 현재 2개의 아주 작은 segment를 가지고 있어서 Lucene은 merge하지 않기로 결정한 것으로 보인다.

<br>

###### **max_num_segments** parameter를 추가하여 다시한번 formerge 해보자.

```shell
POST my_refresh_test/_forcemerge?max_num_segments=1
```

<img src="./pictures/internal-forcemerge.png">

-	`_0`과 `_1`로 시작하는 파일은 사라지고, `_2`로 시작하는 많은 파일들이 생긴다.

<br>

###### 현재 refresh_interval은 1시간으로 설정되어 있다. 때로는 즉각 refresh 필요한 문서가 있을지도 모른다. 이럴 경우, document를 추가할때 refresh option을 true로 설정하고 document를 추가해보자

```shell
PUT my_refresh_test/_doc/5?refresh=true
{
  "level":"test"
}

GET my_refresh_test/_search
```

-	refresh_interval이 1시간이지만 document가 바로 추가되는 것을 확인 할 수 있다.

<br>

###### 때로는 사용자가 response를 받고 싶을 수도 있다. refresh의 wait_for option을 사용해서, refresh_interval을 10초로 변경해보고, document를 추가해서 기다림을 확인해보자.

<details><summary> 정답 </summary>

```shell
PUT my_refresh_test/_settings
{
  "refresh_interval": "10s"
}

PUT my_refresh_test/_doc/6?refresh=wait_for
{
  "level":"waiting test"
}
```

</details>

<br>

###### **my_refresh_test** index를 삭제하자

```shell
DELETE my_refresh_test
```

<br><br><br><br><br>

2.Field Modeling
----------------

3.Fixing Data
-------------

4.Advanced Search & Aggregations
--------------------------------

5.Cluster Management
--------------------

6.Capacity Planning
-------------------

7.Document Modeling
-------------------

8.Monitoring and Alerting
-------------------------

9.From Dev to Production
------------------------

.

.

.

.

.

.

.

.

.

.

.

.

.

.

.

.

.

.

---

Exam Objectives
---------------

<br><br>

To be fully prepared for the Elastic Certified Engineer exam, candidates should be able to complete all of the following exam objectives with only the assistance of the [Elastic documentation](https://www.elastic.co/guide/index.html):

<br>

#### Installation and Configuration

-	Deploy and start an Elasticsearch cluster that satisfies a given set of requirements
-	Configure the nodes of a cluster to satisfy a given set of requirements
-	Secure a cluster using Elasticsearch Security ([doc](https://www.elastic.co/guide/en/elasticsearch/plugins/current/security.html)\)
-	Define role-based access control using Elasticsearch Security ([doc](https://www.elastic.co/guide/en/kibana/current/development-security-rbac.html)\)

#### Indexing Data

-	Define an index that satisfies a given set of requirements

-	Perform index, create, read, update, and delete operations on the documents of an index

-	Define and use index aliases

-	Define and use an index template for a given pattern that satisfies a given set of requirements

-	Define and use a dynamic template that satisfies a given set of requirements

-	Use the Reindex API and Update By Query API to reindex and/or update documents

-	Define and use an ingest pipeline that satisfies a given set of requirements, including the use of Painless to modify documents

#### Queries

-	Write and execute a search query for terms and/or phrases in one or more fields of an index

-	Write and execute a search query that is a Boolean combination of multiple queries and filters

-	Highlight the search terms in the response of a query

-	Sort the results of a query by a given set of requirements

-	Implement pagination of the results of a search query

-	Use the scroll API to retrieve large numbers of results

-	Apply fuzzy matching to a query

-	Define and use a search template

-	Write and execute a query that searches across multiple clusters

#### Aggregations

-	Write and execute metric and bucket aggregations

-	Write and execute aggregations that contain sub-aggregations

-	Write and execute pipeline aggregations

#### Mappings and Text Analysis

-	Define a mapping that satisfies a given set of requirements

-	Define and use a custom analyzer that satisfies a given set of requirements

-	Define and use multi-fields with different data types and/or analyzers

-	Configure an index so that it properly maintains the relationships of nested arrays of objects

-	Configure an index that implements a parent/child relationship

#### Cluster Administration

-	Allocate the shards of an index to specific nodes based on a given set of requirements

-	Configure shard allocation awareness and forced awareness for an index

-	Diagnose shard issues and repair a cluster’s health

-	Backup and restore a cluster and/or specific indices

-	Configure a cluster for use with a hot/warm architecture

-	Configure a cluster for cross cluster search
