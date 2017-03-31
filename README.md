## Use Case
You can use this plugin to calculate the relevance score of the two feature vector, such as:
- Personalized Search;
- Find similar product;
- Product recommendation;

## Build Plugin from Code
1. git clone https://github.com/ginobefun/elasticsearch-feature-vector-scoring.git
2. mvn clean package -DskipTests
3. copy target/releases/elasticsearch-feature-vector-scoring-2.3.4.zip to your elasticsearch plugin directory and unzip it
4. restart elasticsearch

## Script Parameters
- **field**: **required**, field in index to store the vector of document;
- **inputFeatureVector**: **required**,  the condition vector, a string connected with comma;
- **version**: version of vector, if it isn't null, it should match the version of vector of document(if use version, the field value should start with ‘$VERSION|’, such as '20170331|0.1,2.3,-1.6,0.7,-1.3');
- **baseConstant** and **factorConstant**: used to calculate the final score, default value are 1. final_score = baseConstant + factorConstant * cos(X, Y)

## Example
### create a test index

    PUT /test
    {
      "mappings": {
        "product": {
          "properties": {
            "productName": {
              "type": "string",
              "analyzer": "standard"
            },
            "productFeatureVector": {
              "type": "string",
              "index": "not_analyzed"
            }
          }
        }
      },
      "settings": {
        "index": {
          "number_of_shards": 1,
          "number_of_replicas": 0
        }
      }
    }

### index some documents

    POST /test/product/1
    {
      "productName": "My favorite brand of shoes",
      "productFeatureVector": "0.1,2.3,-1.6,0.7,-1.3"
    }
    
    POST /test/product/2
    {
      "productName": "Normal brand of shoes",
      "productFeatureVector": "-0.5,1.6,1.1,0.9,0.7"
    }
    
    POST /test/product/3
    {
      "productName": "The shoes I don't like",
      "productFeatureVector": "1.2,0.1,0.4,-0.2,0.3"
    }

### normal search

    POST /test/_search
    {
      "query": {
        "match": {
          "productName": "shoes"
        }
      }
    }

the the result is: 

    {
      "took": 2,
      "timed_out": false,
      "_shards": {
        "total": 1,
        "successful": 1,
        "failed": 0
      },
      "hits": {
        "total": 3,
        "max_score": 0.35615897,
        "hits": [
          {
            "_index": "test",
            "_type": "product",
            "_id": "2",
            "_score": 0.35615897,
            "_source": {
              "productName": "Normal brand of shoes",
              "productFeatureVector": "-0.5,1.6,1.1,0.9,0.7"
            }
          },
          {
            "_index": "test",
            "_type": "product",
            "_id": "1",
            "_score": 0.3116391,
            "_source": {
              "productName": "My favorite brand of shoes",
              "productFeatureVector": "0.1,2.3,-1.6,0.7,-1.3"
            }
          },
          {
            "_index": "test",
            "_type": "product",
            "_id": "3",
            "_score": 0.3116391,
            "_source": {
              "productName": "The shoes I don't like",
              "productFeatureVector": "1.2,0.1,0.4,-0.2,0.3"
            }
          }
        ]
      }
    }

### search with feature vector score

    POST /test/_search
    {
      "query": {
        "function_score": {
          "query": {
            "match": {
              "productName": "shoes"
            }
          },
          "functions": [
            {
              "script_score": {
                "script": {
                  "inline": "feature_vector_scoring_script",
                  "lang": "native",
                  "params": {
                    "inputFeatureVector": "0.1,2.3,-1.6,0.7,-1.3",
                    "field": "productFeatureVector"
                  }
                }
              }
            }
          ],
          "boost_mode": "multiply"
        }
      }
    }

and the result is: 

    {
      "took": 5,
      "timed_out": false,
      "_shards": {
        "total": 1,
        "successful": 1,
        "failed": 0
      },
      "hits": {
        "total": 3,
        "max_score": 0.6232782,
        "hits": [
          {
            "_index": "test",
            "_type": "product",
            "_id": "1",
            "_score": 0.6232782,
            "_source": {
              "productName": "My favorite brand of shoes",
              "productFeatureVector": "0.1,2.3,-1.6,0.7,-1.3"
            }
          },
          {
            "_index": "test",
            "_type": "product",
            "_id": "2",
            "_score": 0.4336441,
            "_source": {
              "productName": "Normal brand of shoes",
              "productFeatureVector": "-0.5,1.6,1.1,0.9,0.7"
            }
          },
          {
            "_index": "test",
            "_type": "product",
            "_id": "3",
            "_score": 0.25049925,
            "_source": {
              "productName": "The shoes I don't like",
              "productFeatureVector": "1.2,0.1,0.4,-0.2,0.3"
            }
          }
        ]
      }
    }
    
### A personalized search case details
see http://ginobefunny.com/post/personalized_search_implemention_based_word2vec_and_elasticsearch/