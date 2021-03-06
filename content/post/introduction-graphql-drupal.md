---
title: "An Introduction to Graphql in Drupal 8.5.*"
date: 2018-05-25T17:25:44+02:00
update: 2018-07-14T17:25:44+02:00
draft: false
tags: ["front-end", "graphql", "gql", "drupal 8", "d8", "open source"]
categories: ["Technology"]
git: "https://github.com/alejandroq/graphql-drupal.meetup.edu"
---

_This article was originally part of my "edu" series originally publicized via a GitHub repository and live presented to the Drupal NOVA Meetup Group. Therefore much of it is organized in a manner meant to direct said presentation and is not optimized as an article._

<!--more-->

- [Goal](#goal)
- [Ulterior Motives](#ulterior-motives)
- [Requirements](#requirements)
- [Optional Requirements](#optional-requirements)
- [Agenda](#agenda)
- [What is GraphQL?](#what-is-graphql)
- [Why try GraphQL?](#why-try-graphql)
- [Why GraphQL over Drupal's JSON API?](#why-graphql-over-drupals-json-api)
- [GraphQL Gotchas](#graphql-gotchas)
- [Headless Drupal 8 Setup](#headless-drupal-8-setup)
- [gql](#gql)
    - [Inputs](#inputs)
    - [Mutation](#mutation)
    - [Query](#query)
- [React Setup](#react-setup)
- [Resources](#resources)

## Goal
Demo GraphQL in Drupal and point out further resources if interested.

## Ulterior Motives
Loosely demonstrate that 'fuzzy' contracts are a more scalable solution then strignant ones. The latter requiring alignment between back-end and front-end developers and resulting: 1) dependency brittleness, 2) undesirable side-effects of Conway's Law and 3) more conscientious deprecation strategies. TL;DR: Querying an API from the client-side vs implementing a strict contract on said side.

GraphQL enables 'fuzzy' contracts between interdependent entities.

## Requirements
- PHP 7.x
- Drupal 8
- Node
- NPM or Yarn

## Optional Requirements
- Composer
- Docker
- Drush
- Lando

## Agenda
- What is GraphQL?
- Why try GraphQL?
- Why GraphQL over Drupal's JSON API?
- GraphQL Gotchas
- gql
- Headless Drupal 8 Setup
- React Setup

## What is GraphQL?
GraphQL (GQL), open sourced and developed by Facebook, defines itself as a query language for your API. 

## Why try GraphQL?
As a consumer, GQL allows you to compose a query that declares exactly what you need; it returns exactly that. The composition process allows you to concatenate what would be numerous requests in a RESTful service into one request. As a result, a server can transmit both smaller and less frequent payloads, especially benefitting mobile devices with poor connections/compute.

As a provider, GQL has powerful developer tools like GraphiQL and a type system. In our Drupal use case, the [GraphQL module](https://www.drupal.org/project/graphql), maintains both Voyager and Explorer (GraphiQL) routes: `/graphql/voyager` and `/graphql/explorer` appropriately. 

A few use cases: 
- As an obfuscating layer over an ugly and/or legacy API
- Requesting data at a specific component level data
- Transmitting data of loosely connected resources for a dashboard

## Why GraphQL over Drupal's JSON API?
The JSON API is a really great configuration-less module, but its disadvantages are:
- complicated query 
- large output footprint
- web of callbacks as you hop from related resource to related resource
- multiple requests necessary for any UI's of loosely connected resources (likely many routes in a web application)
- strict contract between front-end and back-end; especially complicated if working with large entities (ex. 300+ attribute custom entities - why I first tried drupal/graphql)
- neither can handle Drupal's map field serialization well (as far as I know)

![strict contracts vs fuzzy contracts](../../../images/introduction-graphql-drupal/strict-vs-fuzzy.png)

GraphQL's serving specific queried data especially benefits mobile users. With tools and extensible options, Drupal/graphql extends a great provider, developer and consumer experiences. 

Why try it over the JSON API? Of course the "corner cases" (i.e progressive versioning, speed of development, get it?):

![why are manholes round](../../../images/introduction-graphql-drupal/sewer-edge-case.png)

## GraphQL Gotchas
- GraphQL is protocol agnostic and only depends on the I/O of strings. Whereas an erroneous HTTP response may return a 404, GraphQL always returns a 200. Health checks beware. 
- GraphQL does not rely on HTTP caching and outside of libraries like Apollo or Relay, GraphQL does not handle caching. 
- Not as self-documenting as a well designed RESTful API as the focus is on data and not on affordances. 
- Solves some problems and introduces others
- In Drupal, `filters` require that the field name is the SQL Database's field name: i.e. GraphQL Entity Type may equal fieldIsPrivateGroup, but you would utilize:

```gql
# Note: the field is a boolean in Drupal, but MySQL does not natively support a boolean. The field's* type is `tinyint`, therefore 1 == true and 0 == false.
# *field_is_private_group_value | tinyint(4)
{
  groupQuery(filter: {conditions: [{field: "field_is_private_group", value: ["1"], operator: EQUAL}]}) {
    count
  }
}
```

## Headless Drupal 8 Setup
We are going to setup Drupal as a Todo List CMS. On the client-side we will use React. In-order to fulfill our Headless Drupal we will need a few things:
- Drupal GraphQL setup
- Composer dependency management
- `drupal gm` -> `TodosModule`
- `drupal geco` -> `Todos`
- `drupal geco` -> `TodoList`

Our Custom Entities (TodoList and Todos) will have custom GQL Plugin Input Types and Mutations in-order that drupal/GraphQL module can utilize them. What drupal/GraphQL module CAN utilize can be validated in the `https://headlessdrupal.lndo.site/graphql/explorer` or `https://headlessdrupal.lndo.site/graphql/voyager`.

You can find the results in `02-headless-drupal` directory. 

I utilized Lando (a Docker utility) and Make to simplify environment instantiation to two commands:

```sh
make dependencies && make
```

Outbound will be an entire LAMP Stack with custom entities and proper configuration as stated above. 

*Note: OAuth or other authentication methods are out of the scope of this presentation - but in should absolutely be used in a real application*

## gql
Basics: http://graphql.org/learn/queries/

GQL is protocol agnostic therefore it does not leverage typical HTTP methods of GET, POST, PUT and DELETE. However: 

| **HTTP** | **GQL**  |
| :------- | :------- |
| POST     | Mutation |
| GET      | Query    |
| PUT      | Mutation |
| DELETE   | Mutation |

### Inputs

- We define the TodoInput GraphQL input for Drupal 8 in `02-headless-drupal/web/modules/custom/todo_lists/src/Plugin/GraphQL/InputTypes/TodoInput.php`:

```php
<?php

namespace Drupal\todo_lists\Plugin\GraphQL\InputTypes;

use Drupal\graphql\Plugin\GraphQL\InputTypes\InputTypePluginBase;

/**
 * The input type for article mutations.
 *
 * @GraphQLInputType(
 *   id = "todo_input",
 *   name = "TodoInput",
 *   fields = {
 *     "lid" = "String",
 *     "state" = "String",
 *     "content" = "String",
 *   }
 * )
 */
class TodoInput extends InputTypePluginBase {

}

```

### Mutation

- The below is defined in found in `02-headless-drupal/web/modules/custom/todo_lists/src/Plugin/GraphQL/Mutations/CreateTodo.php`:

```gql
mutation createTodoForTodoList($input: TodoInput) {
  createTodo(input: $input) {
    errors
    entity {
      entityId
    }
  }
}

# query variables:
# {
#  "input": {"lid": "1", "content": "hello world"}
# }
```

- The below is defined in found in `02-headless-drupal/web/modules/custom/todo_lists/src/Plugin/GraphQL/Mutations/CreateTodoList.php`:

```gql
mutation createTodoList($input: TodoListInput) {
  createTodoList(input: $input) {
    errors
    entity {
      entityId
    }
  }
}

# query variables:
# {
#  "input": {"name": "Best List"}
# }

# output (JSON):
# {
#   "data": {
#     "createTodoList": {
#       "errors": [],
#       "entity": {
#         "entityId": "1"
#       }
#     }
#   }
# }
```

The above's PHP code:

```php
<?php

namespace Drupal\todo_lists\Plugin\GraphQL\Mutations;

use Drupal\graphql_core\Plugin\GraphQL\Mutations\Entity\CreateEntityBase;
use Drupal\graphql\GraphQL\Execution\ResolveContext;
use GraphQL\Type\Definition\ResolveInfo;

/**
 * Simple mutation for creating a new article node.
 *
 * @GraphQLMutation(
 *   id = "create_todo_list",
 *   entity_type = "todo_list_entity",
 *   entity_bundle = "todo_list_entity",
 *   secure = true,
 *   name = "createTodoList",
 *   type = "EntityCrudOutput!",
 *   arguments = {
 *      "input" = "TodoListInput"
 *   }
 * )
 */
class CreateTodoList extends CreateEntityBase {

  /**
   * {@inheritdoc}
   */
  protected function extractEntityInput($value, array $args, ResolveContext $context, ResolveInfo $info) {
    return [
      'name' => $args['input']['name'],
    ];
  }

}
```

- The below updates an already created Todo `02-headless-drupal/web/modules/custom/todo_lists/src/Plugin/GraphQL/Mutations/UpdateTodo.php`:

```gql
mutation updateTodo($id: String, $input: TodoInput) {
  updateTodo(id:$id, input:$input) {
    errors
    entity{
      ...on TodoEntity{
        content
        state
      }
    }
  }
}

# query variables:
# {
#   "input": {"lid": 1, "content": "goodbye world", "state": "1"},
#   "id": "1" 
# }

# post query:
query {
  todoEntityQuery(filter: {}) {
    count
    entities{ 
      ...on TodoEntity{
        content
        state
      }
    }
  }
}

# post query result (JSON):
# {
#   "data": {
#     "todoEntityQuery": {
#       "count": 2,
#       "entities": [
#         {
#           "content": "goodbye world",
#           "state": "1"
#         },
#         {
#           "content": "hello world",
#           "state": "0"
#         }
#       ]
#     }
#   }
# }
```

- The below updates the TodoList from `state 0` to `state 1`:

```gql
mutation updateTodoList($id: String, $input: TodoListInput) {
  updateTodoList(id:$id, input:$input) {
    errors
    entity{
      ...on TodoListEntity{
        name
        state
      }
    }
  }
}

# query variables:
# {
#   "input": {"state": "1"},
#   "id": "1" 
# }

# output:
# {
#   "data": {
#     "updateTodoList": {
#       "errors": [],
#       "entity": {
#         "name": "Best List",
#         "state": "1"
#       }
#     }
#   }
# }
```

### Query

- Now that we have created a Todo List lets query it:

```gql
query {
  todoListEntityQuery(filter:{}) {
    count
    entities{
      entityId
      ...on TodoListEntity{
        name
      }
    }
  }
}

# output:
# {
#   "data": {
#     "todoListEntityQuery": {
#       "count": 1,
#       "entities": [
#         {
#           "entityId": "1",
#           "name": "Best List"
#         }
#       ]
#     }
#   }
# }
```

We keep `filter: {}` as we are not explicitly filtering any content. 

The previous queries were tested locally in `https://headlessdrupal.lndo.site/graphql/explorer`.

## React Setup
We will be using `create-react-app` CLI tool to quickly create a React app. 

*Quick note: in-order to avoid Access-Control-Origin errors, update your `web/sites/default/services.yml` appropriately:*

```yml
cors.config:
    enabled: true
    # Specify allowed headers, like 'x-allowed-header'.
    allowedHeaders: ['Content-Type', 'Authorization']
    # Specify allowed request methods, specify ['*'] to allow all possible ones.
    allowedMethods: ['GET', 'POST']
    # Configure requests allowed from specific origins.
    allowedOrigins: ['*']
    # Sets the Access-Control-Expose-Headers header.
    exposedHeaders: false
    # Sets the Access-Control-Max-Age header.
    maxAge: false
    # Sets the Access-Control-Allow-Credentials header.
    supportsCredentials: false

```

*This will avoid request pre-flight errors once your working from a single page application*

We will create a couple components, namely a `TodoList` comprised of `Todos`. We will compose them of behaviors that include GraphQL queries via Higher Order Components. This way we can maintain the components as simple as possible.

Status Update as of 5/22/18 @ 7PM: the Headless React app hasn't been completed. But here are a few takeaways about combining the Apollo GraphQL client, React and Recompose.

- Locally setup a new ApolloClient

```js
import { ApolloClient } from 'apollo-client';
import { HttpLink } from 'apollo-link-http';
import { InMemoryCache } from 'apollo-cache-inmemory';

export const client = new ApolloClient({
  link: new HttpLink({ uri: 'http://headlessdrupal.lndo.site:8000/graphql' }),
  cache: new InMemoryCache(),
  connectToDevTools: true,
});
```

- Enwrap your application into with the ApolloProvider so every component has props to query/mutate data

```js
import React from 'react';
import { ApolloProvider } from 'react-apollo';
import styled from 'styled-components';

import { client } from './client';
import { Lists } from '../Lists/Lists';


const Container = styled.div`
  padding: 1em 3em;
`

export const App = () => (
  <ApolloProvider client={client}>
    <Container>
      <h1>
        Todo List
      </h1>
      <Lists />
    </Container>
  </ApolloProvider>
)
```

- The following is a Higher Order Component (HOC) is an example that queries for a list of TodoLists. *Note how we utilize the Recompose utility library to enable branching edge cases such as Loading and Errors during HTTP transit*

```js
import React from 'react';
import { graphql } from 'react-apollo';
import gql from 'graphql-tag';
import { compose, branch, renderComponent, mapProps } from 'recompose';

const _listsQuery = graphql(gql`
    query {
        todoListEntityQuery(filter:{}) {
            count
            entities{
                entityId
                ...on TodoListEntity{
                    name
                    state
                }
            }
        }
    }
`);

const massageLists = ({ data: { todoListEntityQuery: { entities }}}) => ({
    lists: entities,
});

const isLoading = ({ data: { loading }}) => loading;

const hasError = ({ data: { error }}) => error;

const LoadingComponent = () => <p>Loading...</p>

const ErrorComponent = ({ data: { error: { message }, ...props } }) => (
    <React.Fragment>
        <p style={{color: 'red'}}>{message}</p>
        <code>{JSON.stringify(props)}</code>
    </React.Fragment>
);

export const listsQuery = compose(
    _listsQuery,
    branch(isLoading, renderComponent(LoadingComponent)),
    branch(hasError, renderComponent(ErrorComponent)),
    mapProps(massageLists)
);
```

- We utilize the above HOC as so

```js
import React from 'react';
import { compose } from 'recompose';

import { listsQuery } from '../../hoc/listsQuery';
import { TodoList } from '../TodoList/TodoList';

const enhance = compose(
    listsQuery
)

export const Lists = enhance(({ lists }) => (
   <React.Fragment>
       {
           lists.map(({ entityId, ...props }, i) => <TodoList id={entityId} key={i} {...props} />)
       }
   </React.Fragment>
));
```

- Our TodoList leverages the following HOC so we can toggle its state of "Completion" or "Incompletion"

```js
import gql from 'graphql-tag';
import { graphql } from 'react-apollo';
import { compose, withHandlers, withState, mapProps } from 'recompose';

const mutation = gql`
    mutation updateTodoList($id: String, $input: TodoListInput) {
        updateTodoList(id:$id, input:$input) {
            errors
            entity{
                ...on TodoListEntity{
                    name
                    state
                }
            }
        }
    }
`

export const todoListMutation = compose(
    graphql(mutation),
    withState('originState', 'setOriginState', 0),
    withHandlers({
        completeTodoList: ({ mutate, id, state, originState, setOriginState }) => event => {
            if (originState !== state) {
                setOriginState(state)
                originState = state;
            }
            mutate({
                variables: {
                    id,
                    input: { 
                        state: (originState === "1") ? "-1" : "1"
                    },
                },
            }).then(({ data: { updateTodoList: { entity: { state } }}}) => {
                console.log('[todoListMutation] State', state);
                setOriginState(state);
            }).catch(err => {
                console.error(err);
            });
        }
    }),
    mapProps(({ originState, ...props }) => ({ state: originState, ...props })),
)
```

- Our actual TodoList with branching Components and access to an event handler for the triggering of our mutation function

```js
import React from 'react';
import FlexView from 'react-flexview';
import { branch, compose, renderComponent } from 'recompose';

import { todoListMutation } from '../../hoc/todoListMutation';

const isComplete = ({ state }) => state === "1";

const CompleteList = ({ id, name, completeTodoList }) => (
    <FlexView>
        <FlexView marginRight={'1em'}>
        <h2 style={{color: 'gray', textDecoration: 'line-through'}}>{name}</h2>
        </FlexView>
        <FlexView vAlignContent={'center'}>
            <div>
                <button onClick={() => completeTodoList()}>Mark as Incomplete</button>
            </div>
        </FlexView>
    </FlexView>
);

const IncompleteList = ({ id, name, completeTodoList }) => (
    <FlexView>
        <FlexView marginRight={'1em'}>
            <h2>{name}</h2>
        </FlexView>
        <FlexView vAlignContent={'center'}>
            <div>
                <button onClick={() => completeTodoList()}>Mark as Complete</button>
            </div>
        </FlexView>
    </FlexView>
);

const enhance = compose(
    todoListMutation,
    branch(isComplete, renderComponent(CompleteList), renderComponent(IncompleteList)),
);

export const TodoList = enhance();
```

*NOTE: as for 5/22/18 by 7PM I had only a few hours to put the above React components together and therefore left much to be desired as per the other CRUD components*

## Resources
- This is an expansive topic (and my review thus far very basic), so be sure to checkout proper documentation, etc! 
- https://www.amazeelabs.com/en/blog/graphql-for-drupal-basics

