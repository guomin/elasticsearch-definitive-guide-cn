## 检索文档
现在我们的Elasticsearch中已经存储了一些数据，我们可以根据业务需求开始进行工作了。第一个需求是能够检索单个员工的信息。

这对于Elasticsearch非常简单。我们只要执行HTTP GET请求并指出文档的“地址”——索引、类型和ID既可。使用这三部分，就可以返回JSON原始文档：

```Jacscript
GET /megacorp/employee/1
```

响应的内容中包含一些文档的元数据信息，John Smith的原始JSON文档做为`_source`字段存在。

```Javascript
{
  "_index" :   "megacorp",
  "_type" :    "employee",
  "_id" :      "1",
  "_version" : 1,
  "found" :    true,
  "_source" :  {
      "first_name" :  "John",
      "last_name" :   "Smith",
      "age" :         25,
      "about" :       "I love to go rock climbing",
      "interests":  [ "sports", "music" ]
  }
}
```

****

同样的方式，我们通过把HTTP方法`PUT`改为`GET`来检索文档，我们可以使用`DELETE`方法删除文档，使用`HEAD`方法检查文档是否存在。如果为了替换已存在的文档，我们只需使用`PUT`方法即可。

****

## 简单搜索
`GET`请求确实足够简单——你想要什么，它就可以返回什么文档。让我们来看一些更复杂的东西，就像简单搜索！

The first search we will try is the simplest search possible.  We will search
for all employees, with this request:

我们尝试一个最简单的搜索全部员工的请求：

```Javascript
GET /megacorp/employee/_search
```

你可以看到我们依旧使用`megacorp`索引和`employee`类型，但是与使用指定ID的文档不同，现在使用`_search`端点。响应内容的`hits`数组中包含了所有的三个文档。一般情况下搜索会返回前10个结果。

```Javascript
{
   "took":      6,
   "timed_out": false,
   "_shards": { ... },
   "hits": {
      "total":      3,
      "max_score":  1,
      "hits": [
         {
            "_index":         "megacorp",
            "_type":          "employee",
            "_id":            "3",
            "_score":         1,
            "_source": {
               "first_name":  "Douglas",
               "last_name":   "Fir",
               "age":         35,
               "about":       "I like to build cabinets",
               "interests": [ "forestry" ]
            }
         },
         {
            "_index":         "megacorp",
            "_type":          "employee",
            "_id":            "1",
            "_score":         1,
            "_source": {
               "first_name":  "John",
               "last_name":   "Smith",
               "age":         25,
               "about":       "I love to go rock climbing",
               "interests": [ "sports", "music" ]
            }
         },
         {
            "_index":         "megacorp",
            "_type":          "employee",
            "_id":            "2",
            "_score":         1,
            "_source": {
               "first_name":  "Jane",
               "last_name":   "Smith",
               "age":         32,
               "about":       "I like to collect rock albums",
               "interests": [ "music" ]
            }
         }
      ]
   }
}
```

>**注意**：响应内容不仅会告诉我们哪些文档被匹配，而且这些文档内容会被包含其中：所有的信息我们都需要显示搜索结果给用户。

Next, let's try searching for employees who have ``Smith'' in their last name.
To do this, we'll use a ``lightweight'' search method which is easy to use
from the command line. This method is often referred to as a _query string_
search, since we pass the search as a URL query string parameter:

[source,js]
--------------------------------------------------
GET /megacorp/employee/_search?q=last_name:Smith
--------------------------------------------------
// SENSE: 010_Intro/30_Simple_search.json

We use the same `_search` endpoint in the path, and we add the query itself in
the `q=` parameter. The results that come back show all Smith's:

[source,js]
--------------------------------------------------
{
   ...
   "hits": {
      "total":      2,
      "max_score":  0.30685282,
      "hits": [
         {
            ...
            "_source": {
               "first_name":  "John",
               "last_name":   "Smith",
               "age":         25,
               "about":       "I love to go rock climbing",
               "interests": [ "sports", "music" ]
            }
         },
         {
            ...
            "_source": {
               "first_name":  "Jane",
               "last_name":   "Smith",
               "age":         32,
               "about":       "I like to collect rock albums",
               "interests": [ "music" ]
            }
         }
      ]
   }
}
--------------------------------------------------

=== Search with Query DSL

Query-string search is handy for _ad hoc_ searches from the command line, but
it has its limitations (see <<search-lite>>). Elasticsearch provides a rich,
flexible, query language called the _Query DSL_, which allows us to build
much more complicated, robust queries.

The DSL (_Domain Specific Language_) is specified using a JSON request body.
We can represent the previous search for all Smith's like so:


[source,js]
--------------------------------------------------
GET /megacorp/employee/_search
{
    "query" : {
        "match" : {
            "last_name" : "Smith"
        }
    }
}
--------------------------------------------------
// SENSE: 010_Intro/30_Simple_search.json

This will return the same results as the previous query.  You can see that a
number of things have changed.  For one, we are no longer using _query string_
parameters, but instead a request body.  This request body is built with JSON,
and uses a `match` query (one of several types of queries, which we will learn
about later).

=== More complicated searches

Let's make the search a little more complicated.  We still want to find all
employees with a last name of ``Smith'', but  we only want employees who are
older than 30.  Our query will change a little to accommodate a _filter_,
which allows us to execute structured searches efficiently:

[source,js]
--------------------------------------------------
GET /megacorp/employee/_search
{
    "query" : {
        "filtered" : {
            "filter" : {
                "range" : {
                    "age" : { "gt" : 30 } <1>
                }
            },
            "query" : {
                "match" : {
                    "last_name" : "smith" <2>
                }
            }
        }
    }
}
--------------------------------------------------
// SENSE: 010_Intro/30_Query_DSL.json

<1> This portion of the query is a `range` _filter_, which will find all ages
    older than 30 -- `gt` stands for ``greater than''.
<2> This portion of the query is the same `match` _query_ that we used before.

Don't worry about the syntax too much for now, we will cover it in great
detail later on.  Just recognize that we've added a _filter_ which performs a
range search, and reused the same `match` query as before.  Now our results
only show one employee who happens to be 32 and is named ``Jane Smith'':

[source,js]
--------------------------------------------------
{
   ...
   "hits": {
      "total":      1,
      "max_score":  0.30685282,
      "hits": [
         {
            ...
            "_source": {
               "first_name":  "Jane",
               "last_name":   "Smith",
               "age":         32,
               "about":       "I like to collect rock albums",
               "interests": [ "music" ]
            }
         }
      ]
   }
}
--------------------------------------------------

=== Full-text search

The searches so far have been simple:  single names, filtering by age. Let's
try a more advanced, full-text search -- a task which traditional databases
would really struggle with.

We are going to search for all employees who enjoy ``rock climbing'':

[source,js]
--------------------------------------------------
GET /megacorp/employee/_search
{
    "query" : {
        "match" : {
            "about" : "rock climbing"
        }
    }
}
--------------------------------------------------
// SENSE: 010_Intro/30_Query_DSL.json

You can see that we use the same `match` query as before to search the `about`
field for ``rock climbing''. We get back two matching documents:

[source,js]
--------------------------------------------------
{
   ...
   "hits": {
      "total":      2,
      "max_score":  0.16273327,
      "hits": [
         {
            ...
            "_score":         0.16273327, <1>
            "_source": {
               "first_name":  "John",
               "last_name":   "Smith",
               "age":         25,
               "about":       "I love to go rock climbing",
               "interests": [ "sports", "music" ]
            }
         },
         {
            ...
            "_score":         0.016878016, <1>
            "_source": {
               "first_name":  "Jane",
               "last_name":   "Smith",
               "age":         32,
               "about":       "I like to collect rock albums",
               "interests": [ "music" ]
            }
         }
      ]
   }
}
--------------------------------------------------
<1> The relevance scores.

By default, Elasticsearch sorts matching results by their relevance score,
that is: by how well each document matched the query.  The first and highest
scoring result is obvious: John Smith's `about` field clearly says ``rock
climbing'' in it.

But why did Jane Smith, come back as a result?  The reason her document was
returned is because the word ``rock'' was mentioned in her `about` field.
Because only ``rock'' was mentioned, and not ``climbing'', her `_score` is
lower than John's.

This is a good example of how Elasticsearch can search *within* full text
fields and return the most relevant results first. This concept of _relevance_
is important to Elasticsearch, and is a concept that is completely foreign to
traditional relational databases where a record either matches or it doesn't.

=== Phrase search

Finding individual words in a field is all well and good, but sometimes you
want to match exact sequences of words or _phrases_. For instance, we could
perform a query that will only match  employees that contain both  ``rock''
_and_ ``climbing'' _and_ where the words are next to each other in the phrase
``rock climbing''.

To do this, we use a slight variation of the `match` query called the
`match_phrase` query:

[source,js]
--------------------------------------------------
GET /megacorp/employee/_search
{
    "query" : {
        "match_phrase" : {
            "about" : "rock climbing"
        }
    }
}
--------------------------------------------------
// SENSE: 010_Intro/30_Query_DSL.json

Which, to no surprise, returns only John Smith's document:

[source,js]
--------------------------------------------------
{
   ...
   "hits": {
      "total":      1,
      "max_score":  0.23013961,
      "hits": [
         {
            ...
            "_score":         0.23013961,
            "_source": {
               "first_name":  "John",
               "last_name":   "Smith",
               "age":         25,
               "about":       "I love to go rock climbing",
               "interests": [ "sports", "music" ]
            }
         }
      ]
   }
}
--------------------------------------------------

[[highlighting-intro]]
=== Highlighting our searches

Many applications like to _highlight_ snippets of text from each search result
so that the user can see *why* the document matched their query.  Retrieving
highlighted fragments is very easy in Elasticsearch.

Let's rerun our previous query, but add a new `highlight` parameter:

[source,js]
--------------------------------------------------
GET /megacorp/employee/_search
{
    "query" : {
        "match_phrase" : {
            "about" : "rock climbing"
        }
    },
    "highlight": {
        "fields" : {
            "about" : {}
        }
    }
}
--------------------------------------------------
// SENSE: 010_Intro/30_Query_DSL.json

When we run this query, the same hit is returned as before, but now we get a
new section in the response called `highlight`.  This contains a snippet of
text from the `about` field with the matching words wrapped in `<em></em>`
HTML tags:

[source,js]
--------------------------------------------------
{
   ...
   "hits": {
      "total":      1,
      "max_score":  0.23013961,
      "hits": [
         {
            ...
            "_score":         0.23013961,
            "_source": {
               "first_name":  "John",
               "last_name":   "Smith",
               "age":         25,
               "about":       "I love to go rock climbing",
               "interests": [ "sports", "music" ]
            },
            "highlight": {
               "about": [
                  "I love to go <em>rock</em> <em>climbing</em>" <1>
               ]
            }
         }
      ]
   }
}
--------------------------------------------------

<1> The highlighted fragment from the original text.

You can read more about the highlighting of search snippets in the
{ref}search-request-highlighting.html[highlighting reference documentation].
