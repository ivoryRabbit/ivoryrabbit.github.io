---
title:        Text-to-SQL를 알아보자
date:         2024-11-09
categories:   [Data, Science]
comments:     true
---

<style>
H2 { color: #298294 }
H3 { color: #1e7ed2 }
H4 { color: #C7A579 }
</style>

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

많은 회사들이 의사결정에 데이터를 사용하면서 이미 SQL에 익숙한 개발자뿐만 아니라 PO, PM, 마케터 등 비개발자들에게도 데이터를 읽고 분석할 수 있는 능력이 요구되기 시작했다.

기존에는 데이터 추출을 위해서 데이터 분석가들에게 SQL 쿼리 작성을 요청했겠지만, Text-to-SQL를 사용하면 데이터베이스 및 SQL에 대한 전문 지식이 부족하더라도 원하는 데이터를 언제든지 찾아볼 수 있다.

비개발자도 "지난 한 달 동안 GMV는 얼마였지?" 같은 질문을 자연어로 입력해 데이터를 조회할 수 있으므로, SQL 지식의 장벽을 없애고 데이터 접근성을 크게 높인다.

#### 2. Data Discovery

SQL 지식이 있더라도 조직이 갖고 있는 데이터에 익숙해지기 위해서는 많은 시간과 경험이 필요하다.

실제로 데이터 분석가들은 SQL 쿼리를 작성하는 것 외에도 데이터를 찾아다니는데 어느정도 시간을 할애한다.
    - 조직이 어떤 데이터를 수집하고 있는지
    - 데이터는 어디에 보관되어 있는지
    - 데이터가 비지니스적으로 어떤 의미를 갖고 있는지

데이터가 어디에 있는지, 어떤 테이블과 컬럼이 필요한지 모르더라도 Text-to-SQL은 자연어로 질문만 입력하면 알아서 SQL을 작성해준다.

이는 Data Discovery 도구를 보조하는 역할을 할 수 있으며, 데이터의 생김새와 맥락을 모르는 사용자도 빠르게 데이터를 탐색할 수 있게 돕는다.

### Text-to-SQL 구현

그러면 이제 Python에서 가장 간단한 형태의 Text-to-SQL을 구현해보자.

#### 1. LLM

LLM 서버로는 `OpenAI ChatGPT`를 사용하기로 하였다. OpenAI의 API 키를 발급받은 후, Python에서 API를 호출하기 위한 library를 설치한다.

##### [requirements]
```shell
pip3 install openai sqlparse
```

#### 2. Contextual Prompting

이제 LLM 모델이 다음 질문에 대한 SQL 쿼리를 생성할 수 있도록 프롬프트를 작성해보자.

- "2020년부터 2024년까지 각 년도 별 개봉한 영화들의 평균 평점을 계산해줘."

이때 우리가 보유하고 있는 테이블과 스키마 정보를 LLM 모델에게 제공하기 위해서는 가이드라인 뿐만 아니라 테이블 정보를 프롬프트에 포함시켜야 한다.

이와 같이 모델이 참고할 수 있도록 배경 지식을 제공하는 프롬프팅 방식을 **Contextual Prompting**이라고 한다.

```python
import textwrap
from typing import Dict

import sqlparse
from openai import OpenAI


API_KEY = "{ Your API Token }"
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


dialect = "Trino"

initial_prompt = textwrap.dedent(
    """
    당신은 {dialect} SQL 전문가입니다. 유저의 질문에 대해 SQL 쿼리를 생성하여 답변해주세요.
    제공되는 컨텍스트를 참고하여 SQL 쿼리를 생성해야 하며, 반드시 가이드라인을 준수하여 응답해주세요.
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

raw_sql = response.choices[0].message.content
sql = sqlparse.format(raw_sql, indent_tabs=True, keyword_case="upper")

print(sql)
```

실행 결과:

```sql
SELECT m.year, AVG(r.rating) AS avg_rating
FROM movies m
JOIN ratings r ON m.id = r.movie_id
WHERE m.year BETWEEN 2020 AND 2024
GROUP BY m.year;
```

#### 3. RAG

위의 Contextual Prompting 방식에는 한가지 치명적인 단점이 있다. 사용자가 어떤 질문을 할지 아직 모르는 상태이므로 가능한 모든 배경 지식을 프롬프트에 추가해주어야 한다는 것이다.

하지만 LLM 모델에는 토큰 개수에 제한이 있기 때문에, 우리가 가진 모든 테이블 DDL을 프롬프트에 추가하기에는 한계가 있다.

이때 사용가능한 방법이 바로 **RAG(Retrieval-Augmented Generation)**이다. 사용자로부터 질문이 들어오면, 관련 배경 지식을 검색(Retrieval)하여 프롬프트를 통해 제공할 Context를 보강(Augmented)해주는 역할을 한다.

즉, 프롬프트에 모든 지식을 넣는게 아니라 질문이 들어오면 답변에 필요한 지식만 추출하여 프롬프트에 추가해주는 것이다. 이때 지식 데이터베이스로는 Vector Database 혹은 Graph Database들이 사용된다.

이번 프로젝트에서는 Postgres의 extension 중 하나인 pgvector를 사용하여 Vector Database를 구축해보았다.

먼저 Vector Database를 띄우기 위해 docker compose 파일을 작성해보자.

##### [docker-compose.yaml]
```yaml
services:
  vector-store:
    image: ankane/pgvector:latest
    container_name: vector-store
    ports:
      - "5432:5432"
    environment:
      - POSTGRES_DB=postgres
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres
    volumes:
      - ./docker/postgres/init_db.sql:/docker-entrypoint-initdb.d/init_db.sql
      - ./docker/volume/data:/var/lib/postgresql/data
```

pgvector는 HNSW, IVF 등의 VSS 알고리즘을 지원하고 있다. 이번에는 그 중에서 HNSW 알고리즘을 사용하였다.

- [IVF 알고리즘](https://ivoryrabbit.github.io/posts/IVF/){: target="_blank"}
- [HNSW 알고리즘](https://ivoryrabbit.github.io/posts/HNSW/){: target="_blank"}

##### [init_db.sql]

```sql
CREATE SCHEMA vectordb;

SET search_path TO vectordb;

CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE IF NOT EXISTS ddl_collection (
	ID SERIAL PRIMARY KEY,
	TABLE_NAME VARCHAR(128) NULL,
	DDL_CONTENT VARCHAR NOT NULL,
	EMBEDDING VECTOR(768)
);

CREATE INDEX IF NOT EXISTS ddl_collection_index ON ddl_collection USING hnsw (embedding vector_cosine_ops);
```

```shell
docker compose up --build
```

#### 4. ORM

Python의 SQLAlchemy는 Postgres의 pgvector를 위한 ORM을 지원하고 있다. 이를 활용하면 Vector 타입의 파싱 고민 없이 DML를 실행시킬 수 있다.

##### [requirements]
```shell
pip3 install psycopg2-binary pgvector sqlalchemy
```

##### [model]
```python
from pgvector.sqlalchemy import Vector
from sqlalchemy import Integer, String
from sqlalchemy.orm import DeclarativeBase, mapped_column


class Base(DeclarativeBase):
    pass


class DDLCollection(Base):
    __tablename__ = "ddl_collection"

    id = mapped_column(Integer, primary_key=True)
    table_name = mapped_column(String, unique=True)
    ddl_content = mapped_column(String, nullable=False)
    embedding = mapped_column(Vector(768))
```

#### 5. Sentence Transformers

이제 테이블 스키마 정보를 Vector Database에 저장해보자. 자연어 문장을 Embedding Vector로 변환하는데는 주로 Sentence Transformer 계열의 모델들이 사용된다.

이번 글에서는 허깅페이스에서 제공하는 SentenceTransformer 패키지를 통해 경량화 모델 중 하나인 `paraphrase-albert-small-v2`를 사용하였다.

- [https://huggingface.co/sentence-transformers/paraphrase-albert-small-v2](https://huggingface.co/sentence-transformers/paraphrase-albert-small-v2){: target="_blank"}

##### [requirements]
```shell
pip3 install transformers
```

이제 테이블 DDL과 그 Embedding Vector들을 ORM을 이용해 데이터베이스에 삽입한다.

##### [insert]
```python
import textwrap

from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from sentence_transformers import SentenceTransformer

from model import DDLCollection

URL = "postgresql+psycopg2://postgres:postgres@localhost:5432/postgres"
SCHEMA = "vectordb"

session_maker = sessionmaker(expire_on_commit=False)
engine = create_engine(
    url=URL,
    connect_args={"options": f"-csearch_path={SCHEMA}"},
    echo=True,
)

session_maker.configure(bind=engine)

LOCAL_CACHE_PREFIX = "/tmp/text-to-sql/model"
model = SentenceTransformer(
    model_name_or_path="sentence-transformers/paraphrase-albert-small-v2",
    cache_folder=LOCAL_CACHE_PREFIX,
)

movies_ddl_content = textwrap.dedent(
    """
    CREATE TABLE movies (
        id       BIGINT PRIMARY KEY,
        title    VARCHAR,
        genres   VARCHAR,
        year     SMALLINT
    )
    """
)

users_ddl_content = textwrap.dedent(
    """
    CREATE TABLE users (
        id         BIGINT PRIMARY KEY,
        gender     VARCHAR(4),
        age        SMALLINT,
        occupation INTEGER,
        zipcode    VARCHAR(10)
    )
    """
)

ratings_ddl_content = textwrap.dedent(
    """
    CREATE TABLE ratings (
        user_id    BIGINT,
        movie_id   BIGINT,
        rating     FLOAT,
        timestamp  TIMESTAMP(3),
        PRIMARY KEY (user_id, movie_id)
    )
    """
)

movies_ddl = DDLCollection(
    table_name="movies",
    ddl_content=movies_ddl_content,
    embedding=model.encode(movies_ddl_content),
)

users_ddl = DDLCollection(
    table_name="users",
    ddl_content=users_ddl_content,
    embedding=model.encode(users_ddl_content),
)

ratings_ddl = DDLCollection(
    table_name="ratings",
    ddl_content=ratings_ddl_content,
    embedding=model.encode(ratings_ddl_content),
)

with session_maker.begin() as session:
    session.add_all([movies_ddl, users_ddl, ratings_ddl])
    session.commit()
```

#### 6. Prompting with RAG

마지막으로 사용자의 질문을 Embedding Vector로 인코딩하여 Consine Similarity가 작은 순으로 테이블 DDL들을 검색하고, 이를 프롬프트에 주입하는 코드를 작성해보자.

이번 예제에서는 테이블 개수가 3개 밖에 없기 때문에, 최대로 검색할 데이터의 개수를 2개로 제한하였다. 현업에서는 Join이나 CTE를 위해 3~5개 정도의 테이블을 검색하도록 설정하면 좋을 것 같다.

```python
import textwrap
from typing import Dict

import sqlparse
from openai import OpenAI
from sqlalchemy import create_engine, select
from sqlalchemy.orm import sessionmaker
from sentence_transformers import SentenceTransformer

from model import DDLCollection

URL = "postgresql+psycopg2://postgres:postgres@localhost:5432/postgres"
SCHEMA = "vectordb"

session_maker = sessionmaker(expire_on_commit=False)
engine = create_engine(
    url=URL,
    connect_args={"options": f"-csearch_path={SCHEMA}"},
)

session_maker.configure(bind=engine)

LOCAL_CACHE_PREFIX = "/tmp/text-to-sql/model"
model = SentenceTransformer(
    model_name_or_path="sentence-transformers/paraphrase-albert-small-v2",
    cache_folder=LOCAL_CACHE_PREFIX,
)

API_KEY = "{ Your API Token }"
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


dialect = "Trino"

initial_prompt = textwrap.dedent(
    """
    당신은 {dialect} SQL 전문가입니다. 유저의 질문에 대해 SQL 쿼리를 생성하여 답변해주세요.
    제공되는 컨텍스트를 참고하여 SQL 쿼리를 생성해야 하며, 반드시 가이드라인을 준수하여 응답해주세요.
    """
)

question = "2020년부터 2024년까지 각 년도 별 개봉한 영화들의 평균 평점을 계산해줘."
question_vector = model.encode(question)

with session_maker.begin() as session:
    rows = session.scalars(
        select(DDLCollection)
        .order_by(
            DDLCollection.embedding.cosine_distance(question_vector),
        )
        .limit(2)
    ).all()

context_prompt = "\n".join([row.ddl_content for row in rows])

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

raw_sql = response.choices[0].message.content
sql = sqlparse.format(raw_sql, indent_tabs=True, keyword_case="upper")

print(sql)
```

실행 결과:

```sql
SELECT m.year, AVG(r.rating) AS avg_rating
FROM movies m
JOIN ratings r ON m.id = r.movie_id
WHERE m.year BETWEEN 2020 AND 2024
GROUP BY m.year;
```

### 고찰

이번 글에서는 가장 간단한 형태의 Text-to-SQL을 구현하기 위해서 테이블 DDL을 사용하였다. 만약 이보다 더 정확한 수준의 SQL 쿼리 생성이 요구된다면 다음과 같은 방법들을 사용할 수 있다.

#### 1. Data Catalog

DDL 대신 Data Catalog를 주입해준다.
- Table name, description
- Column name, type, description
- Primary key, partition key

#### 2. 비지니스 용어 사전

데이터 분석에서 사용되는 비지니스 용어들의 정의는 회사 by 회사, 도메인 by 도메인으로 다를 수 있다.

- 용어 사전을 만들고 사용자들이 편집할 수 있도록 어드민 개발
- 사용자의 질문과 가장 관련 있는 용어들의 정의들을 프롬프트에 추가

#### 3. Few-shot Prompting

Few-shot prompting은 몇 가지 예시를 프롬프트와 함께 제공하여 LLM 모델이 더 정확하게 SQL을 생성할 수 있도록 돕는 방식이다.

- 사용자의 질문과 이에 대응되는 SQL 쿼리를 하나의 예제로 만들어 RAG에 저장
- 사용자가 새로운 질문을 입력하면 가장 관련 있는 예제들을 검색하여 추가
- 사람이 직접 예제를 만들 수도 있지만 쿼리 로그를 수집하여 LLM 모델에게 질문을 생성하도록 자동화
- 피드백을 수집하여 양질의 예제를 RAG에 저장