# ververica-langchain4j

## Local Setup

### Running StreamAPI

1. `docker compose run`
2. Run the class `com.evoura.ververica.langchain4j.stream.VervericaLangchain4jApplication` to start Flink App
3. Runt the helper class `com.evoura.ververica.langchain4j.stream.ConsoleVervericaApplication` to start a chat

### Requirements
- Java 11
- Maven 3.8.X
- Docker 27.X

### Deployment
The application can run locally by executing the `main` method from the `VervericaLangchain4jApplication` class.  
Also, a docker-compose file is available, it contains all required services
    - Kafka Cluster
    - Kafka UI
    - Flink Cluster
    - Ollama
    - Open WebUI (to interact with Ollama)

#### Deploy using docker-compose
1. Build Flink Job
   1. Run `mvn clean install` to generate the fat jar file with all required dependencies
   2. Run `mvn compile jib:dockerBuild` to create a docker image with the job
2. Run `docker-compose up --profiles flink -d` to start the services

## Data Models

### Chat Message
This is a model used by Kafka to represent a chat message between the user and the AI model.

It is used in the following Kafka topics:
- `user-message` - the user messages
- `ai-response` - the AI model response for the user message
- `chat-memory` - the chat memory between the user and the AI model

#### Fields
- `userId`
    - type: _long_
    - description: _the id of the user_
- `chatId`
    - type: _long_
    - description: _the id of the chat_
- `messageId`
    - type: _long_
    - description: _the id of the message_
- `message`
    - type: string
    - description: _the actual message from the user_
- `response`
    - type: string
    - description: _the response from the AI model_
- `timestamp`
    - type: timestamp
    - description: _the timestamp of when the message was created_

#### User Message Example

```json
{
  "userId": 1,
  "chatId": 1,
  "messageId": 1,
  "message": "Hello!",
  "response": null,
  "timestamp": 1694465095000
}
```

#### AI Response Message Example

```json
{
  "userId": 1,
  "chatId": 1,
  "messageId": 1,
  "message": "Hello!",
  "response": "Hello to you too!",
  "timestamp": 1694465095000
}
```


### User LLM Configuration Message
This is a Kafka record that represents the LLM configuration for a specific user. 

**TOPIC: `llm-config`**

#### Fields
- `userId`
    - type: _long_
    - description: _the id of the user_
- `aiModel`
    - type: _string_
    - description: _the name of the AI model_
    - possible values: _"OLLAMA", "OPENAI"_
- `properties`
    - type: _map<string,string>_
    - description: _a map with all properties specific to an AI model_
- `systemMessage`
    - type: _string_
    - description: _a message that provides context, rules, or guidelines for the conversation_

#### Example
```json
{
  "userId": 1,
  "aiModel": "OLLAMA",
  "properties": {
    "baseUrl": "http://localhost:11434",
    "modelName": "llama3.1:latest"
  },
  "systemMessage": "You are an experienced coding mentor, help users understand programming concepts."
}
```

## Flink State

### Chat Memory

A `MapState` is used as a Chat Memory storage in the `ChatMemoryProcessingFunction`.  
It stores both the user message and the response at a key that represents the message id. 

### State Eviction Policies
_NOT IMPLEMENTED_

- The state shall only retain a configurable maximum amount of messages in a sliding window type cleanup.
- The state shall be fully cleaned up if:
  - The chat was closed (`isClosed=true`)
  - A configurable amount of time has passed since the last interaction 

    
# Real-Time RAG

Leveraging Apache Flink in a Retrieval-Augmented Generation (RAG) system offers several key benefits:
- Granular performance optimization: By decoupling the retrieval and generation processes, Flink allows for more precise control over each stage.
- Real-time data querying: Flink can manage continuous data ingestion with a dedicated job, ensuring up-to-date data availability.
- Data transformation support: Flink's built-in functions enable efficient data transformation within the pipeline.

Here's a simplified diagram of the system:  

![flink-rag.svg](flink-rag.svg)

Here, we see the system split into 3 main parts:
### 1. Ingestion  
This stage processes data from multiple services, documents, or other sources via Kafka topics.  
It then converts the data into embeddings using the embedding model, and stores the embeddings in the embedding store.  

### 2. Retrieval  
This component is responsible for taking the user's prompt and finding relevant embeddings.   
Using these embeddings, it builds the context and publishes both the context and the prompt to a Kafka topic.  

### 3. Generation  
This stage consumes the user's prompt along with the context and sends it to the LLM (Large Language Model) to generate a response.   
The generated answer is then sent to a Kafka topic.

# LLM Flink SQL UDF

Interacting with an LLM in Flink SQL can be done by implementing a UDF (User-Defined Function) 
that is based on the [AsyncTableFunction](https://nightlies.apache.org/flink/flink-docs-release-1.20/api/java/org/apache/flink/table/functions/AsyncTableFunction.html).  
This is similar to a [Table Function](https://nightlies.apache.org/flink/flink-docs-release-1.20/docs/dev/table/functions/udfs/#table-functions) that is executed asynchronously.  
The implementation should be similar to what we have done in the Flink DataStream API, check [LangChainAsyncFunction.java](src%2Fmain%2Fjava%2Fcom%2Fevoura%2Fververica%2Flangchain4j%2Fstream%2Ffunction%2FLangChainAsyncFunction.java) for details.  

**Note:** The [AsyncTableFunction](https://nightlies.apache.org/flink/flink-docs-release-1.20/api/java/org/apache/flink/table/functions/AsyncTableFunction.html) can only be used in a **Lookup Connector**.  
Thus, a new connector that extends the [LookupTableSource](https://github.com/apache/flink/blob/e5ce2c34d81ff9c8b20770983838cfaa1e0d5cdf/flink-table/flink-table-common/src/main/java/org/apache/flink/table/connector/source/LookupTableSource.java#L46) class is required.

## Data Embedding

To embed data and save it in an embedding store, we first need to define a table using the langchain-embedding connector, as shown below:
```sql
CREATE TABLE langchain_embedding (
    input_data STRING,
    response STRING
) WITH (
    'connector' = 'langchain-embedding',
    'langchain-embedding.model' = 'OLLAMA',
    'langchain-embedding.model.base.url' = 'http://localhost:11434',
    'langchain-embedding.model.name' = 'nomic-embed-text:latest',
    
    'langchain-embedding.store' = 'QDRANT',
    'langchain-embedding.store.host' = '<qdrant_host',
    'langchain-embedding.store.port' = '<qdrant_port>',
    'langchain-embedding.store.api.key' = '<qdrant_api_key',
    'langchain-embedding.store.collection.name' = 'ververica'
);
```

Then, we need to define a source for the input data:
```sql
CREATE TEMPORARY VIEW input_table AS
SELECT * FROM (
    VALUES 
        ('Ewok language is called Ewokese.', PROCTIME()),
        ('Ewokese was created by Ben Burtt.', PROCTIME()),
        ('Ewok language is Tibetan mixed with Kalmyk languages.', PROCTIME())
) AS input_table(input_data, `timestamp`);
```

Finally, we to process the input data, we perform a lookup on the `langchain_embedding` table using the input data records:
```sql
SELECT e.* 
FROM 
    input_table AS i 
JOIN 
    langchain_embedding FOR SYSTEM_TIME AS OF i.`timestamp` AS e 
ON 
    i.input_data = e.input_data;
```

## Interacting with the Language Model

To interact with the LLM, we need to define a table using the `langchain` connector as shown below:
```sql
CREATE TABLE langchain_llm (
    prompt STRING,
    response STRING
) WITH (
    'connector' = 'langchain',
    'langchain.model' = 'OLLAMA',
    'langchain.model.base.url' = 'http://localhost:11434',
    'langchain.model.name' = 'llama3.1:latest',

    'langchain.embedding.model' = 'OLLAMA',
    'langchain.embedding.model.base.url' = 'http://localhost:11434',
    'langchain.embedding.model.name' = 'nomic-embed-text:latest',

    'langchain.embedding.store' = 'QDRANT',
    'langchain.embedding.store.host' = '<qdrant_host>'',
    'langchain.embedding.store.port' = '<qdrant_port>'',
    'langchain.embedding.store.api.key' = '<qdrant_api_key>'',
    'langchain.embedding.store.collection.name' = 'ververica'
);
```

Then, we need a source for the user messages:

```sql
CREATE TEMPORARY VIEW user_messages AS
SELECT * FROM (
    VALUES 
        ('How is the Ewok language called?', PROCTIME()),
        ('Who created the Ewok language?', PROCTIME()),
        ('What are the languages behind Ewokese?', PROCTIME())
) AS predefined_messages(user_message, `timestamp`);
```

Finally, to query the LLM, we perform a lookup on the `langchain_llm` table using the user's messages:

```sql
SELECT llm.* 
FROM 
    user_messages AS chat 
JOIN 
    langchain_llm FOR SYSTEM_TIME AS OF chat.`timestamp` AS llm 
ON 
    chat.user_message = llm.prompt;
```

## Configuration

### Langchain Table Configuration

| **Config entry**                          | **Explanation**                                              |  **Default value**  | **Required** |
|-------------------------------------------|--------------------------------------------------------------|:-------------------:|:------------:|
| langchain.model                           | AI model, possible values: `DEFAULT`, `OPENAI`, `OLLAMA`     |        None         | **Required** |
| langchain.model.base.url                  | Base URL of the AI model                                     |        None         |   Optional   |
| langchain.model.api.key                   | API key to access the model                                  |        None         |   Optional   |
| langchain.model.name                      | Name of the AI model                                         |        None         |   Optional   |
| langchain.system.message                  | Sets the context or guidelines for the model’s behavior      |        None         |   Optional   |
| langchain.prompt.template                 | Template for structuring prompts                             | See Prompt Template |   Optional   |
| langchain.embedding.model                 | Embedding model, possible values: `DEFAULT`, `OLLAMA`        |       DEFAULT       |   Optional   |
| langchain.embedding.model.base.url        | Base URL of the embedding model                              |        None         |   Optional   |
| langchain.embedding.model.name            | Name of the embedding model                                  |        None         |   Optional   |
| langchain.embedding.store                 | Embedding store, possible values: `DEFAULT`, `QDRANT`        |       DEFAULT       |   Optional   |
| langchain.embedding.store.host            | Host address of the embedding store                          |        None         |   Optional   |
| langchain.embedding.store.port            | Port for the embedding store connection                      |        None         |   Optional   |
| langchain.embedding.store.api.key         | API key for authenticating with the embedding store          |        None         |   Optional   |
| langchain.embedding.store.collection.name | Collection name in the embedding store                       |        None         |   Optional   |
| langchain.embedding.store.min.score       | Minimum score threshold for results in the embedding store   |         0.7         |   Optional   |
| langchain.embedding.store.max.results     | Maximum number of results to return from the embedding store |        1000         |   Optional   |

#### Prompt Template

The prompt template is a structured guide for formatting input queries, helping the model retrieve and generate responses based on relevant external information.
The following variables can be used to insert data into the template:
- `message`, representing the user message
- `information`, representing the relevant information from the embedded store

By default, it has the following value:

```
Answer the following message:

Message:
{{message}}

Base your answer only on the following information:
{{information}}
Do not include any extra information.
```

### Langchain Embedding Table Configuration

| **Config entry**                          | **Explanation**                                          | **Default value** | **Required** |
|:------------------------------------------|----------------------------------------------------------|:-----------------:|:------------:|
| langchain-embedding.model                 | Embedding AI model, possible values: `DEFAULT`, `OLLAMA` |       None        | **Required** |
| langchain-embedding.model.base.url        | Base URL for the embedding model                         |       None        |   Optional   |
| langchain-embedding.model.name            | Name of the embedding model                              |       None        |   Optional   |
| langchain-embedding.store                 | Embedding store, possible values: `DEFAULT`, `QDRANT`    |       None        | **Required** |
| langchain-embedding.store.host            | Host address for the embedding store                     |       None        |   Optional   |
| langchain-embedding.store.port            | Port for the embedding store                             |       None        |   Optional   |
| langchain-embedding.store.api.key         | API key for the embedding store                          |       None        |   Optional   |
| langchain-embedding.store.collection.name | Collection name in the embedding store                   |       None        |   Optional   |

## Notes

Maintaining the lookup syntax is required for Flink to properly interrogate using the lookup connector.

A thing to keep in mind is that built-in support for ML models will be available in Flink soon.  
More details about this change can be found in [FLIP-437](https://cwiki.apache.org/confluence/display/FLINK/FLIP-437%3A+Support+ML+Models+in+Flink+SQL).

Also, there is [FLIP-440](https://cwiki.apache.org/confluence/pages/viewpage.action?pageId=298781093) which will add support for user-defined SQL operators that are stateful and might be a better option.  
But, this FLIP is in a very early stage and there was no official discussion around it.


## Running The Flink SQL POC

The Flink SQL POC with the Langchain Lookup connector can be executed by running the main method from the _**VervericaLangchain4jSqlApplication**_ class.     

It initializes a Flink Table Execution environment, executes the Flink SQL statements, and prints the chat to the console.  

The Langchain connector is configured to connect to OLLAMA, which can be deployed using the docker-compose file.  
