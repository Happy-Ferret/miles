**NOTE:** This is a sketch of an idea. Don't expect things to work properly, or things to be fully thought through. It doesn't actually save data to a database yet.

# 🎺 Miles

**Everything you need to build a modern JavaScript app**

Redux, GraphQL, Babel, Webpack… are you struggling to get your head around the JavaScript ecosystem?

Miles is a different approach. It includes everything you need to build a modern database-backed JavaScript app, from database to UI, so you can actually get on with building something.

You model your data as classes, then Miles does the heavy lifting of storing the data in a PostgreSQL database and getting the data from the server to the client. You write your user interface in React, then hook up the React views to the data model with controllers. (Yes, it's MVC frameworks from 2005 all over again!)

## What does it look like?

First, you define your models:

```javascript
class Todo extends Model {
  static fields = {
    id: new IDField(),
    text: new StringField(),
    completed: new BooleanField(default=false)
  };

  toggle() {
    this.update({ completed: !this.completed });
  }
}
```

This definition is used to create a PostgreSQL table, and a bunch of methods are provided on the model for creating, reading, updating, and deleting data from the client. Miles wires up the client and server automatically, so you don't need to worry about this if you don't want to.

Next, you define your UI with React:

```javascript
const TodoView = ({ onClick, completed, text }) => (
  <li
    onClick={onClick}
    style={{
      textDecoration: completed ? "line-through" : "none"
    }}
  >
    {text}
  </li>
);

const TodoListView = ({ todos, toggleTodo }) => (
  <ul>
    {todos.map(todo => (
      <TodoView key={todo.id} {...todo} onClick={() => toggleTodo(todo)} />
    ))}
  </ul>
);
```

Then, wire the two together with a controller:

```javascript
const TodoController = () => (
  <div>
    <h1>Todos</h1>
    <Todo.Query all>
      {({ loading, error, todos }) => {
        if (loading) return <div>Loading...</div>;
        if (error) return <div>Error: {error.toString()}</div>;

        return (
          <TodoListView todos={todos} toggleTodo={todo => todo.toggle()} />
        );
      }}
    </Todo.Query>
  </div>
);
```

Finally, you hook the controller up to a URL route:

```javascript
const App = () => (
  <Router>
    <Route exact path="/" component={TodoController} />
  </Router>
);
```

That's it! There are a few other bits that need adding, like authentication and authorization, but this hopefully gives you the gist of the architecture.

## Getting started

The easiest way to learn how to use Miles is by example. We're going to create a basic todo app, as outlined above. You'll need to have [Node 10.3 or above installed](https://nodejs.org/).

### Creating an app

First step is to scaffold a basic app:

```
$ npm init miles todoapp
$ cd todoapp/
```

Let's check that everything is working by booting up a development server. Run this command:

```
$ npm start
```

When it finished booting, you can go to http://localhost:3000 and you should see it running.

Open up the `todoapp/` directory in a code editor or file browser to take a look around. You'll see a directory structure like this:

```
todoapp/
  client/
    controllers/
      home.js
    models/
    views/
    App.js
    index.js
  public/
    index.html
  server/
    index.js
  package.json
```

There are three main directories:

- `client/` - The user-facing React app.
- `public/` - Any static files that you want served by your server, including the root `index.html`.
- `server/` - The Node.js server that serves your app and data. This is the entrypoint for your application – it compiles and serves everything in `client/` and `public/`.

Within `client/`, you've got the main building blocks for your app:

- `client/App.js` - Your app's URL routes.
- `client/models/` - Your data models.
- `client/views/` - React components that describe how to present your data.
- `client/controllers/` - The business logic that wires together the models and views.

There are a few other files in here, but don't worry too much about them – we're going to look at all of the files in more detail later.

### Model your data

Models define your database schema. They are classes which define the database fields and the operations that are performed on that data.

The model objects themselves are used from the client. Miles takes care of transmitting the data to and from the server, and ultimately storing that data in a database.

The idea is that models are a single, definitive source of truth about what your data looks like. Database tables, API clients, API servers, etc, can all be derived from it. Modelling your data also helps keep your code organised: instead of operations on the data being scattered all over your codebase, you can keep them all in one place behind a well-organised set of methods.

We're making a todo app, so let's model a todo item. Create `client/models/todo.js` with this content:

```javascript
import models from "miles-prototype/models";
import { createQuery } from "miles-prototype/models/query";

class Todo extends models.Model {
  static fields = {
    id: new models.IDField(),
    text: new models.StringField(),
    completed: new models.BooleanField({ default: false })
  };

  toggle() {
    this.update({ completed: !this.completed });
  }
}
Todo.Query = createQuery(Todo);

export default Todo;
```

As you can see, Miles models subclass `Model`. Models must have a `fields` attribute, which define the database fields.

The field class uses tells Miles what type of database field to create, and what type to use to represent that data in JavaScript. Field classes also have various optional options, such as `default`, which defines a default value to assign to that field if it is not specified.

You can also define your own methods on models to do whatever you need to do with your data. In this example, there is a method to toggle the state of the `completed` field. It calls the `update()` method. When called from within your app, this method makes an API call to the server and runs an `UPDATE` query on the database.

_(Note: Ignore `Todo.Query`. We need to come up with a better syntax for that.)_

You also need to register the model with the server so it knows to create database tables and serve up an API for it.

In `server/index.js`, import the model underneath the other imports at the top:

```javascript
import Todo from "../client/models/todo";
```

Then add this line just before `server.listen(...)`:

```javascript
server.registerModel(Todo);
```

The server is just a plain old Express app. If you need to do any custom server-side stuff, or add any extra API calls, this is where you can do it.

_(Note: This is probably where authorization will happen, perhaps as options passed to `registerModel()`.)_

You'll need to restart your development server now if it's running because I haven't implemented auto-reloading for the server yet. Sorry.

### Writing some views

Next, let's write some views. This section assumes you have some knowledge of React, so if you don't, [give its tutorial a whirl](https://reactjs.org/tutorial/tutorial.html).

Views are just React components. They describe how data should be presented and how the user-interface should behave.

Let's make a simple view that lists todos. Create `client/views/todo-list.js` with this content:

```javascript
import React from "react";

const TodoView = ({ onClick, completed, text }) => (
  <li
    onClick={onClick}
    style={{
      textDecoration: completed ? "line-through" : "none"
    }}
  >
    {text}
  </li>
);

const TodoListView = ({ todos, toggleTodo }) => (
  <ul>
    {todos.map(todo => (
      <TodoView key={todo.id} {...todo} onClick={() => toggleTodo(todo)} />
    ))}
  </ul>
);

export default TodoListView;
```

This is a React component that, when passed a list of todos as a property, outputs a list of todos. The todos are list of instances of the `Todo` model, with `id`, `text`, and `completed` properties.

### Wiring up the views to data

To connect a model to a view, you use a controller. Controllers describe what data you want to fetch, how to handle loading state, how to handle error state, translating user interaction into data mutations, and so on.

Controllers are just React components, but controllers and views are separated out just for the sake of organising our code. In general, business logic lives in controllers, whereas presentational logic lives in views. (If you've used "container" components when building React apps, controllers are pretty much the same thing.)

A controller has already been made in `client/controllers/home.js` to display some content when you run the server. Let's replace that code with some code that actually does something:

```javascript
import React from "react";
import Todo from "../models/todo";
import TodoListView from "../views/todo-list";

const HomeController = () => (
  <div>
    <h1>Todos</h1>
    <Todo.Query all>
      {({ loading, error, todos }) => {
        if (loading) return <div>Loading...</div>;
        if (error) return <div>Error: {error.toString()}</div>;

        return <TodoListView todos={todos} />;
      }}
    </Todo.Query>
  </div>
);

export default HomeController;
```

The `Todo.Query` component is available on all models. It runs a query against the database and provides the data to the function inside it. That function first checks for loading and error states, then if the data has loaded successfully, passes it to the view we created before.

Now, run `yarn start` and open up http://localhost:3000 in your browser. You should see the title, but nothing else. That's because we haven't got any data in the database yet!

### Creating things

Let's add a form for creating todos. Create the view `client/views/todo-create.js` to define how it looks and how the form works:

```javascript
import React from "react";

const TodoCreateView = ({ createTodo }) => {
  let input;

  return (
    <div>
      <form
        onSubmit={e => {
          e.preventDefault();
          if (!input.value.trim()) {
            return;
          }
          createTodo(input.value);
          input.value = "";
        }}
      >
        <input ref={node => (input = node)} />
        <button type="submit">Add Todo</button>
      </form>
    </div>
  );
};

export default TodoCreateView;
```

The view doesn't actually store the data – it just calls the `createTodo` callback when the form is submitted.

We can wire this up to create some data in the controller. First, add this line to the top of `client/controllers/home.js`, underneath the `TodoCreateView` import:

```javascript
import TodoCreateView from "../views/todo-create.js";
```

Then, insert the view between `<h1>` and `<Todo.Query>`:

```javascript
  ...
  <h1>Todos</h1>
  <TodoCreateView createTodo={text => Todo.create({ text: text })} />
  <Todo.Query all>
  ...
```

We pass a function for the `createTodo` callback, which calls the `Todo.create()` method. This sends an API request to the server, which then creates the todo in the database. This method returns a promise to indicate the result of the creation, but we don't need to worry about this because `Todo.Query` will automatically pick up the new todo we have created.

_(Note: Decent error handling is a WIP.)_

Open up the app in your browser and you should see the form to create todos. Give it a try!

### Updating things

Now we've got a bunch of todos, it would be useful if we could mark them as completed.

In `client/views/todo-list.js`, our views already have wired up `onClick` to a `toggleTodo` callback. All we need to do is add a bit of code to the controller to make it actually update the todo.

In `client/controllers/home.js`, change the `<TodoListView ... />` definition to this:

```javascript
return <TodoListView todos={todos} toggleTodo={todo => todo.toggle()} />;
```

The `toggleTodo` callback is passed an instance of `Todo`. On the model, we have already added implemented a `toggle()` method, so we can just call that directly from here.

That's it! Your development server should have automatically reloaded to reflect the change you made, so try clicking on some todos to mark them as done.

### Deleting things

_(Todo. It's just a `todo.delete()` method.)_

### Next steps
