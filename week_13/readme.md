Hmmm,
we have so far covered typescript as backend,react as frontend, zod as validator,
and much more

Basically want to build a website clone known as medium,

## Step 1 - The stack

We'll be building medium in the following stack

1. React in the frontend
2. Cloudflare workers in the backend
3. zod as the validation library, type inference for the frontend types
4. Typescript as the language
5. Prisma as the ORM, with connection pooling
6. Postgres as the database
7. jwt for authentication (Cookies approach explained in the end as well)

## Step 2 - Initialize the backend

Whenever you're building a project, usually the first thing you should do is initialise the project's backend.
Create a new folder called medium

```bash
mkdir medium
cd medium
```

Initialize a hono based cloudflare worker app

```bash
npm create hono@latest
```

⠙(node:3455) MaxListenersExceededWarning: Possible EventEmitter memory leak detected. 11 close listeners added to [TLSSocket]. Use emitter.setMaxListeners() to increase limit
(Use `node --trace-warnings ...` to show where the warning was created)
Need to install the following packages:
create-hono@0.7.2
Ok to proceed? (y) y

> npx
> create-hono

create-hono version 0.7.2
? Target directory backend
? Which template do you want to use? cloudflare-workers
✔ Cloning the template
? Do you want to install project dependencies? yes
? Which package manager do you want to use? npm
✔ Installing project dependencies
🎉 Copied project files
Get started with: cd backend

to start the backend run:

```bash
npm run dev
```

## Step 3 - Initialize handlers

To begin with, our backend will have 4 routes- index.ts

1. POST /api/v1/signup
2. POST /api/v1/signin
3. POST /api/v1/blog
4. PUT /api/v1/blog
5. GET /api/v1/blog

## Step 4 - Initialise Prisma

go to aiven , sign up, select postgres as service.
create a free service

now about the connection pool for prisma
maintain a pool for connection so that people can't connect to db directly

create a project called medium-backend,select accelerate for scalable connection pooling
paste the db url here in the db url field

click on enable accelerate button, then on api key generate button, we get a api key, which i guess will be main key needed, store it somewhere

create a '.env' file , now keep the url in it, the one we created originally
Make sure you are in the backend folder

Add DATABASE_URL as the connection pool url in wrangler.toml

```bash
npm i prisma
npx prisma init
```

### Initialise Schema

we initialise the schema in backend>prisma>schema.prisma

```sql
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id       String   @id @default(uuid())
  email    String   @unique
  name     String?
  password String
  posts    Post[]
}

model Post {
  id        String   @id @default(uuid())
  title     String
  content   String
  published Boolean  @default(false)
  author    User     @relation(fields: [authorId], references: [id])
  authorId  String
}
```

migrate the database from backend folder

```bash
npx prisma migrate dev --name init_schema
```

migrated the database into prisma/migration folder

generate the client

```bash
npx prisma generate --no-engine
npm install @prisma/extension-accelerate
```

## Initialize the prisma client

index.ts->

```ts
import { PrismaClient } from "@prisma/client/edge";
import { withAccelerate } from "@prisma/extension-accelerate";
```

```ts
//this one in one of routes->
const prisma = new PrismaClient({
  datasourceUrl: env.DATABASE_URL,
}).$extends(withAccelerate());
```

## Step 5 - Create non auth routes

1. Simple Signup route
   Add the logic to insert data to the DB, and if an error is thrown, tell the user about it-> app.post("/api/v1/signup")

2. Add JWT to signup route
   Also add the logic to return the user a jwt when their user id encoded.
   This would also involve adding a new env variable JWT_SECRET to wrangler.toml

3. Add a signin route
   app.post('/api/v1/signin')

## Step 6 - Create auth routes/ Middlewares

1. Limit middleware

```ts
app.use("/api/v1/blog/*", async (c, next) => {
  await next();
});
```

('/api/v1/blog/:id'),('/api/v1/blog'),('/api/v1/blog') - routes to be protected
so we added a limit middleware in blog/\* itself

2. Write the Middlewares

3. Write the routes for access routes

## Step 7 - Create a blog route, better routes

update the files so by default index.ts looks like this:
but the new file would look like:

```ts
app.use("/api/v1/blog/*", async (c, next) => {
  const jwt = c.req.header("Authorization");
  if (!jwt) {
    c.status(401);
    return c.json({ error: "unauthorized" });
  }
  const token = jwt.split(" ")[1];
  const payload = await verify(token, c.env.JWT_SECRET);
  if (!payload) {
    c.status(401);
    return c.json({ error: "unauthorized" });
  }
  c.set("userId", payload.id);
  await next();
});

//POST /api/v1/signup
app.post("/api/v1/signup", async (c) => {
  const prisma = new PrismaClient({
    datasourceUrl: c.env.DATABASE_URL,
  }).$extends(withAccelerate());

  const body = await c.req.json();

  const user = await prisma.user.create({
    data: {
      email: body.email,
      password: body.password,
    },
  });

  if (!user) {
    c.status(403);
    return c.json({ error: "user not found" });
  }

  //@ts-ignore- to ignore the checks
  const token = await sign({ id: user.id }, c.env.JWT_SECRET);
  return c.json({
    jwt: token,
  });
});

//POST /api/v1/signin
app.post("/api/v1/signin", async (c) => {
  const prisma = new PrismaClient({
    datasourceUrl: c.env?.DATABASE_URL,
  }).$extends(withAccelerate());

  const body = await c.req.json();
  const user = await prisma.user.findUnique({
    where: {
      email: body.email,
    },
  });

  if (!user) {
    c.status(403);
    return c.json({ error: "user not found" });
  }

  const jwt = await sign({ id: user.id }, c.env.JWT_SECRET);
  return c.json({ jwt });
});

//POST /api/v1/blog
//confirm if user is able to access authorised routes
app.post("/api/v1/blog", (c) => {
  console.log(c.get("userId"));
  return c.text("signin route");
});

//PUT /api/v1/blog
app.put("/api/v1/blog", (c) => {
  return c.text("put Page!");
});

//GET /api/v1/blog/:id
app.get("/api/v1/blog/:id", (c) => {
  return c.text("blog page!");
});

//limit middlewares
app.use("/api/v1/blog/*", async (c, next) => {
  const jwt = c.req.header("Authorization");
  if (!jwt) {
    c.status(401);
    return c.json({ error: "unauthorized" });
  }
  const token = jwt.split(" ")[1];
  const payload = await verify(token, c.env.JWT_SECRET);
  if (!payload) {
    c.status(401);
    return c.json({ error: "unauthorized" });
  }
  c.set("userId", payload.id);
  await next();
});
```

updated:

```ts
import { Hono } from "hono";
import { userRouter } from "./routes/user";
import { bookRouter } from "./routes/blog";

const app = new Hono<{
  Bindings: {
    DATABASE_URL: string;
    JWT_SECRET: string;
  };
  Variables: {
    userId: string;
  };
}>();

app.use("/api/v1/blog/*", async (c, next) => {
  const jwt = c.req.header("Authorization");
  if (!jwt) {
    c.status(401);
    return c.json({ error: "unauthorized" });
  }
  const token = jwt.split(" ")[1];
  const payload = await verify(token, c.env.JWT_SECRET);
  if (!payload) {
    c.status(401);
    return c.json({ error: "unauthorized" });
  }
  c.set("userId", payload.id);
  await next();
});

//POST /api/v1/signup
app.post("/api/v1/signup", async (c) => {
  const prisma = new PrismaClient({
    datasourceUrl: c.env.DATABASE_URL,
  }).$extends(withAccelerate());

  const body = await c.req.json();

  const user = await prisma.user.create({
    data: {
      email: body.email,
      password: body.password,
    },
  });

  if (!user) {
    c.status(403);
    return c.json({ error: "user not found" });
  }

  //@ts-ignore- to ignore the checks
  const token = await sign({ id: user.id }, c.env.JWT_SECRET);
  return c.json({
    jwt: token,
  });
});

//POST /api/v1/signin
app.post("/api/v1/signin", async (c) => {
  const prisma = new PrismaClient({
    datasourceUrl: c.env?.DATABASE_URL,
  }).$extends(withAccelerate());

  const body = await c.req.json();
  const user = await prisma.user.findUnique({
    where: {
      email: body.email,
    },
  });

  if (!user) {
    c.status(403);
    return c.json({ error: "user not found" });
  }

  const jwt = await sign({ id: user.id }, c.env.JWT_SECRET);
  return c.json({ jwt });
});

//POST /api/v1/blog
//confirm if user is able to access authorised routes
app.post("/api/v1/blog", (c) => {
  console.log(c.get("userId"));
  return c.text("signin route");
});

//PUT /api/v1/blog
app.put("/api/v1/blog", (c) => {
  return c.text("put Page!");
});

//GET /api/v1/blog/:id
app.get("/api/v1/blog/:id", (c) => {
  return c.text("blog page!");
});

//limit middlewares
app.use("/api/v1/blog/*", async (c, next) => {
  const jwt = c.req.header("Authorization");
  if (!jwt) {
    c.status(401);
    return c.json({ error: "unauthorized" });
  }
  const token = jwt.split(" ")[1];
  const payload = await verify(token, c.env.JWT_SECRET);
  if (!payload) {
    c.status(401);
    return c.json({ error: "unauthorized" });
  }
  c.set("userId", payload.id);
  await next();
});

export default app;
```

now update the different files as we don't want to keep every logic in index.ts, so we will import other files in it.

## Step 8 - Deploy app

```bash
npm run deploy
```

then go to the website, and cloudflarew for the app, then it's backend, from their to settings,go to variables section
now set the values from .env file in it.

## Step 9 - Zod Validation

deploying npm packages will be used.
We will divide our project into 3 parts
Backend
Frontend
common
common will contain all the things that frontend and backend want to share.
We will make common an independent npm module for now.
Eventually, we will see how monorepos make it easier to have multiple packages sharing code in the same repo.

Initialise common

```bash
mkdir common
cd common
npm init -y
npx tsc --init
```

tsconfig.json->

```json
"rootDir": "./src",
"outDir": "./dist",
```

1. Sign up/login to npmjs.org
   Run npm login
   Update the name in package.json to be in your own npm namespace, Update main to be dist/index.json

```json
{
  "name": "@nalindalal/medium-common",
  "version": "1.0.0",
  "description": "",
  "main": "dist/index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "keywords": [],
  "author": "",
  "license": "ISC"
"description": "",
  "dependencies": {
    "zod": "^3.23.8"
  }"
}
```

Add src to .npmignore

2. Install zod

```bash
npm i zod
```

Put all types in src/index.ts
1.signupInput / SignupInput
2.signinInput / SigninInput
3.createPostInput / CreatePostInput
4.updatePostInput / updatePostInput

to generate the output:

```bash
tsc -b
```

Publish to npm

```bash
npm publish --access public
```

Explore your package on npmjs

## FrontEnd

1.Initialize frontend

```bash
npm create vite@latest

> npx
> create-vite

✔ Project name: … frontend
✔ Select a framework: › React
✔ Select a variant: › TypeScript
```

make app.css, index.css empty; introduce tailwind

```bash
npm install -D tailwindcss postcss autoprefixer
npx tailwindcss init -p
```

update tailwind.config.js, index.css, empty the app.css

2. install your package,run project once locally

```bash
npm install @nalindalal/medium-common
npm run dev
npm i react-router-dom
```

add routing to the App.tsx file as signup,signin,blog

3. creating the components
   we created the Blog,Signin,Signup pages respectively

then regarding Quote it suggest that in right half their's a Quote ; above md visible but ideally invisible

same way the auth page

imp credentials

```md
postgres://avnadmin:AVNS_70RKF80OUVee9wYAozi@pg-3022e223-nalindalal2004-c2ec.l.aivencloud.com:28990/defaultdb?sslmode=require

DATABASE_URL="prisma://accelerate.prisma-data.net/?api_key=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhcGlfa2V5IjoiZWY0OGU4YTMtNzRhZC00MmVlLTgyNmUtZWZmMDExZjBkMTQ1IiwidGVuYW50X2lkIjoiYmNjZjJhN2JjNmVhZGY2MmI4OTA0Y2E3MmUxMjY1OGJmYzkyOGIzOWVlMGZmN2M1OTFjZjRlYWZkMDdiNjk2OCIsImludGVybmFsX3NlY3JldCI6ImQ0ZjM3MmUyLWVmYzItNDI5NS05M2E5LTE1ZTM5ZTdlNGE2ZiJ9.t-7aLAGMzWDj3IYWqqjDPEf9vZE2VbYDb8V5f0oOxqY"
```
