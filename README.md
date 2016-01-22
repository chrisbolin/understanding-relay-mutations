👷_(work in progress)_

# Understanding Relay Mutations

This is a walkthrough of Relay mutations using the Todo MVC example.

## Prerequisites

To get the most out of this walkthrough, you'll probably want to have a good grasp of a few topics...

* **GraphQL.** [Learn GraphQL](https://learngraphql.com/), the [GraphQL homepage](http://graphql.org/), and [Thinking in GraphQL](https://facebook.github.io/relay/docs/thinking-in-graphql.html) are great resources.
* **Relay.** Read through [Thinking in Relay](https://facebook.github.io/relay/docs/thinking-in-relay.html) and complete the [Relay Tutorial](https://facebook.github.io/relay/docs/tutorial.html).
* **React.** Go build a simple React app. Read [Thinking in React](https://facebook.github.io/react/docs/thinking-in-react.html).
* **NodeJS.** [Install it](https://nodejs.org/en/) and maybe do a [quick tutorial](http://blog.modulus.io/absolute-beginners-guide-to-nodejs).

## Mutations

Mutations are arguably the most difficult feature of feature of Relay to learn. Mutations allow us to use a GraphQL query to change our data (in this case that data is on the server). All mutations consist of three pieces:

1. Mutation schema on the **GraphQL server**
1. Relay Mutation class on the **Relay client**
1. Relay Store update on the **Relay client**

#### 1. Mutation Schema (Server)

The mutation schema on the server is where we define how the mutation should be called (**inputs**), what it does to our data (**mutations**), and what it returns (**outputs**). With just the mutation schema we can successfully call the mutation with a simple HTTP request or use [GraphiQL](https://github.com/graphql/graphiql). This is the best place to start creating a mutation.

Relay mutations are GraphQL mutations with a few extra requirements. We will use `mutationWithClientMutationId()` from the `graphql-relay` package to help create a Relay-compliant GraphQL mutation.

#### 2. Relay.Mutation (Client)

The `Relay.Mutation` class must be extended to call the mutation from the client. This is the most complicated and confusing aspect of Relay mutations. If you are struggling to get a mutation to work, chances are that your problem lies here.

#### 3. Relay Store Update (Client)

The `Relay.Mutation` subclass you create needs to be instantiated and sent to `Relay.Store.commitUpdate()` to run the mutation from the Relay client. The mutation will be sent to the server, carried out, and the results will be updated in the Relay client store.

### Field Change - Renaming a Todo

The simplest type of mutation in Relay is the change to a single field. In this app we can rename a todo.

#### Mutation Schema (Server)

`todo/data/schema.js`

```javascript
var GraphQLRenameTodoMutation = mutationWithClientMutationId({
  name: 'RenameTodo',
  inputFields: {
    id: { type: new GraphQLNonNull(GraphQLID) },
    text: { type: new GraphQLNonNull(GraphQLString) },
  },
  outputFields: {
    todo: {
      type: GraphQLTodo,
      resolve: ({localTodoId}) => getTodo(localTodoId),
    },
  },
  mutateAndGetPayload: ({id, text}) => {
    var localTodoId = fromGlobalId(id).id; // transform the global id to a local id
    renameTodo(localTodoId, text); // database call
    return {localTodoId};
  },
});
```


#### Relay.Mutation (Client)

`todo/js/mutations/RenameTodoMutation.js`

```javascript
export default class RenameTodoMutation extends Relay.Mutation {
  // we only need to request the id to perform this mutation
  static fragments = {
    todo: () => Relay.QL`
      fragment on Todo {
        id,
      }
    `,
  };
  getMutation() {
    return Relay.QL`mutation{renameTodo}`;
  }
  getFatQuery() {
    return Relay.QL`
      fragment on RenameTodoPayload {
        todo {
          text,
        }
      }
    `;
  }
  getConfigs() {
    return [{
      type: 'FIELDS_CHANGE',
      fieldIDs: {
        todo: this.props.todo.id,
      },
    }];
  }
  getVariables() {
    // there are two inputs to the mutation: the todo `id` and the new `text` name.
    return {
      id: this.props.todo.id,
      text: this.props.text,
    };
  }
  getOptimisticResponse() {
    // optionally spoof the server response
    // same output as the above outputFields()
    return {
      todo: {
        id: this.props.todo.id,
        text: this.props.text,
      },
    };
  }
}
```

#### Relay Store Update (Client)

`todo/js/components/Todo.js`

```javascript
_handleTextInputSave = (text) => {
  Relay.Store.commitUpdate(
    new RenameTodoMutation({todo: this.props.todo, text})
  );
}
```
