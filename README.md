GIthub Repository to reproduce experiments of the paper 
## **GraphRAG: Leveraging Graph-Based Efficiency to Minimize Hallucinations in LLM-Driven RAG for Finance Data**



# Ingestion was made using this *cypher* script

A dump of the resulting database can be downlowded [here](https://drive.google.com/file/d/1zqc1LeKH__AngZQKYKZ7MuDgvSBspK1F/view?usp=sharing).

## Cleaning database

```cypher
MATCH (n) DETACH DELETE n
```

## Ingest FFIEC

### Create constraints

```cypher
CREATE CONSTRAINT paragraph_url_ord IF NOT EXISTS FOR (n:Paragraph) REQUIRE (n.url,n.
paragraph_ordering) IS NODE KEY
```

```cypher
CREATE CONSTRAINT booklet_url IF NOT EXISTS FOR (n:Booklet) REQUIRE n.url IS UNIQUE
```
### Ingest FFIEC

```cypher
LOAD CSV WITH HEADERS FROM "file:///ffiec_paragraphs_with_embedding.csv" AS line
MERGE (p:Paragraph {
    url:line.url,
    paragraph_ordering:toInteger(line.paragraph_ordering)
    })
    ON CREATE SET p.paragraph_text = line.paragraph_text,
        p.paragraph_embedding = apoc.convert.fromJsonList(line.paragraph_embedding),
        p.section = line.section,
        p.title = line.title,
        p.depth = toInteger(line.depth),
        p.ordering_ix = toInteger(line.ordering_ix),
        p:Ffiec
```

### Link paragraphs

```cypher
MATCH (n:Paragraph)
WITH n.url AS url, n.paragraph_ordering AS ord, n
ORDER BY ord ASC
WITH url, collect(n) AS ns
CALL apoc.nodes.link(ns, "NEXT_PARAGRAPH")
```

### Connect subsection to parent

```cypher
MATCH (child:Paragraph&Ffiec {paragraph_ordering:0})
MATCH (parent:Paragraph&Ffiec {paragraph_ordering:0})
WHERE child.url STARTS WITH parent.url
AND parent.depth + 1= child.depth
MERGE (child)-[:SUBSECTION_OF]->(parent)
```

### Create booklet

```cypher
MATCH (p:Paragraph {depth:1, paragraph_ordering:0})
MERGE (b:Booklet {url: p.section})
    ON CREATE SET b.depth = 0, b:Ffiec, b.paragraph_ordering=0
MERGE (p)-[:SUBSECTION_OF]->(b);
```

### Create FFIEC handbook

```cypher
MERGE (hb:Handbook:Ffiec {paragraph_ordering:0})
WITH hb
MATCH (b:Booklet:Ffiec)
MERGE (b)-[:SUBSECTION_OF]->(hb)
```

### Add DocumentBackbone label

```cypher
MATCH (n:Ffiec {paragraph_ordering:0})
SET n:DocumentTreeBackbone
```

### Set paragraph embedding as Neo4j vector property

```cypher
MATCH (f:Ffiec)
WHERE NOT f.paragraph_embedding IS NULL
CALL db.create.setNodeVectorProperty(f, 'text_v_embedding', f.paragraph_embedding)
```

```cypher
CALL db.index.vector.createNodeIndex(
    'ffiec_text',
    'Ffiec',
    'text_v_embedding',
    1536,
    'cosine'
)
```

### Set booklet and handbook title

```cypher
MATCH (n:Booklet)
SET n.title = apoc.text.replace(split(n.url,'/')[-2],'-',' ')
```

```cypher
MATCH (n:Ffiec&Handbook)
SET n.title = "FFIEC Handbook"
```

## Ingest Dora

### Create constraints

```cypher
CREATE CONSTRAINT sub_article_article_ix IF NOT EXISTS FOR (n:SubArticle) REQUIRE (n.from_article,n.sub_article_ix) IS NODE KEY
```

```cypher
CREATE CONSTRAINT article_id IF NOT EXISTS FOR (n:Article) REQUIRE n.article_id IS UNIQUE
```

### ingest subarticles and chain

```cypher
LOAD CSV WITH HEADERS FROM "file:///dora_sub_articles_with_embedding.csv" AS line
MERGE (sa:SubArticle {
    from_article:line.from_article,
    sub_article_ix:toInteger(line.sub_article_ix)}
    )
    ON CREATE SET
        sa.text = line.sub_article_text,
        sa.article_title = line.title_article,
        sa.text_embedding = line.sub_article_text_embedding,
        sa:Dora,
        sa.depth = 2
```

```cypher
MATCH (n:SubArticle)
WITH n.from_article AS art, n.sub_article_ix AS ord, n
ORDER BY ord ASC
WITH art, collect(n) AS ns
CALL apoc.nodes.link(ns, "NEXT_SUBARTICLE")
```

### Ingest and chain Articles

```cypher
MATCH (sa:SubArticle&Dora WHERE NOT EXISTS {()-[:NEXT_SUBARTICLE]->(sa)} )
MERGE (a:Article {article_id: sa.from_article})
    ON CREATE SET a.depth = 1, a.title = sa.article_title, a:Dora,
    a.article_ordering = toInteger(split(sa.from_article, " ")[-1])
MERGE (sa)-[:SUBSECTION_OF]->(a);
```

```cypher
MATCH (n:Article)
WITH n.article_ordering AS ord, n
ORDER BY ord ASC
WITH collect(n) AS ns
CALL apoc.nodes.link(ns, "NEXT_ARTICLE")
```

### Create HandBook

```cypher
MERGE (hb:Handbook:Dora {title:"DORA Handbook"})
WITH hb
MATCH (b:Article:Dora)
MERGE (b)-[:SUBSECTION_OF]->(hb)
```

### Add DocumentBackbone label

```cypher
MATCH (n:Dora)
WHERE EXISTS {(n)-[:SUBSECTION_OF]-()}
SET n:DocumentTreeBackbone
```

### Set paragraph embedding as Neo4j vector property

```cypher
MATCH (f:Dora)
WHERE NOT f.text_embedding IS NULL
CALL db.create.setNodeVectorProperty(f, 'text_v_embedding', apoc.convert.fromJsonList(f.text_embedding))
```

```cypher
CALL db.index.vector.createNodeIndex(
    'dora_text',
    'Dora',
    'text_v_embedding',
    1536,
    'cosine'
)
```

## Spotting contradictions

### No GDS screening

```cypher
MATCH (sa:Dora:SubArticle)
CALL {
    WITH sa
    CALL db.index.vector.queryNodes('ffiec_text', 10, sa.text_v_embedding)
    YIELD node AS similar, score
    MERGE (sa)-[r:SIMILAR]->(similar)
    SET r.score = score
} IN TRANSACTIONS OF 100 ROWS
```

This could be done in a more efficient way using GDS library KNN algorithm.

### Set openAI API key

```cypher
:param openai_api_key=>"sk-..."
```

### Spot actual contradictions with relatively cheap LLM

```cypher
MATCH (sa:SubArticle)-[r:SIMILAR]->(similar)
WHERE r.contradiction_with_ffiec_checked IS null
OR NOT r.contradiction_with_ffiec_checked
CALL {
WITH sa, r, similar
WITH sa, r, sa.text AS dora_rule, apoc.convert.toJson(collect(similar.paragraph_text)) AS ffiec_eq_rule
WITH sa, r, dora_rule, ffiec_eq_rule, "DORA extract:\n"+dora_rule+"\nFFIEC similar rules:\n"+ffiec_eq_rule AS context
CALL apoc.ml.openai.chat([
{role:"system", content:"Your a legal expert in banking. You're known for being brief and synthetic.
Any answer to a close question will begin with Yes or No. Give specific elements when you spot something.
I will show you extracts of regulations, the first one is from the on Digital Operational Resilience for
the financial sector and Amending regulations (DORA - UE) and the other ones are from Federal Financial
Institutions Examination Council (FFIEC - US). You will answer a question about this extracts:\n"+context},
{role:"user", content:"Is there any actual contradiction between DORA and FFIEC ?"}
],  $openai_api_key, {model:"gpt-3.5-turbo"}) yield value
WITH sa, r, coalesce(value.choices[0].message.content, "No.") AS answer
SET r.contradiction_with_ffiec_checked = true
WITH sa, r, answer
WHERE toLower(answer) STARTS WITH "yes"
SET r.contradiction_with_ffiec_spotted = answer
} IN TRANSACTIONS OF 10 ROWS
```

### Sum up contradictions

```cypher
MATCH (sa:SubArticle)-[r:SIMILAR]->(similar)
WHERE r.contradiction_with_ffiec_checked
AND NOT r.contradiction_with_ffiec_spotted IS NULL
WITH apoc.convert.toJson(collect("- "+substring(r.contradiction_with_ffiec_spotted, 5)+"\n")) AS contradictions
CALL apoc.ml.openai.chat([
{role:"system", content:"Your a legal expert in banking. You're known for being brief and synthetic. Any answer to a close question will begin with Yes or No. Give specific elements when you spot something. I will show you extracts of regulations, the first one is from the on Digital Operational Resilience for the financial sector and Amending regulations (DORA - UE) and the other ones are from Federal Financial Institutions Examination Council (FFIEC - US)."},
{role:"user", content:"Can you spot contradictions between DORA and FFIEC:\n"},
{role:"assistant", content:"Here are the contradictions I found between DORA and FFIEC:\n"+contradictions},
{role:"user", content:"Can you synthesize ?"}
],  $openai_api_key, {model:"gpt-3.5-turbo"}) yield value
WITH value.choices[0].message.content AS contradictions
RETURN contradictions
```

### Sum up contradictions (markdown formatting)

```cypher
MATCH (sa:SubArticle)-[r:SIMILAR]->(similar)
WHERE r.contradiction_with_ffiec_checked
AND NOT r.contradiction_with_ffiec_spotted IS NULL
WITH apoc.convert.toJson(collect("- "+substring(r.contradiction_with_ffiec_spotted, 5)+"\n")) AS contradictions
CALL apoc.ml.openai.chat([
{role:"system", content:"Your a legal expert in banking. You're known for being brief and synthetic. You've studied two regulations: Digital Operational Resilience for the financial sector and Amending regulations (DORA - UE) and the other ones are from Federal Financial Institutions Examination Council (FFIEC - US)."},
{role:"user", content:"Can you spot contradictions between DORA and FFIEC:\n"},
{role:"assistant", content:"Here are the contradictions I found between DORA and FFIEC:\n"+contradictions},
{role:"user", content:"Can you sum up as a markdown table ?"}
],  $openai_api_key, {model:"gpt-3.5-turbo"}) yield value
WITH value.choices[0].message.content AS contradictions
RETURN contradictions
```
