GIthub Repository to reproduce experiments of the paper 
## **GraphRAG: Leveraging Graph-Based Efficiency to Minimize Hallucinations in LLM-Driven RAG for Finance Data**


# Retrieval augmented generation examples

## Ask Dora, get contextualized answer (Retrieval Augmented Generation)

[![Ask DORA chat bot with context](https://img.youtube.com/vi/utAtu9tT5bE/default.jpg)](https://youtu.be/utAtu9tT5bE)

## Check discrepancies between Dora and FFIEC regulations (Retrieval Augmented Graph Relationship Generation)

[![Ask DORA chat bot with context](https://img.youtube.com/vi/3fqo5DeZXXg/default.jpg)](https://youtu.be/3fqo5DeZXXg)

## Find inner references in GDPR regulation (Retrieval Augmented Graph Relationship Generation)

[![Ask DORA chat bot with context](https://img.youtube.com/vi/6q7a502ANjI/default.jpg)](https://youtu.be/6q7a502ANjI)

# Ingestion of DORA and FFIEC database was made using this *cypher* script

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

## GDPR Retrieval augmented graph generation queries

A dump of the GDPR knowledge graph can be downlowded [here](https://drive.google.com/file/d/10OIyNstnkkpJbvUxQ5kH-bbe0fL9jnXW/view?usp=sharing).
It can be rebuilt using [this material](./gdpr_KG).

### compute references

```cypher
"MATCH (n)
WHERE n.references IS NULL
CALL{
WITH n
WITH n, n.text_path + "\n----------\n" + n.text AS txt
CALL apoc.ml.openai.chat([
{role:"system", content:"GDPR regulation is parsed as a tree"},
{role:"system", content:"I will provide you some chunks of text prefixed by their path from the root of the document"},
{role:"system", content:"You mission is to produce a JSON array of references"},
{role:"system", content:"A reference is another part of the GPDR regulation a given chunk mentions"},
{role:"system", content:'Here is an example:

Regulation gdpr
Chapter II
Article 9
Paragraph 2
Paragraph 1
Point a
----------
the data subject has given explicit consent to the processing of those personal data for one or more specified purposes, except where Union or Member State law provide that the prohibition referred to in paragraph 1 may not be lifted by the data subject;

should produce

[{
  "label": "Paragraph",
  "regulation": "gdpr",
  "chapter": "II",
  "article": "9",
  "paragraph": "1"
}]

Here another example:

Regulation gdpr
Chapter III
Article 12
Paragraph 1
----------
The controller shall take appropriate measures to provide any information referred to in Articles 13 and 14 and any communication under Articles 15 to 22 and 34 relating to processing to the data subject in a concise, transparent, intelligible and easily accessible form, using clear and plain language, in particular for any information addressed specifically to a child. The information shall be provided in writing, or by other means, including, where appropriate, by electronic means. When requested by the data subject, the information may be provided orally, provided that the identity of the data subject is proven by other means.

should produce:

[
  {
    "label": "Article",
    "regulation": "gdpr",
    "chapter": "III",
    "article": "13"
  },
  {
    "label": "Article",
    "regulation": "gdpr",
    "chapter": "III",
    "article": "14"
  },
  {
    "label": "Article",
    "regulation": "gdpr",
    "chapter": "III",
    "article": "15"
  },
  {
    "label": "Article",
    "regulation": "gdpr",
    "chapter": "III",
    "article": "16"
  },
  {
    "label": "Article",
    "regulation": "gdpr",
    "chapter": "III",
    "article": "17"
  },
  {
    "label": "Article",
    "regulation": "gdpr",
    "chapter": "III",
    "article": "18"
  },
  {
    "label": "Article",
    "regulation": "gdpr",
    "chapter": "III",
    "article": "19"
  },
  {
    "label": "Article",
    "regulation": "gdpr",
    "chapter": "III",
    "article": "20"
  },
  {
    "label": "Article",
    "regulation": "gdpr",
    "chapter": "III",
    "article": "21"
  },
  {
    "label": "Article",
    "regulation": "gdpr",
    "chapter": "III",
    "article": "22"
  },
  {
    "label": "Article",
    "regulation": "gdpr",
    "chapter": "III",
    "article": "34"
  }
]

other example:

Regulation gdpr
Chapter I
Article 3
Paragraph 2
----------
This Regulation applies to the processing of personal data of data subjects who are in the Union by a controller or processor not established in the Union, where the processing activities are related to

should produce no reference :

[]

'},
{role:"system", content:"Note that everything in the json is lower case except the label"},
{role:"system", content:"No reference in the chunk should produce []"},
{role:"user", content:txt}
],  $apiKey, {model: 'gpt-4'}) yield value
SET n.references = value.choices[0].message.content
} IN TRANSACTIONS OF 1 ROWS
ON ERROR CONTINUE"
```

### Persist relationship

```cypher
"MATCH (n WHERE NOT n.references IS null)
WHERE n.references <> "[]"
CALL {
WITH n
WITH n, coalesce(apoc.convert.fromJsonList(n.references), []) AS refs
UNWIND refs AS ref
WITH n, apoc.text.replace(ref.label, " ", "") AS label, apoc.map.removeKeys(ref, ["label", "chapter"]) AS ref
WITH n, label, ref, apoc.text.join([k IN KEYS(ref) | k+':"'+ref[k]+'"'], ',') AS map_text
WHERE ref.regulation = "gdpr"
CALL apoc.cypher.run("MATCH (cited:" + label + " {"+map_text+"}) RETURN cited AS cited", {ref:ref})
YIELD value
WITH n, ref, label, value.cited AS cited
MERGE (n)-[:REFERS_TO]->(cited)
} IN TRANSACTIONS OF 1 ROW
ON ERROR CONTINUE"
```
