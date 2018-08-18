# Forward-Search-Installation-Guide
This contains step-by-step guide to set up Forward Search for a particular domain. Input is a URL template for the target domain and output is a web-based entity-semantic search engine for the domain.

# Crawl Data
You need to install Jupyter Notebook to open file `PrepareUrlList.ipynb` in https://github.com/forward-uiuc/Common-Crawl-URL-Searcher. Change the domain in cell 6, e.g., `domain = "forwarddatalab.org"`, and run all cells. If necessary, please install missing packages by using pip, such as `pip install url_normalize`. Then you will find a new CSV file containing all urls corresponding to the domain in the same folder as the notebook file.

# Annotate Data
Clone repository https://github.com/forward-uiuc/Entity-Search-Annotation-Indexing. 

Go to the directory of the code repository you just cloned. Run `mvn package` to compile and create jar file. The repository is tested with Java 8.

Copy the csv created in the step above to folder input in the directory.
Go to this link http://chromedriver.chromium.org/downloads to download the chromedriver compatible with your operating system. Copy the driver file to lib folder in the directory. 

Open `run.sh` to understand what it does. Open terminal and run `sh run.sh $domain $output`, in which `$domain.csv` is the csv file you in the step created above, and `$output` is the output folder of the serialized file. For example, `./run.sh forwarddatalab.org ../output/`.  You should create the output folder `output` ahead. The syntax is fragile so please follow the provided example precisely.

You can check log file in `output/$domain/log` to see errors. You can also run `ps -ef | grep chrome` to see if chrome driver is running. Please note that this code is for production so I set the flag to not show Chrome UI. So you may see it is not running but it is actually running. Check the log files to see updates. For each done URL, an entry is noted on std file in the log folder. For each URL, this step downloads the file, and does expensive annotation, so it is relatively expensive for personal computer. You may expect 1-2 minutes for an URL. For a big domain, it could take days to finish. Finally, please take note that it runs very slow on my Macbook Pro while it is a few times faster on Falcon machine, probably thanks to its superior CPUs and GPUs.

# Create import file for Elastic Search
ElasticSearch allows bulk import from a file https://www.elastic.co/guide/en/elasticsearch/reference/5.6/docs-bulk.html. We just need to follow the format. In the same folder as above, run a command similar with the following:
```
java -cp target/uber-EntityAnnotation-1.0-SNAPSHOT.jar org.forward.entitysearch.experiment.CreateImportFileForElasticSearch "/Users/longpham/Workspace/ForwardSearchInstallationGuide/output/forwarddatalab.org/data" test_es.json
```
It will create file `test_es.json` for import. Please change the command sample above with regard to your own environment.

# Install Elastic Search

Download Elastic Search version 5.6.1 here: https://www.elastic.co/downloads/past-releases 

Optionally download Kibana version 5.6.1 here: https://www.elastic.co/downloads/past-releases Kibana is a friendly GUI of Elastic Search

# Install plugins

Download Entity Elastic Search Analysis Plugin, which encodes our rule to read the import file created above as well as a special tokenizer called layout_tokenizer: https://github.com/forward-uiuc/Entity-Elastic-Search-Analysis-Plugin

Change deployPlugin.sh to reflect the setting of your system (i.e., the path to Elastic Search in your system). And then run it: `./deployPlugin.sh`, which both compiles and installs the plugin.

Download Entity Elastic Search API Extension Plugin, which translates hashtag query such as #professor data mining into the format Elastic Search can understand https://github.com/forward-uiuc/Entity-Elastic-Search-API-Extension-Plugin

Change deployPlugin.sh to reflect the setting of your system (i.e., the path to Elastic Search in your system). And then run it: `./deployPlugin.sh`, which both compiles and installs the plugin.

# Run Elastic Search

Run Elastic Search by `bin/elasticsearch` from Elastic Search folder

Run Kibana by `bin/kibana/` from Kibana folder

Please note that whenever you import plugin, you need to restart Elastic Search.

# Create schema for Index

In order for our plugin to work, the index needs to have a schema.

Open query console on Kibana: localhost:5601/app/kibana#/dev_tools/

Run the following json query:

```
PUT /test_annotation/
{
  "mappings": {
    "document": {
      "properties": {
        "text": {
          "type": "text",
          "term_vector": "with_positions_offsets_payloads",
          "store": true,
          "analyzer": "fulltext_analyzer"
        }
      },
      "dynamic_templates": [
        {
          "entity_type": {
            "match_mapping_type": "string",
            "match": "_entity_*",
            "mapping": {
              "type": "text",
              "term_vector": "with_positions_offsets_payloads",
              "store": true,
              "analyzer": "entity_analyzer"
            }
          }
        },
        {
          "layout_type": {
            "match_mapping_type": "string",
            "match": "_layout_*",
            "mapping": {
              "type": "text",
              "term_vector": "with_positions_offsets_payloads",
              "store": true,
              "analyzer": "entity_analyzer"
            }
          }
        }
      ]
    }
  },
  "settings": {
    "index": {
      "number_of_shards": 1,
      "number_of_replicas": 0
    },
    "analysis": {
      "analyzer": {
        "entity_analyzer": {
          "type": "custom",
          "tokenizer": "layout_tokenizer"
        },
        "fulltext_analyzer": {
          "type": "custom",
          "tokenizer": "layout_tokenizer",
          "filter": [
            "lowercase"
          ]
        }
      }
    }
  }
}
```
The query above creates an index called test_annotation with custom tokenizer "layout_tokenizer" which is defined in Entity Elastic Search Analysis Plugin. It also uses dynamic schema. You may need to read Elastic Search tutorial to understand dynamic schema.

# Bulk import data
Now we can import data from the json data file generated above. Go to the folder of Entity Search Annotation Indexing and run the following command:

```
curl -XPOST localhost:9200/test_annotation/_bulk -H ‘Content-Type: application/json’ --data-binary @test_es.json
```

It will import a few pages we crawled before to the Elastic Search. To test, run the following json query on Kibana:

```
GET test_annotation/_es_document_search?
{
  "search_request":{
    "query": "@near [ true 1000 ] ( #person data ) @near [ true 1000 5 ] ( #person mining )",
    "size" : 1000
  }
}
```

You should receive one page result. The meaning of the query above is searching for document where #person is near data and order matters, and in the same document, #person is near mining and order matters.

At this step, we already have the Search Engine. The step below is to create the web interface.

# Install Web Interface

## Summary
This user interface uses [nodejs](https://nodejs.org/en/) as the server. You need to install [nodejs](https://nodejs.org/en/) first. The backend simply wraps ElasticSearch call. The frontend supports both entity search and entity-semantic document search. Users can click a button to switch between the two. When typing # (hash token) in search input box, the system will autosuggest available entity types.

To understand the code, you need background in [nodejs](https://www.w3schools.com/nodejs/default.asp), [reactjs](https://reactjs.org/tutorial/tutorial.html) and [react semantic ui](https://react.semantic-ui.com/introduction). Start reading code at [index.jsx](https://github.com/forward-uiuc/Spring-2018-Entity-Search/blob/master/entity-search-web-interface/frontend/source/index.jsx) and trace other components from there.

## How to install
At the folder containing this README:
```
cd backend
npm install
```

```
cd frontend
npm install
```

## How to start
At the folder containing this README:
```
cd backend
npm start &
```

* If listen locally (localhost:8080):
```
cd frontend
npm run dev
```
* If listen publicly (0.0.0.0:8080):
```
cd frontend
npm start &
```

## Troubleshooting
* No error but cannot open the webpage with a browser
  * Clear the cache, or use Incognito mode, or switch to another browser. Will fix this issue soon.
* Error EADDRINUSE:::xxxx means the port xxxx is in use (there is already a running server process)
  * To see if there is node/nodemon process running: 
  ``` aux ps | grep node ```
  * To kill other node processes: 
  ``` killall node ```
  * Now you should be able to start servers normally.

# Create Screenshots for Web Interface

Go to Entity Search Annotation Indexing folder, and run the following command to create Screenshots (make folder public inside folder backend in the Interface folder if necessary)

```
java -cp target/uber-EntityAnnotation-1.0-SNAPSHOT.jar org.forward.entitysearch.experiment.CreateScreenshots "/Users/longpham/Workspace/ForwardSearchInstallationGuide/output/forwarddatalab.org/data/" "/Users/longpham/Workspace/ForwardSearchInstallationGuide/Entity-Semantic-Document-Search-Web-Interface/backend/public/"
```

# Test

Now, the web-based search engine is ready at localhost:8080

# Notes
* Currently, the backend for web interface runs on port 1720, which is open to public. However, for some reason, if using UIUC VPN, that port cannot be used. Thus, please not use VPN when trying the demo.
* There is a live demo on http://harrier08.cs.illinois.edu:8080





