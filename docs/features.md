# Features

!!! note

    This section uses the [Movies database](https://github.com/neo4j-graph-examples/recommendations) from the
    Neo4j graph examples repository. This is slightly different than the Movie Graph exanple in the Neo4j
    docker instance. You can browse the database by connecting to https://demo.neo4jlabs.com:7473/ and logging
    in with username `recommendations` and password `recommendations`.

## Data modelling

Cypher graphs contain nodes and relationships. Nodes are the *nouns* and relationships are the *verbs* in a graph.
Cypherantic represents both concepts as pydantic models with customization options represented by class-level
configuration dictionary named `cypherantic_config`. The model instance fields of the model become the *properties*
of the node or relationship. Each field is customized by adding a [cypherantic.Field][] or [cypherantic.Relationship][]
type annotation to the field type.

### Nodes

```python
import datetime
import typing as t

import pydantic

class Person(pydantic.BaseModel):
    name: str
    born: datetime.date
    born_in: str
    died: datetime.date | None = None

class Actor(Person):
    cypherantic_config: t.ClassVar = {'labels': ['Actor', 'Person']}
    imdb_id: str
    url: pydantic.HttpUrl

class Director(Person):
    cypherantic_config: t.ClassVar = {'labels': ['Director', 'Person']}
```

### Relationships

Relationships are modeled using a pair of classes – a named tuple that represents the edge and a model that
represents the relationship properties. The named tuple has two fields. The `node` contains the target node of the
relationship and the `properties` field contains the relationship properties. Relationships can be added as fields
to a node model by adding a sequence of named tuple type and annotating the field with [cypherantic.Relationship][].

The following example adds the `ACTED_IN` relationship to the `Actor` model.

```python
import datetime
import typing as t

import cypherantic
import pydantic

class Movie(pydantic.BaseModel):
    budget: int
    countries: list[str]
    imdb_id: str
    imdb_rating: float
    plot: str

class MovieRole(pydantic.BaseModel):
    role: str

class ActedInEdge(t.NamedTuple):
    node: Movie
    properties: MovieRole

class Actor(pydantic.BaseModel,):
    cypherantic_config: t.ClassVar = {'labels': ['Actor', 'Person']}
    name: str
    born: datetime.date
    born_in: str
    died: datetime.date | None = None
    imdb_id: str
    url: pydantic.HttpUrl
    acted_in: t.Annotated[
        list[ActedInEdge],
        cypherantic.Relationship(rel_type='ACTED_IN', direction='OUTGOING')
    ] = []
```

Since the `acted_in` field is annotated with a [cypherantic.Relationship][] instance it will not be retrieved as
part of the node properties. You are required to call either [cypherantic.refresh_relationship][] or
[cypherantic.retrieve_relationship_edges][] on the actor model instance. The former requires a relationship field
on the model whereas the latter returns a list of edges.

!!! warning

    I am not entirely happy with how relationships/edges are modeled currently. It feels clunky though necessary if
    you want the edges to be present as model fields. I expect that this portion of the API will change during the
    early releases.
