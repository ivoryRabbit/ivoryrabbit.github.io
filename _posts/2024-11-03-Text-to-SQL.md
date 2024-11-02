---
title:        Text-to-SQL 애플리케이션 개발해보기
date:         2024-11-03
categories:   [Data, Science]
math:         true
comments:     true
---

<style>
H2 { color: #298294 }
H3 { color: #1e7ed2 }
H4 { color: #C7A579 }
</style>

(WIP)

## Text-to-SQL이란?

자연어를 SQL 쿼리로 변환하는 기술

### 왜 필요할까?

SQL에 대한 지식 없이도 빠르게 데이터를 조회할 수 있도록 도움

### 언제 유용할까?

1. 빠르고 간단한 데이터 조회 업무
2. SQL 교육
3. Data discovery 대체


## Text-to-SQL 만들어보기

### LLM

- OpenAI ChatGPT

### Agent

#### Zero-shot prompt

```python
import textwrap
from typing import Dict

from openai import OpenAI


API_KEY = "{{ Your API Token }}"
MODEL = "gpt-3.5-turbo"

temperature = 0.2
max_tokens = 4096

client = OpenAI(api_key=API_KEY)


def generate_system_message(message: str) -> Dict[str, str]:
    return {"role": "system", "content": message}


def generate_user_message(message: str) -> Dict[str, str]:
    return {"role": "user", "content": message}


def generate_assistant_message(message: str) -> Dict[str, str]:
    return {"role": "assistant", "content": message}


dialect = "Presto"

initial_prompt = textwrap.dedent(
    """
    당신은 {dialect} 전문가입니다. 유저의 질문에 대해 SQL 쿼리를 생성하여 답변해주세요.
    제공되는 컨텍스트를 참고로 SQL 쿼리를 생성해야 하며, 반드시 가이드라인을 준수하여 응답해주세요.
    """
)

context_prompt = textwrap.dedent(
    """
    ==Given Context
    CREATE TABLE movies (
        id       BIGINT PRIMARY KEY,
        title    VARCHAR,
        genres   VARCHAR,
        year     SMALLINT
    )
    
    CREATE TABLE users (
        id         BIGINT PRIMARY KEY,
        gender     VARCHAR(4),
        age        SMALLINT,
        occupation INTEGER,
        zipcode    VARCHAR(10)
    )
    
    CREATE TABLE ratings (
        user_id    BIGINT,
        movie_id   BIGINT,
        rating     FLOAT,
        timestamp  TIMESTAMP(3),
        PRIMARY KEY (user_id, movie_id)
    )
    """
)

guideline_prompt = textwrap.dedent(
    """
    ==Response Guidelines
    1. 제공된 컨텍스트가 충분하다면 질문에 대한 설명 없이 정확한 SQL 쿼리를 생성해주세요.
    2. 제공된 컨텍스트가 충분하지 않을 때에는 쿼리를 생성할 수 없는 이유를 설명해주세요.
    3. 컨텍스트에서 테이블 리스트가 주어지면 그 중에서 가장 관련성 높은 테이블들을 사용해주세요.
    4. JOIN 구문을 사용할 경우, 반드시 테이블 별칭(alias)을 명시해 주세요. 
    5. Subquery의 depth가 2보다 큰 경우, CTE를 사용해주세요.
    6. 응답 전에 쿼리의 문법이 맞는지, 사용한 컬럼이 테이블에 존재하는지 한번 더 확인해주세요.
    7. SQL 쿼리를 응답해야 하는 경우에는 SQL 포멧으로 정리해서 응답해주세요.
    """
)

question = "2020년부터 2024년까지 각 년도 별 개봉한 영화들의 평균 평점을 계산해줘."

prompts = [
    generate_system_message(
        initial_prompt.format(dialect=dialect)
        + context_prompt
        + guideline_prompt
    ),
    generate_user_message(question)
]

response = client.chat.completions.create(
    model=MODEL,
    messages=prompts,
    max_tokens=max_tokens,
    temperature=temperature,
)

sql = response.choices[0].message.content

print(sql)
```

result

```sql
SELECT m.year, AVG(r.rating) AS avg_rating
FROM movies m
JOIN ratings r ON m.id = r.movie_id
WHERE m.year BETWEEN 2020 AND 2024
GROUP BY m.year;
```

#### Few-shot prompt

### RAG

- postgres pgvector extension
- sqlalchemy

#### Table Design

- Collection of table descriptions
- Collection of business documentation
- Pairs of question-sql for few-shopt prompting

```sql
CREATE SCHEMA vector_store;

SET search_path TO vector_store;

CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE IF NOT EXISTS ddl_collection (
	ID SERIAL PRIMARY KEY,
	DOCUMENT VARCHAR NOT NULL,
	QUESTION VARCHAR(256) NULL,
	EMBEDDING VECTOR(768)
);

CREATE INDEX IF NOT EXISTS ddl_collection_index ON ddl_collection USING hnsw (embedding vector_cosine_ops);

CREATE TABLE IF NOT EXISTS doc_collection (
	ID SERIAL PRIMARY KEY,
	DOCUMENT VARCHAR NOT NULL,
	QUESTION VARCHAR(256) NULL,
	EMBEDDING VECTOR(768)
);

CREATE INDEX IF NOT EXISTS doc_collection_index ON doc_collection USING hnsw (embedding vector_cosine_ops);

CREATE TABLE IF NOT EXISTS sql_collection (
	ID SERIAL PRIMARY KEY,
	DOCUMENT VARCHAR NOT NULL,
	QUESTION VARCHAR(256) NULL,
	EMBEDDING VECTOR(768)
);

CREATE INDEX IF NOT EXISTS sql_collection_index ON sql_collection USING hnsw (embedding vector_cosine_ops);
```

#### Docker Compose
```yaml
services:
  vector-store:
    image: ankane/pgvector:latest
    container_name: vector-store
    ports:
      - "5432:5432"
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres
    volumes:
      - ./docker/postgres/init_db.sql:/docker-entrypoint-initdb.d/create_tables.sql
      - ./docker/volume/data:/var/lib/postgresql/data
```


#### 책임 분산 시키기

- Vector store (RAG)
- AI assistant
- SQL Agent

