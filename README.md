# Reactive_Knowledge_Management
Github repository for the article 'Reactive Knowledge Management' (Ceri, Bernasconi, Gagliardi)
[![License](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

This GitHub repository contains the instructions and the data needed to deploy a Neo4j demo in which are installed the four rules presented in the related article . The repository wants demonstrates the feasibility of the proposal outlined in the "Partitioned Knoweldge Management" article.

## Table of Contents

- [Introduction](#introduction)
- [Installation](#installation)
- [Usage](#usage)
- [Dataset](#dataset)
- [License](#license)

## Introduction

Neo4j is a powerful graph database that offers a flexible data model and advanced querying capabilities. It is widely used for modeling and analyzing complex relationships in various domains such as social networks, recommendation systems, and disease spreading models.

The APOC plugin (Awesome Procedures on Cypher) extends Neo4j's functionality by providing a collection of useful procedures and functions. It offers additional operations and utilities that are not available out of the box, making it easier to perform complex graph operations and data manipulations.

This repository serves as a proof of concept, showcasing the implementation of the proposed approach in the "Partitioned Knoweldge Management" article using Neo4j with the APOC plugin.

In this tedbed we refer to the Neo4j 5.10 version and the APOC version 5.10.0 .

## Installation

To use the Neo4j Docker image with the APOC plugin, follow the instructions below:

1. Ensure that Docker is installed on your machine. You can download Docker from the official website: [https://www.docker.com/get-started](https://www.docker.com/get-started).
   
3. Create a local folder on your machine to store the files of this repository.

4. Download from the repository the "data" ZIP folder and unzip it inside the local directory previuosly created.

5. Open terminal and navigate to the directory containing the data files.

6. Run the Neo4j container with the APOC plugin using the following command:

   ```bash
   docker run --rm -p 7474:7474 -p 7687:7687 -v $PWD/data:/data --name reactive_knowledge -e NEO4JLABS_PLUGINS='["apoc"]' -e   NEO4J_apoc_trigger_enabled=true -e NEO4J_AUTH=none neo4j
   ```
    
   This command starts the Neo4j container and exposes the default Neo4j ports (7474 for HTTP and 7687 for Bolt). The Neo4j server will already contain the APOC plugin, the dataset and the installed PG-Triggers. 

8. Access the Neo4j browser by opening http://localhost:7474 in your web browser. 

9. You are now ready to start exploring the testbed!


## Dataset

The repository includes a dataset representing the schema of the COVID-19 spreading model presented in the "Partitioned Knoweldge Management"" article as shown in the following image:
<img src="Images/Dataset.png" alt="Dataset" width="90%"/>


## Usage

Once the Neo4j container is up and running, you can interact with the database using the Neo4j browser or programmatically through Neo4j drivers.

To access the Neo4j browser, open [http://localhost:7474](http://localhost:7474) in your web browser. From there, you can execute Cypher queries, create nodes and relationships, and explore the graph database.

To programmatically interact with Neo4j, you can use any Neo4j driver compatible with the Neo4j version used in the Docker image. Configure the driver to connect to `localhost:7687` with the appropriate credentials.

In this Docker container, you will find the four rules presented in the article that acts upon changes in the dataset.

### Rule R1

```
CALL apoc.trigger.install("neo4j", "R1",
"UNWIND $createdNodes AS newNode
CALL apoc.do.when(newNode:Mutation,
'MATCH (r:Region)-[]-(:Patient)-[]-(newNode)
-[:Risk]->(:CriticalEffect)  
 CREATE (:Alert{rule:'R1', hub:'E', datetime:dateTime(),Region:r.name,
  description:'New critical mutation'}', 
'',{newNode:newNode}) YIELD value RETURN count(*)",
{phase: 'afterAsync'});
MATCH (n:Lineage{lineage:'B.1.1.7'}) SET n.whoDesignation = 'Delta' RETURN * 
```

### Rule R2

```
CALL apoc.trigger.install("neo4j", "R2",
"UNWIND $createdNodes AS newNode
MATCH (newNode)-[]-(:Lineage{name:'unassigned'})
CALL apoc.do.when(newNode:Mutation,
'MATCH (r:Region)-[]-(:Lab)-[]-(newNode)
 MATCH (r:Region)-[]-(:Lab)-[]-(s:Sequence)-[]-(:Lineage{name:'unassigned'})
 WITH r, count(s) as n_sequences
 WHERE n_sequences > 50
 CREATE (:Alert{rule:'R2', hub:'A', datetime:dateTime(),Region:r.name,
  description:'Region-based count of unassigned sequences above threshold'}', 
'',{newNode:newNode}) YIELD value RETURN count(*)",
{phase: 'afterAsync'});

```

### Rule R3


```
CALL apoc.trigger.install("neo4j", "R3",
"UNWIND $createdNodes AS newNode
MATCH (newNode)-[]-(:Lineage{name:'unassigned'})
CALL apoc.do.when(newNode:Mutation,
'MATCH (r:Region)-[]-(:Lab)-[]-(newNode)
 MATCH (r)-[]-(:Lab)-[]-(s:Sequence)-[]-(:Lineage{name:'unassigned'})
 MATCH (s)-[]-(:Mutation)-[]-(:CriticalEffect)
 WITH r, count(s) as n_sequences
 WHERE n_sequences > 50
 CREATE (:Alert{rule:'R3', hub:'A', datetime:dateTime(),Region:r.name,
  description:'Location-based count of unassigned sequences with 
  critical-effect-mutation above threshold'}', 
'',{newNode:newNode}) YIELD value RETURN count(*)",
{phase: 'afterAsync'});

```

### Rule R4

```
CALL apoc.trigger.install("neo4j", "R4",
"UNWIND $createdNodes AS newNode
MATCH (p:Patient)-[]-(:Sequence)-[]-(:Mutation)-[]-(:CriticalEffect)
WITH newNode, count(p) as n_patients
CALL apoc.do.when(newNode:Mutation AND n_patients > 50 ,
'MATCH (r:Region)-[]-(:Hospital)-[]-(newNode)
 MATCH (r)-[]-(:Hospital)-[]-(p:Patients)-[]-(:IcuPatient)
 WITH newRel, r, count(p) as tot_icuPat
 MATCH (r)-[]-(:Hospital)-[]-(p:Patients{admissionDate:newRel.admissionDate})-[]-(:IcuPatient)
 WITH r, tot_icuPat, count(p) today_icuPat
 WHERE toFloat(today_icuPat-tot_icuPat)/toFloat(tot_icuPat) > 0.1
 CREATE (:Alert{rule:'R4', hub:'C', datetime:dateTime(),Region:r.name,
  description:'Significant increase of region-based ICU admission'}', 
'',{newNode:newNode}) YIELD value RETURN count(*)",
{phase: 'afterAsync'});
```

## License

This project is licensed under the [MIT License](LICENSE). Feel free to use, modify, and distribute the code as per the terms of this license.
