_incomplete, but you'll forgive me_ 🌈

# Understanding Relay Mutations

Mutations are arguably the most difficult feature of Relay to learn. This is a walkthrough of mutations using the official [Todo MVC example](https://github.com/facebook/relay/tree/master/examples/todo) that's also included with this guide.

### Prerequisites

To get the most out of this walkthrough, you'll probably want to have a good grasp on a few topics...

* **GraphQL.** [Learn GraphQL](https://learngraphql.com/), the [GraphQL homepage](http://graphql.org/), and [Thinking in GraphQL](https://facebook.github.io/relay/docs/thinking-in-graphql.html) are great resources. After using them, you should be familiar with [GraphiQL](https://github.com/graphql/graphiql).
* **Relay.** Read through [Thinking in Relay](https://facebook.github.io/relay/docs/thinking-in-relay.html) and complete the [Relay Tutorial](https://facebook.github.io/relay/docs/tutorial.html). Of course, it's expected that Relay mutations are still a mystery to you, or you probably wouldn't be reading this.
* **React.** Go build a simple React app. Read [Thinking in React](https://facebook.github.io/react/docs/thinking-in-react.html).
* **NodeJS.** [Install it](https://nodejs.org/en/) and maybe do a [quick tutorial](http://blog.modulus.io/absolute-beginners-guide-to-nodejs).

### Running the Example

The Todo example in this repo is a slightly modified version of the [original](https://github.com/facebook/relay/tree/master/examples/todo) (for starters, GraphiQL is enabled and all local Relay dependencies are replaced).

```
git clone https://github.com/chrisbolin/understanding-relay-mutations
cd understanding-relay-mutations/examples/todo
npm install
npm start
```

After the servers start navigate to [localhost:3000](http://localhost:3000) to view the app and [localhost:3000/graphql](http://localhost:3000/graphql) to play with GraphiQL. I'd recommend opening them both in separate tabs.

This example stores its data in the server's memory. Therefore, reloading the app page will not discard any saved changes, but restarting the server will reset the data.

## The Anatomy of Relay Mutations

Mutations allow us to use a GraphQL query to change our data. All mutations consist of three pieces:

1. Mutation schema on the **GraphQL server**
1. Relay Mutation class on the **Relay client**
1. Relay Store update on the **Relay client**

We'll look at each step in the process using the simplest mutation in our example app, _renaming a todo_.

#### 1. Mutation Schema (Server)

This is the best place to start creating a mutation. The mutation schema on the server is where we define how the mutation should be called (**inputs**), what it does to our data (**mutations**), and what it returns (**outputs**). With just this mutation schema we can successfully call the mutation with a simple HTTP request or GraphiQL.

##### *`mutationWithClientMutationId(params)`*

Relay mutations are GraphQL mutations with a few extra requirements. We will use `mutationWithClientMutationId()` from the `graphql-relay` package to help create a Relay-compliant GraphQL mutation. This function accepts an object with four parameters and creates GraphQL mutation.

**The Code**

Below is the full code for creating the mutation with `mutationWithClientMutationId`. We'll cover each piece in detail below.

**File:** `todo/data/schema.js`

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
    var localTodoId = fromGlobalId(id).id;
    renameTodo(localTodoId, text);
    return {localTodoId};
  },
});
```

There are two more small pieces of bookkeeping, which will be familiar to you from queries in GraphQL: creating the parent mutation GraphQL Object and exporting the GraphQL schema.

```javascript
var Mutation = new GraphQLObjectType({
  name: 'Mutation',
  fields: {
    renameTodo: GraphQLRenameTodoMutation,
    ...
  },
});

export var schema = new GraphQLSchema({
  query: Root,
  mutation: Mutation,
});
```

These importantly determine the path we'll call the mutation from:
```python
mutation { # from the GraphQLSchema
	renameTodo # from the Mutation GraphQLObjectType
}
```

**Parameters for `mutationWithClientMutationId`**

`name` (String)

The name of the mutation is important. `mutationWithClientMutationId` will create a payload type using this name suffixed with "Payload"; for example a mutation with the name `RenameTodo` creates an output payload with the name `RenameTodoPayload`. This payload type will be used by the client.

`inputFields` (Object)

The `inputFields` object should feel familiar to other GraphQL fields, for example the `fields` object used when creating a new `GraphQLObjectType` instance. Each field inside of `inputFields` is an object with a `type` and an optional `description`. For example, here's the `RenameTodo` mutation's `inputFields`:

```javascript
inputFields: {
  id: { type: new GraphQLNonNull(GraphQLID) },
  text: { type: new GraphQLNonNull(GraphQLString) },
}
```

It's important to note that one other input will be added to this mutation by `mutationWithClientMutationId`: a required String input, `clientMutationId`. This ID is generated by the Relay client behind the scenes to track the mutation's progress.

All Relay mutations have only one input, conveniently named `input`; this is an object that holds all of the `inputFields` plus `clientMutationId`. (This is not a GraphQL requirement, but rather an extra Relay standard.)

**Calling the Mutation without Relay**

To try out the mutation without Relay, go to GraphiQL (at [localhost:3000/graphql](http://localhost:3000/graphql)). We'll rename the first todo "Taste Javascript" to "Get Cereal". Remember, the clientMutationId input is required by the mutation; don't worry, we can spoof it when we're interacting with the mutation outside of Relay.

```python
mutation {
  renameTodo(input: {id: "VG9kbzow" text: "Get Cereal" clientMutationId: "1"}) {
    todo {
      text
    }
  }
}
```

You should this response back (but if you previously deleted the todo with the ID `VG9kbzow` you'll get an error)...

```javascript
{
  "data": {
    "renameTodo": {
      "todo": {
        "text": "Get Cereal"
      }
    }
  }
}
```

 Now go to the app at [localhost:3000](http://localhost:3000) and refresh it to see the renamed todo. (As an aside, you have to refresh the app because Relay [doesn't yet](https://github.com/facebook/relay/issues/541) support "subscribing" to updates from other clients.)

#### 2. Relay.Mutation (Client)

The `Relay.Mutation` class must be extended to call the mutation from the client. This is the most complicated and confusing aspect of Relay mutations. If you are struggling to get a mutation to work, chances are that your problem lies here.

**File:** `todo/js/mutations/RenameTodoMutation.js`

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

#### 3. Relay Store Update (Client)

The `Relay.Mutation` subclass you create needs to be instantiated and sent to `Relay.Store.commitUpdate()` to run the mutation from the Relay client. The mutation will be sent to the server, carried out, and the results will be updated in the Relay client store.

**File:** `todo/js/components/Todo.js`

```javascript
_handleTextInputSave = (text) => {
  Relay.Store.commitUpdate(
    new RenameTodoMutation({todo: this.props.todo, text})
  );
}
```
