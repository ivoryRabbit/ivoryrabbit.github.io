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

## Text-to-SQL

Text-to-SQL이란 사용자가 자연어를 입력하면 이를 SQL 쿼리로 변환하는 기술을 의미한다.

예를 들어 어떤 사용자가 "2024년 한 해 동안 개봉한 영화들을 알려줘."라고 입력하면 Text-to-SQL은 그 응답으로 다음과 같은 SQL 쿼리를 생성해준다.

```sql
SELECT title
FROM movies
WHERE YEAR(release_dttm) = 2024
```

그런데 이 Text-to-SQL은 왜 필요한걸까?

#### 1. 데이터 접근성 향상

많은 회사들이 의사결정에 데이터를 사용하면서, 이미 SQL에 익숙한 개발자뿐만 아니라 PO, PM, 마케터 등 비개발자들에게도 데이터를 읽고 분석할 수 있는 능력이 요구되기 시작했다.

기존에는 데이터 추출을 위해서 데이터 분석가들에게 SQL 쿼리 작성을 요청했지만, Text-to-SQL를 사용하면 데이터베이스와 SQL에 대한 지식이 부족하더라도 원하는 데이터를 언제든지 찾아볼 수 있다.

비개발자도 "지난 한 달 동안 GMV는 얼마였지?" 같은 질문을 자연어로 입력해 데이터를 조회할 수 있으므로, SQL 지식의 장벽을 없애고 데이터 접근성을 크게 향상시킨다.

#### 2. Data Discovery

SQL 지식이 있더라도 조직이 갖고 있는 데이터에 익숙해지기 위해서는 많은 시간과 경험이 필요하다.

실제로 데이터 분석가들은 SQL 쿼리를 작성하는 것 외에도 데이터를 찾아다니는데 어느정도 시간을 할애한다.
    - 조직이 어떤 데이터를 수집하고 있는지
    - 데이터는 어디에 보관되어 있는지
    - 데이터가 비지니스적으로 어떤 의미를 갖고 있는지

데이터가 어디에 있는지, 어떤 테이블과 컬럼이 필요한지 모를 때에도 Text-to-SQL은 자연어로 질문만 입력하면 알아서 SQL을 작성해준다.

이는 Data Discovery 도구를 대체하거나 보완하는 역할을 하며, 데이터 구조와 배경에 대한 지식이 부족한 사용자도 효율적으로 데이터를 탐색할 수 있게 돕는다.

### Text-to-SQL 구현

그러면 이제 파이썬을 이용해 Text-to-SQL 애플리케이션을 구핸해보자.

#### 1. LLM

LLM 서버로는 `OpenAI ChatGPT`를 사용하기로 하였다.

#### 2. Agent

#### 3. Zero-shot prompting

Zero-shot 프롬프팅이란 사전에 학습된 예시나 정보 없이도 모델에게 질문을 던져 바로 답을 생성하는 방식을 의미한다.

```python
import textwrap
from typing import Dict

from openai import OpenAI


API_KEY = "{{ Your API Token }}"
MODEL = "gpt-3.5-turbo"

temperature = 0.2
max_tokens = 1000

client = OpenAI(api_key=API_KEY)


def generate_system_message(message: str) -> Dict[str, str]:
    return {"role": "system", "content": message}


def generate_user_message(message: str) -> Dict[str, str]:
    return {"role": "user", "content": message}

dialect = "Presto"

initial_prompt = textwrap.dedent(
    """
    당신은 {dialect} 전문가입니다. 유저의 질문에 대해 SQL 쿼리를 생성하여 답변해주세요.
    제공되는 컨텍스트를 참고로 SQL 쿼리를 생성해야 하며, 반드시 가이드라인을 준수하여 응답해주세요.
    """
)

guideline_prompt = textwrap.dedent(
    """
    ==Response Guidelines
    1. JOIN 구문을 사용할 경우, 반드시 테이블 별칭(alias)을 명시해 주세요.
    2. Subquery의 depth가 2보다 큰 경우, CTE를 사용해주세요.
    3. SQL 쿼리를 응답해야 하는 경우에는 SQL 포멧으로 정리해서 응답해주세요.
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
WITH yearly_movies AS (
    SELECT 
        EXTRACT(YEAR FROM release_date) AS release_year,
        movie_id
    FROM movies
    WHERE EXTRACT(YEAR FROM release_date) BETWEEN 2020 AND 2024
)

SELECT 
    release_year,
    AVG(rating) AS avg_rating
FROM yearly_movies ym
JOIN ratings r ON ym.movie_id = r.movie_id
GROUP BY release_year
ORDER BY release_year
```

#### 4. Contextual prompting

이제 우리가 보유하고 있는 테이블 및 컬럼 정보를 사용해 SQL 쿼리를 생성하도록 해보자.

```python
import textwrap
from typing import Dict

from openai import OpenAI


API_KEY = "{{ Your API Token }}"
MODEL = "gpt-3.5-turbo"

temperature = 0.2
max_tokens = 1000

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

#### 5. Few-shot prompting

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

