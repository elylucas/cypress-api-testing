---
slug: /
---

## Getting Started

Greetings, friend! In this tutorial, we are going to cover creating an API using the [NestJS](https://nestjs.com) framework. Along the way, we will build out the application using tests with [Cypress](https://cypress.io) and the [Cypress API Plugin](https://github.com/filiphric/cypress-plugin-api).

If you are not familiar with NestJS, it is a Node based framework heavily influenced by Angular and is great for building any type of server side application with. What I like about Nest is that it provides building blocks (similar to those found in Angular) for creating applications. This helps those who are already familiar with Nest and how it works to jump from project to project and know roughly how things already work.

Cypress is typically known for end-to-end testing web applications, as well as component testing. With the Cypress API Plugin, we can use Cypress to test our backend applications as well. The plugin provides a nice interface that shows the results from each API call. Typically I would reach for something like Postman to use when I develop APIs, but with Cypress, I can nearly replace Postman and have a nice suite of automated tests to go along with it when I'm done.

For this tutorial, we are creating an mission board for a fictional agency known as Cypress Heroes. This board will feature a list of missions that our heroes can partake in. We'll be able to get, create, update, and delete missions in the board. Yes, its pretty much a todo list, but its much cooler because, you know, super heroes.

To get started, you will need to have a relatively newer of Node installed and a code editor (like VS Code).

## Create Nest Application

The first thing we will do is install the Nest CLI tool:

```bash
npm i -g @nestjs/cli
```

Next, use the CLI to create a new application:

```bash
nest new cypress-heroes-api
```

When prompted, select NPM as your package manager.

Go into the newly created **cypress-heroes-api** directory and install Cypress and the plugin:

```bash
npm i -D cypress cypress-plugin-api
```

Next, open the project in your code editor (we'll be using VS Code in the tutorial):

```bash
code .
```
### Fetching Heroes



```bash
nest g controller missions
```

Go over new controller code and how it was added to module.

Delete spec

## Add Get Method

Add a get method to return a list of missions:

```ts
@Get()
getMissions() {
  return [
    {
      description: 'save the world',
      complete: false,
    },
  ];
}
```

### Configure Cypress

Launch Cypress and configure e2e

update config with baseUrl:

```ts
e2e: {
  baseUrl: 'http://localhost:3000',
  setupNodeEvents(on, config) {
    // implement node event listeners here
  },
},
```

Import plugin in support file:

```ts title=cypress/support/e2e.ts
import 'cypress-plugin-api';
```

Start nest server:

```bash
npm run start:dev
```

Add first test:

```ts title=cypress/e2e/mission.api.cy.ts
describe('missions api', () => {
  it('should get missions', () => {
    cy.api('/missions').as('response');
    cy.get('@response').its('status').should('equal', 200);
  });
});
```

## Add service

Refactor code to pull from service

create service

```bash
nest g service missions
```

Show how service was added to module

Replace the service:

```ts title=src/missions/missions.service.ts
import { Injectable } from '@nestjs/common';

export interface Mission {
  id?: number;
  description: string;
  complete: boolean;
}

const defaultMission: Mission = {
  id: 1,
  description: 'save the galaxy',
  complete: false,
};

@Injectable()
export class MissionsService {
  missions: Mission[] = [{ ...defaultMission }];

  getAll() {
    return this.missions;
  }
}
```

update controller:

```ts
import { Controller, Get } from '@nestjs/common';
import { MissionsService } from './missions.service';

@Controller('missions')
export class MissionsController {
  constructor(private missionsService: MissionsService) {}

  @Get()
  getMissions() {
    return this.missionsService.getAll();
  }
}
```

## Get single mission

add test:

```ts
it('should get single mission', () => {
  cy.api('/missions/1').as('response');
  cy.get('@response').its('status').should('equal', 200);
  cy.get('@response').its('body').should('include', {
    id: 1,
    description: 'take out the trash',
    complete: false,
  });
});
```

```ts
//service
get(id: number) {
  const mission = this.missions.find(x => x.id === id);
  return mission;
}

//controller
@Get(':id')
get(@Param('id') id: number) {
  return this.missionsService.get(id);
}
```

show no response is coming back, but the status code is 200. debug to show why.

update the controller get to use parse int pipe:

```ts
@Get(':id')
getMission(@Param('id', ParseIntPipe) id: number) {
  return this.missionsService.get(id);
}
```

Show it now works. However, show it still fails in the same way as before if we
pass an id that doesn't exist, like 100.

```ts
it('should throw 404 if single mission is not found', () => {
  cy.api({
    url: '/missions/100',
    failOnStatusCode: false,
  }).as('response');
  cy.get('@response').its('status').should('equal', 404);
});
```

Update the service to throw a NotFoundException

```ts
get(id: number) {
  const mission = this.missions.find((x) => x.id === id);
  if (!mission) {
    throw new NotFoundException();
  }
  return mission;
}
```

### add delete method

add test:

```ts
it('can delete mission', () => {
  cy.api('DELETE', '/missions/1').as('response');
  cy.get('@response').its('status').should('equal', 204);
});
```

add controller/service

```ts
@Delete(':id')
@HttpCode(204)
deleteMission(@Param('id', ParseIntPipe) id: number) {
  this.missionsService.delete(id);
}

delete(id: number) {
  this.missions = this.missions.filter((x) => x.id !== id);
}
```

## add db reset for testing

service/controller

```ts
@Post('/reset')
reset() {
  if (process.env.NODE_ENV === 'test' || process.env.NODE_ENV === 'dev') {
    this.missionsService.reset();
  }
}

reset() {
  this.missions = [{ ...defaultMission }];
}
```

add dev env to start script:

```json
"start:dev": "NODE_ENV=dev nest start --watch",
"start:test": "NODE_ENV=test nest start",
```

add beforeEach call in e2e.ts

```ts title=cypress/support/e2e.ts
beforeEach(() => {
  cy.request({
    log: false,
    method: 'POST',
    url: '/missions/reset',
    headers: {
      Authorization: 'resetcreds',
    },
  });
  cy.log('seeding db');
});
```

## add mission method

test:

```ts
it('can add mission', () => {
  const mission = {
    description: 'test mission',
    complete: false,
  };
  cy.api('POST', '/missions', mission).as('response');
  cy.get('@response').its('status').should('equal', 201);
  cy.get('@response').its('body').should('include', mission);
});
```

```ts
//controller
@Post()
addMission(@Body() mission: Mission) {
  return this.missionsService.add(mission);
}

//service
add(mission: Mission) {
  const newId = Math.max(...this.missions.map((x) => x.id)) + 1;
  mission.id = newId;
  this.missions.push(mission);
  return mission;
}
```

## update mission

test:

```ts
it('should update mission', () => {
  const mission = {
    description: 'get cat out of tree',
    complete: true,
  };
  cy.api({
    url: '/missions/1',
    method: 'PUT',
    body: mission,
  }).as('response');
  cy.get('@response').its('status').should('equal', 200);
  cy.get('@response').its('body').should('include', mission);
});
```

code:

```ts
//service
update(id: number, mission: Mission) {
  mission.id = id;
  this.missions = this.missions.map((x) => {
    return x.id === id ? mission : x;
  });
  return this.get(id);
}

//controller
@Put(':id')
updateMission(
  @Param('id', ParseIntPipe) id: number,
  @Body() mission,
) {
  return this.missionsService.update(id, mission);
}
```

## Validation

add class transformer and validator:

```bash
npm i class-transformer class-validator
```

refactor Mission into class and move to new file:

```ts title=Mission.ts
export class Mission {
  id?: number;
  description: string;
  complete: boolean;
  created: string;

  constructor(partial: Partial<Mission>) {
    Object.assign(this, partial);
  }
}
```

update missions and reset to use class in service:

```ts
//missions
missions: Mission[] = [new Mission(defaultMission)];

//reset
reset() {
  this.missions = [new Mission(defaultMission)];
}
```

update Mission class to use decorators

```ts
import { IsString, IsNotEmpty, IsBoolean } from 'class-validator';

export class Mission {
  id?: number;

  @IsString({ message: 'description must be a string' })
  @IsNotEmpty({ message: 'description must not be an empty string' })
  description: string;

  @IsBoolean({ message: 'complete must be a boolean' })
  complete: boolean;
  created: string;
}
```

use validationpipe in app.module:

```ts
{
  provide: APP_PIPE,
  useValue: new ValidationPipe({ transform: true }),
},
```

delete parseint pipe from controller because the validator will now convert
types automatically

## Exclude properties

add check to get single mission test to check for created:

```ts
cy.get('@response').its('body.created').should('be.undefined');
```

Add exclude decorator to created prop on class

```ts
import { Exclude } from 'class-transformer';

//prop
@Exclude()
created: string;
```

show fail, then add ClassSerializerInterceptor to module providers:

```ts
{
  provide: APP_INTERCEPTOR,
  useClass: ClassSerializerInterceptor,
},
```

## Protect reset db route with guard

create guard:

```bash
nest g guard TestEnvOnly
```

update guard:

```ts title=test-env-only.guard.ts
import { CanActivate, ExecutionContext, Injectable } from '@nestjs/common';
import { Observable } from 'rxjs';

@Injectable()
export class TestEnvOnlyGuard implements CanActivate {
  canActivate(
    context: ExecutionContext
  ): boolean | Promise<boolean> | Observable<boolean> {
    const request = context.switchToHttp().getRequest();

    if (!(process.env.NODE_ENV === 'test' || process.env.NODE_ENV === 'dev')) {
      return false;
    }

    if (request.headers.authorization !== 'resetcreds') {
      return false;
    }

    return true;
  }
}
```

add guard to just reset method:

```ts
@UseGuards(TestEnvOnlyGuard)
@Post('/reset')
reset() {
  this.missionsService.reset();
}
```
