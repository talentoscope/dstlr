# dstlr

`dstlr` is an open-source platform for scalable, end-to-end knowledge graph construction from unstructured text. The platform takes a collection of documents, extracts mentions and relations to populate a raw knowledge graph, links mentions to entities in Wikidata, and then enriches the knowledge graph with facts from Wikidata.
See [`dstlr.ai`](http://dstlr.ai/) for an overview of the platform.

The current `dstlr` demo "distills" the [TREC Washington Post Corpus](https://trec.nist.gov/data/wapost/) containing around 600K documents into a raw knowledge graph comprised of approximately 97M triples, enriched with facts from Wikidata for the 324K distinct entities discovered in the corpus.
On top of this knowledge graph, we have implemented a subgraph-matching approach to align extracted relations with facts from Wikidata using the declarative Cypher query language.
This simple demo shows that fact verification, locating textual support for asserted facts, detecting inconsistent and missing facts, and extracting distantly-supervised training data can all be performed within the same framework.
 
This README provies instructions on how to replicate our work.

# Setup

Clone [dstlr](https://github.com/dstlry/dstlr):

```
git clone https://github.com/dstlry/dstlr.git
```

[sbt](https://www.scala-sbt.org/) is the build tool used for Scala projects, download it if you don't have it yet.

Build the JAR using sbt:

```
sbt assembly
````

There is a [known issue](https://github.com/stanfordnlp/CoreNLP/issues/556) between recent Spark versions and CoreNLP 3.8. To fix this, delete the `protobuf-java-2.5.0.jar` file in `$SPARK_HOME/jars` and replace it with [version 3.0.0](https://repo1.maven.org/maven2/com/google/protobuf/protobuf-java/3.0.0/protobuf-java-3.0.0.jar).

## Anserini

### Download and build Anserini

Clone [Anserini](http://anserini.io):

```
git clone https://github.com/castorini/anserini.git

cd anserini
```

Build Anserini using Maven:

```
mvn clean package appassembler:assemble
```

### Setting up a SolrCloud Instance for indexing text documents

From the Solr [archives](https://archive.apache.org/dist/lucene/solr/), find the Solr version that matches Anserini's [Lucene version](https://github.com/castorini/anserini/blob/master/pom.xml#L36), download the `solr-[version].tgz` (non `-src`), and move it into the `anserini/` directory.

Extract the archive:

```
mkdir solrini && tar -zxvf solr*.tgz -C solrini --strip-components=1
```

Start Solr:

```
solrini/bin/solr start -c -m 8G
```

Note: Adjust memory usage (i.e., `-m 8G` as appropriate).

Run the Solr bootstrap script to copy the Anserini JAR into Solr's classpath and upload the configsets to Solr's internal ZooKeeper:

```
pushd src/main/resources/solr && ./solr.sh ../../../../solrini localhost:9983 && popd
```
   
Solr should now be available at [http://localhost:8983/](http://localhost:8983/) for browsing.

### Indexing document collections into SolrCloud from Anserini

We'll index [Washington Post collection](https://github.com/castorini/anserini/blob/master/docs/regressions-core18.md) as an example.

First, create the `core18` collection in Solr:

```
solrini/bin/solr create -n anserini -c core18
```

Run the Solr indexing command for `core18`:

```
sh target/appassembler/bin/IndexCollection -collection WashingtonPostCollection -generator WapoGenerator \
   -threads 8 -input /path/to/WashingtonPost \
   -solr -solr.index core18 -solr.zkUrl localhost:9983 \
   -storePositions -storeDocvectors -storeTransformedDocs
```

Note: Make sure `/path/to/WashingtonPost` is updated with the appropriate path.

Once indexing has completed, you should be able to query `core18` from the Solr [query interface](http://localhost:8983/solr/#/core18/query).

## neo4j

Start a neo4j instance via Docker with the command:
```bash
docker run -d --name neo4j --publish=7474:7474 --publish=7687:7687 \
    --volume=`pwd`/neo4j:/data \
    -e NEO4J_dbms_memory_pagecache_size=2G \
    -e NEO4J_dbms_memory_heap_initial__size=4G \
    -e NEO4J_dbms_memory_heap_max__size=16G \
    neo4j
```

Note: You may wish to update the memory settings based on the amount of available memory on your machine.

neo4j should should be available shortly at [http://localhost:7474/](http://localhost:7474/) with the default username/password of `neo4j`/`neo4j`. You will be prompted to change the password, this is the password you will pass to the load script.

In order for efficient inserts and queries, build the following indexes in neo4j:
```
CREATE INDEX ON :Document(id)
CREATE INDEX ON :Entity(id)
CREATE INDEX ON :Fact(relation)
CREATE INDEX ON :Fact(value)
CREATE INDEX ON :Fact(relation, value)
CREATE INDEX ON :Mention(id)
CREATE INDEX ON :Mention(class)
CREATE INDEX ON :Mention(index)
CREATE INDEX ON :Mention(span)
CREATE INDEX ON :Mention(id, class, span)
CREATE INDEX ON :Relation(type)
CREATE INDEX ON :Relation(type, confidence)
```

## Running

### Extraction

For each document in the collection, we extract mentions of named entities, the relations between them, and links to entities in an external knowledge graph.

Run `ExtractTriples`:

```
./bin/extract.sh
```

Note: Modify `extract.sh` based on your environment (e.g., available memory, number of executors, Solr, neo4j password, etc.) - options available [here](src/main/scala/io/dstlr/package.scala).

After the extraction is done, check if an output folder (called `triples/` by default) is created, and several Parquet files are generated inside the output folder.

If you want to inspect the Parquet file:

- Download  and build [parquet-tools](https://github.com/apache/parquet-mr/tree/master/parquet-tools) following instructions.

Note: If you are on Mac, you could also install it with Homebrew `brew install parquet-tools`.

- View the Parquet file in JSON format:

```
parquet-tools cat --json [filename]
```

### Enrichment

We augment the raw knowledge graph with facts from the external knowledge graph (Wikidata in our case).

Run `EnrichTriples`:

```
./bin/enrich.sh
```

Note: Modify `enrich.sh` based on your environment.

After the enrichment is done, check if an output folder (called `triples-enriched/` by default) is created with output Parquet files.

### Load

Load raw knowledge graph and enriched knowledge graph produced from the above commands to neo4j.

Set `--input triples` in `load.sh`, run `LoadTriples`:

```
./bin/load.sh
```

Note: Modify `load.sh` based on your environment.

Set `--input triples-enriched` in `load.sh`, run `LoadTriples` again:

```
./bin/load.sh
```

Open [http://localhost:7474/](http://localhost:7474/) to view the loaded knowledge graph in neo4j.

## Data Cleaning Queries

The following queries can be run against the knowledge graph in neo4j to discover sub-graphs of interest.

### Supporting Information

This query finds sub-graphs where the value extracted from the document matches the ground-truth from Wikidata.

```
MATCH (d:Document)-->(s:Mention)-->(r:Relation {type: "ORG_CITY_OF_HEADQUARTERS"})-->(o:Mention)
MATCH (s)-->(e:Entity)-->(f:Fact {relation: r.type})
WHERE o.span = f.value
RETURN d, s, r, o, e, f
```

### Inconsistent Information

This query finds sub-graphs where the value extracted from the document does not match the ground-truth from Wikidata.

```
MATCH (d:Document)-->(s:Mention)-->(r:Relation {type: "ORG_CITY_OF_HEADQUARTERS"})-->(o:Mention)
MATCH (s)-->(e:Entity)-->(f:Fact {relation: r.type})
WHERE NOT(o.span = f.value)
RETURN d, s, r, o, e, f
```

### Missing Information

This query finds sub-graphs where the value extracted from the document does not have a corresponding ground-truth in Wikidata.

```
MATCH (d:Document)-->(s:Mention)-->(r:Relation {type: "ORG_CITY_OF_HEADQUARTERS"})-->(o:Mention)
MATCH (s)-->(e:Entity)
OPTIONAL MATCH (e)-->(f:Fact {relation: r.type})
WHERE f IS NULL
RETURN d, s, r, o, e, f
```

### Delete Relationships

This query deletes all relationships in the database.

```
MATCH (n) DETACH DELETE n
```
