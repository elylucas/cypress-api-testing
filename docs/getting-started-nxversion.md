---
slug: /
---

# Getting Started

Welcome to the Workshop!

This workshop covers cool stuff.

To get started, clone the repo and install the dependencies:

## Setup

### Create NX workspace:

```bash
npx create-nx-workspace@15.1.1 cypress-heroes --preset=angular-nest --appName=client --style=css --nxCloud false
```

Open project in code

### Setup Cypress

Create api-tests cypress app:

```bash
nx g @nrwl/cypress:cypress-project api-tests --baseUrl=http://localhost:3333/api
```

Add cypress-plugin-api in root angular-todos:

```bash title=/
npm i -D cypress-plugin-api
```

Import plugin in support file:

```ts title=apps/api-tests/src/support/e2e.ts
import 'cypress-plugin-api';
```

Clean up base test and delete app.po.ts file

## Create missions service and controller

Use NX Console to create a nest missions resource. Uncheck crud, select api for
app, put app for directory, and select none for tests

update service:

```ts
import { Injectable } from '@nestjs/common';

const defaultTodo = {
  id: 1,
  description: 'take out the trash',
  complete: false,
};

@Injectable()
export class TodosService {
  todos: Todo[] = [{ ...defaultTodo }];

  getAll() {
    return this.todos;
  }
}
```

update controller:

```ts
import { Controller, Get } from '@nestjs/common';
import { TodosService } from './todos.service';

@Controller('todos')
export class TodosController {
  constructor(private readonly todosService: TodosService) {}

  @Get()
  getTodos() {
    return this.todosService.getAll();
  }
}
```



add client-e2e/src/api/todos.api.cy.ts:

```ts
describe('todos api', () => {
  it('should get todos', () => {
    cy.api('/todos').as('response');
    cy.get('@response').its('status').should('equal', 200);
  });
});
```

run cypress from client-e2e folder

```bash
npx cypress open
```

## add todo

update todoservice:

```ts
import { Injectable } from '@nestjs/common';

export interface Todo {
  id?: number;
  description: string;
  complete: boolean;
}

const defaultTodo: Todo = {
  id: 1,
  description: 'take out the trash',
  complete: false,
};

@Injectable()
export class TodosService {
  todos: Todo[] = [{ ...defaultTodo }];

  getAll() {
    return this.todos;
  }

  add(todo: Todo) {
    const newId = Math.max(...this.todos.map((x) => x.id)) + 1;
    todo.id = newId;
    this.todos.push(todo);
    return todo;
  }
}
```

update controller:

```ts
@Post()
addTodo(@Body() todo: Todo) {
  return this.todosService.add(todo);
}
```

add test:

```ts
it('can add todo', () => {
  cy.api('POST', '/todos', {
    description: 'test todo',
    complete: false,
  }).as('response');
  cy.get('@response').its('status').should('equal', 201);
  cy.get('@response').its('body').should('include', {
    description: 'test todo',
    complete: false,
  });
});
```

### get single api

update service/controller:

```ts
//service
get(id: number) {
  const todo = this.todos.find(x => x.id === id);
  return todo;
}

//controller
get(id: number) {
  const todo = this.todos.find(x => x.id === id);
  return todo;
}
```

add test:

```ts
it('should get single todo', () => {
  cy.api('/todos/1').as('response');
  cy.get('@response').its('status').should('equal', 200);
});
```

show no response is coming back, but the status code is 200. debug to show why.

update the controller get to use parse int pipe:

```ts
@Get(':id')
getTodo(@Param('id', ParseIntPipe) id: number) {
  return this.todosService.get(id);
}
```

Show it now works. However, show it still fails in the same way as before if we
pass an id that doesn't exist, like 100.

```ts
it.only('should  throw 404 if single todo is not found', () => {
  cy.api({
    url: '/todos/100',
    failOnStatusCode: false,
  }).as('response');
  cy.get('@response').its('status').should('equal', 404);
});
```

Update the service to throw a NotFoundException

```ts
get(id: number) {
  const todo = this.todos.find(x => x.id === id);
  if (!todo) {
    throw new NotFoundException();
  }
  return todo;
}
```

## add db reset for testing

service/controller

```ts
@Post('/reset')
reset() {
  this.todosService.reset();
}

reset() {
  this.todos = [{...defaultTodo}];
}
```

explain how you would want to have better security around this

add beforeeach to test:

```ts
beforeEach(() => {
  cy.request({
    method: 'POST',
    url: '/todos/reset',
    headers: {
      Authorization: 'testadmin',
    },
  });
});
```

## delete todo

add controller/service

```ts
@Delete(':id')
deleteTodo(@Param('id', ParseIntPipe) id: number) {
  this.todosService.delete(id);
}

delete(id: number) {
  this.todos = this.todos.filter((x) => x.id !== id);
}
```

add test:

```ts
it('can delete todo', () => {
  cy.api('DELETE', '/todos/1');
  cy.api({
    url: '/todos/1',
    failOnStatusCode: false,
  }).as('response');
  cy.get('@response').its('status').should('equal', 404);
});
```

# App updates

remove all the cruft from the app component, including nx component

## create service

use vscode nx generator @scematics/angular:service
name: todos, project: client, no flat, skipTests true


