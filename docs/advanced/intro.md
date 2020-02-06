## Advanced example

[back to homepage](../../README.md)

### main project folders:

- [./examples/advanced/client](./examples/advanced/client)
- [./examples/advanced/server](./examples/advanced/server)

### Server configuration, basic setup for using Apollo Server with Express.

if you will have express application as I have, you will need install graphql packages:<br>

```
npm install apollo-server-express graphql
```

For start you need define import functions from `apollo-server-express`<br>

```js
const { ApolloServer, gql } = require('apollo-server-express');
```

after that you should type definitions by using the `gql` tag function, here we can call it as a regular function. The `fs.readFileSync` function to read our `schema.graphql` file.<br>

```js
const fs = require('fs');
const typeDefs = gql(fs.readFileSync('./schema.graphql', { encoding: 'utf8' }));
```

Then you need define a `resolvers` object, better if you require it like this:

```js
const resolvers = require('./resolvers');
```

Now you can create a new `ApolloServer` instance by passing a configuration object with our `typeDefs` and `resolvers`. Next you need to plug `apolloServer` into our existing Express application. You should do that by calling the `applyMiddleware` method and here you should pass an object with our `app`, this is the Express app defined above. Optionally, you can also set a `path`, that is where you want to expose the GraphQL endpoint on your server. This is set to `/graphql` as default.<br>

```js
const apolloServer = new ApolloServer({ typeDefs, resolvers });
apolloServer.applyMiddleware({ app, path: '/graphql' });
```

<hr>

### TypeDefs configuration, graphql schema for `jobs` and get every `job` fields

Best practice if you write your schema in separate file and call it [schema.graphql](./example//advanced/server/schema.graphql). You can start write your schema with a `type Query`, as always. Here you should add queries that return data. Data for my example are a list of jobs. Each job should have an `id`, a `title`, a `company` and a `description`.<br>

First of all you need to define a new type called `Job`. I want to be able to fetch some `jobs` which is an array of `type Job` elements.<br>

```graphql
type Query {
  jobs: [Job]
}
type Job {
  #TODO
}
```

Now I can define the `Job` fields. I certainly want an id to uniquely identify each job. Graphql provides a built-in type for `ID` just like it has `String`, `Boolean`, and so on.
By adding an exclamation mark after ID in the `type Job` definition, we set a constraint that ID requires a value `!= null` (i.e., ID value is not optional).<br>

```graphql
type Query {
  jobs: [Job]
}

type Job {
  id: ID! # <-- here
}
```

Next we want a `title` that is a `String` and a `description` that is also a `String`.<br>

```graphql
type Query {
  jobs: [Job]
}

type Job {
  id: ID!
  title: String # <-- here
  description: String # <-- here
}
```

### Create resolver function `jobs` which will return an array of `jobs`

For type `jobs` I need to define function which will return an array of `jobs`. This project has [db.js](examples/advanced/server/db.js) file.<br>

In this project I use a package [notarealdb](https://www.npmjs.com/package/notarealdb), it lets you create a collection for each object type you want to store like `jobs` and each collection is stored in its own JSON file so jobs are saved in the [jobs.json](././..examples/advanced/server/data/jobs.json) file. There are all the jobs data.<br>

Let's go to [resolver.js](././..examples/advanced/server/resolver.js) file and you need to import `db` module and define return `db.jobs.list()` for jobs variale in the constant Query.<br>

```js
const db = require('./db');

const Query = {
  jobs: () => db.jobs.list()
};

module.exports = { Query };
```

#### Running server

```bash
npm run start
```

or

```bash
yarn start
```

after running the server, you can open GraphQl playground. It should be accessible at this url: [http://localhost:9000/graphql](http://localhost:9000/graphql)<br>

for `type Query`, you can define resolver `Query` with function `jobs` in [resolvers.js](../../examples/advanced/server/resolvers.js) like this:<br>

### Graphql query for each `jobs` and get every job fields (id, title, description):

```graphql
{
  jobs {
    id
    title
    description
  }
}
```

### Create associations between different objects, between `Job` and `Company`:

If you check [companies.json](././..examples/advanced/server/data/companies.json) file you will see that, each company has an `id`, then the `name` of the company and a `description`.<br>

So this fields should be to return from our GraphQL server. Let's go to [schema.graphql](../../examples/advanced/server/schema.graphql) file and define a new `type Company` with the same fields. There is an `id` which is of type `ID` and should be not `null` just like for the `type Job` then a `name` that is a `String` and a `description` that's also a `String`. So that's my Company type defined.<br>

Now I added a `company` field to `Job` and the type of this field is `Company` so in a custom type, like `Job`, I used another custom type, like `Company` that's the way I represent associations between different objects.<br>

```graphql
type Query {
  jobs: [Job]
}

type Company { # <-- here
  id: ID!
  name: String
  description: String
}

type Job {
  id: ID!
  title: String
  # you can add a company field to Job between different objects
  company: Company # <-- here
  description: String
}
```

So you now have two custom types with one referencing the other.<br>

### Create resolver function `company` which will return the company for each job

Each object in [jobs.json](././..examples/advanced/server/data/jobs.json) array has a companyId and each companyId matches the id of a company in the [companies.json](././..examples/advanced/server/data/companies.json) file. So the companyId in the jobs collection is what you would call a foreign key in relational databases. you can use it as the reference to fetch the company object with the same id. You can get from the database have a companyId property but what we declare in our schema is that each Job object will have a `company` field that is a full `Company` object it's not just a `companyId`. Company is a field of the Job type.<br>

So let's go to [resolver.js](././..examples/advanced/server/resolver.js) file and you can do it like this:<br>

```js
const db = require('./db');

const Query = {
  jobs: () => db.jobs.list()
};

const Job = {
  // <-- here
  company: job => db.companies.get(job.companyId)
};

module.exports = { Query, Job };
```

### Test Graphql query for each `jobs` and get every job fields (id, title, description, company) with company fields (id, name, description)

```graphql
{
  jobs {
    id
    title
    description
    company {
      id
      name
      description
    }
  }
}
```

<hr>

### In the client side project create function for fetching `Jobs` and ready to use in `JobBoard` component

In [requests.js](././..examples/advanced/client/src/requests.js) file I defined async function `loadJobs()` that will call the server and fetch the jobs data.<br>

```js
const endpointURL = 'http://localhost:9000/graphql';

export async function loadJobs() {
  const response = await fetch(endpointURL, {
    method: 'POST',
    headers: { 'content-type': 'application/json' },
    body: JSON.stringify({
      query: `
        {
          jobs {
            id
            title
            company {
              id
              name
            }
          }
        }
      `
    })
  });
  const responseBody = await response.json();
  return responseBody.data.jobs;
}
```

### `JobBoard` component consume async function `loadJobs()` for show the job titles in the list instead of fake data

In [JobBoard.js](examples/advanced/client/src/JobBoard.js) file I defined import function `loadJobs` from [requests.js](././..examples/advanced/client/src/requests.js) file. It uses fetch command to make an HTTP request with the GraphQL query so that the jobs you can see in the frontend now use the data from the server returned from the GraphQL endpoint data.<br>

```js
import React, { useState, useEffect } from 'react';
import { JobList } from './JobList';
// const { jobs } = require('./fake-data');
import { loadJobs } from './requests';

export const JobBoard = () => {
  const [jobs, setJobs] = useState([]);

  useEffect(() => {
    const getJobs = async () => setJobs(await loadJobs());
    getJobs();
  });

  return (
    <div>
      <h1 className="title">Job Board</h1>
      <JobList jobs={jobs} />
    </div>
  );
};
```

<hr>

### `JobDetail` component update schema for get a single `job` data by its `id`

At the moment, the only `query` you can make is to get all the `jobs`. What you want to do is to add a new field to the `Query type` to get a single `job` however you need to specify which `job` as you don't want to get just any `job`. You want the one with a specific `id` in GraphQL. You can pass arguments to a `query` using this syntax inside parentheses you put the argument name, like `id` and then you declare its type that in this case is `ID` you can also add an exclamation mark to say that this argument cannot be null it's not possible to get a `job` without passing an `id` finally, outside parentheses you declare the return type of this field and what you want back is a `Job`. As you can see, this way to define a query in GraphQL is pretty similar to writing a function in JavaScript.<br>

So let's go to [schema.graphql](././..examples/advanced/server/schema.graphql) file and you can do it like this:<br>

```graphql
type Query {
  job(id: ID!): Job # <-- here
  jobs: [Job]
}

type Company {
  id: ID!
  name: String
  description: String
}

type Job {
  id: ID!
  title: String
  company: Company
  description: String
}
```

### `JobDetail` component needs update the ability to fetch a `job` by its `id` and handle it in the resolver function

First you need to go [resolver.js](././..examples/advanced/server/resolver.js) file. For our new query field let's add it inside the Query object together with `jobs` `job` is a function but this time you pass some GraphQL arguments how do you access them from a resolver? the first argument to a resolver function is the parent value that in this case is the `root` object you used the first argument when you wrote the resolver for the company field in the Job type in that case the parent is the job for which you want to resolve the company here however you're inside the Query type that is the root object that's why I called the parameter `root` you're not actually going to use that parameter anyway what you're really interested in is the second parameter, that is `args` this will contain the arguments passed in the GraphQL query next you need to return something from this function where that something is a job from the jobs collection in the db you want to `get` the one with the id passed in the arguments i.e. `args.id`. You could use object destructuring to extract the `id` directly here and then just pass `id` to `db.jobs.get`. I prefer this way because you can look at the resolver function and immediately see that it expects an id in the arguments.<br>

```js
const db = require('./db');

const Query = {
  // job: (root, args) => db.jobs.get(args.id),
  // to extract the `id` directly like this:
  job: (root, { id }) => db.jobs.get(id), // <-- here
  jobs: () => db.jobs.list()
};

const Job = {
  company: job => db.companies.get(job.companyId)
};

module.exports = { Query, Job };
```

### Test `JobDetail` component and his query `job` with the argument `id`

You can select `job` from the field list you can pass an argument by opening parentheses and write the argument name, that is `id` then colon and the value here you need a valid job id. GraphQL query strings must be enclosed in double quotes as you cannot use single quotes like in JavaScript. Next, this returns a Job object so you need to specify which fields you want. Let's pick the id and `title` for now, run this query and as a result you get a single job object.<br>

```graphql
{
  job(id: "rJKAbDd_z") {
    id
    title
  }
}
```

### Test `JobDetail` component and his query `job` with the argument `id` defined as variable

```graphql
query JobQuery($id: ID!) {
  job(id: $id) {
    id
    title
    description
    company {
      id
      name
    }
  }
}
```

query variables:<br>

```graphql
{
  "id": "rJKAbd_z"
}
```

### In the client side project create function for fetching data about `Job` by `id` variable and `JobDetail` component can use it.

In [requests.js](././..examples/advanced/client/src/requests.js) file I defined async function `loadJob()`. That's very similar to the previous HTTP request I wrote except that I am accepting a parameter which will call the server and fetch the job data by `id`.<br>

```js
const endpointURL = 'http://localhost:9000/graphql';

export async function loadJob() {
  const response = await fetch(endpointURL, {
    method: 'POST',
    headers: { 'content-type': 'application/json' },
    body: JSON.stringify({
      query: `
        query JobQuery($id: ID!) {
          job(id: $id) {
            id
            title
            description
            company {
              id
              name
            }
          }
        }
      `,
      variables: { id }
    })
  });
  const responseBody = await response.json();
  return responseBody.data.job;
}

export async function loadJobs() {
  const response = await fetch(endpointURL, {
    method: 'POST',
    headers: { 'content-type': 'application/json' },
    body: JSON.stringify({
      query: `
        {
          jobs {
            id
            title
            company {
              id
              name
            }
          }
        }
      `
    })
  });
  const responseBody = await response.json();
  return responseBody.data.jobs;
}
```

### `JobDetail` component consume async function `loadJob()` for show the job titles in the list instead of fake data

In [JobDetail.js](examples/advanced/client/src/JobDetail.js) file I defined import function `loadJob()` from [requests.js](././..examples/advanced/client/src/requests.js) file. I replaced the `fake-data` import with one for our new asynchrounous `loadJob` function that's exported by the `requests` module. I created the constructor where I defined state job with initial value of `null`. I used the `componentDidMount` method as async where will be called `loadJob()` function with defined argument `jobId` of the job to be loaded. This function need to await because it returns a promise with data of job and `setState` function will set job data for ready to use as state. If React tries to render this component when job is `null`, trying to access the `job.title` for example will cause an error; as a quick fix I wrote a check if `job` is not set then just return `null`.<br>

```js
import React, { useState, useEffect } from 'react';
import { Link } from 'react-router-dom';
// const { jobs } = require('./fake-data');
import { loadJob } from './requests';

export class JobDetail extends Component {
  constructor(props) {
    super(props);
    this.state = { job: null };
  }

  async componentDidMount() {
    const { jobId } = this.props.match.params;
    const job = await loadJob(jobId);
    this.setState({ job });
  }

  render() {
    const { job } = this.state;
    if (!job) {
      return null;
    }
    return (
      <div>
        <h1 className="title">{job.title}</h1>
        <h2 className="subtitle">
          <Link to={`/companies/${job.company.id}`}>{job.company.name}</Link>
        </h2>
        <div className="box">{job.description}</div>
      </div>
    );
  }
}
```

<hr>

### Handling GraphQL Error Responses

In [requests.js](././..examples/advanced/client/src/requests.js) file you can noticed there's a fair amount of duplication between two functions `fetch(...)` where is only query not similar, better way is to extract a new function as `graphqlRequest(...)` with all the common functionality since it's going to be similar to the existing ones. For function `graphqlRequest(...)` also I defined error handling if responseBody return errors.<br>

```js
const endpointURL = 'http://localhost:9000/graphql';

async function graphqlRequest(query, variables = {}) {
  const request = {
    method: 'POST',
    headers: { 'content-type': 'application/json' },
    body: JSON.stringify({ query, variables })
  };
  const response = await fetch(endpointURL, request);
  const responseBody = await response.json();
  if (responseBody.errors) {
    const message = responseBody.errors.map(error => error.message).join('\n');
    throw new Error(message);
  }
  return responseBody.data;
}

export async function loadJob(id) {
  const query = `query JobQuery($id: ID!) {
      job(id: $id) {
        id
        title
        description;
        company {
          id
          name
        }
      }
    }
  `;
  const { job } = await graphqlRequest(query, { id });
  return job;
}

export async function loadJobs() {
  const query = `
    {
      jobs {
        id
        title
        company {
          id
          name
        }
      }
    }
  `;
  const { jobs } = await graphqlRequest(query);
  return jobs;
}
```

<hr>

### Fetching a Company by ID

#### 1. Create schema - server side

Let's go to [schema.graphql](././..examples/advanced/server/schema.graphql) file and you can add a new field to my Query type called `company` and I added the `id` of the `company` as an argument. This will return a `Company` object it's very similar to getting a `job` by `id`.<br>

```graphql
type Query {
  company(id: ID!): Company # <-- here I added new field...
  job(id: ID!): Job
  jobs: [Job]
}

type Company {
  id: ID!
  name: String
  description: String
}

type Job {
  id: ID!
  title: String
  company: Company
  description: String
}
```

#### 2. Create resolver - server side

In [resolver.js](././..examples/advanced/server/resolver.js) file you can define `company` resolver function with the first parameter being the root object that it doesn't actually use and the second parameter is the GraphQL query variables that will contain the id. Next step, return the `company` with that `id`<br>

so that's `db.companies.get`, and the `id` that's it adding support for.<br>

```js
const db = require('./db');

const Query = {
  company: (root, { id }) => db.companies.get(id), // <-- like this
  job: (root, { id }) => db.jobs.get(id),
  jobs: () => db.jobs.list()
};

const Job = {
  company: job => db.companies.get(job.companyId)
};

module.exports = { Query, Job };
```

#### 3. Test in GraphQl playground

If you want to test it, you can copy this setup to GraphQl playground. It should be accessible at this url: [http://localhost:9000/graphql](http://localhost:9000/graphql)...<br>

```graphql
query CompanyQuery($id: ID!) {
  company(id: $id) {
    id
    name
    description
  }
}
```

in the `Query Variables` object you can set a value `id` like this:<br>

```graphql
{
  "id": "HJRa-DOuG"
}
```

#### 4. Create function loadCompany - client side

after that let's go to client side [requests.js](././..examples/advanced/client/src/requests.js) file and you can add a new function exported and async like the others and called loadCompany. This takes the id as parameter and here we need the query same as we tested in GraphQl playground.<br>

```js
const endpointURL = 'http://localhost:9000/graphql';

async function graphqlRequest(query, variables = {}) {
  const request = {
    method: 'POST',
    headers: { 'content-type': 'application/json' },
    body: JSON.stringify({ query, variables })
  };
  const response = await fetch(endpointURL, request);
  const responseBody = await response.json();
  if (responseBody.errors) {
    const message = responseBody.errors.map(error => error.message).join('\n');
    throw new Error(message);
  }
  return responseBody.data;
}

export async function loadJob(id) {
  const query = `query JobQuery($id: ID!) {
      job(id: $id) {
        id
        title
        description;
        company {
          id
          name
        }
      }
    }
  `;
  const { job } = await graphqlRequest(query, { id });
  return job;
}

export async function loadJobs() {
  const query = `
    {
      jobs {
        id
        title
        company {
          id
          name
        }
      }
    }
  `;
  const { jobs } = await graphqlRequest(query);
  return jobs;
}
// like this
export async function loadCompany(id) {
  const query = `query CompanyQuery($id: ID!) {
      company(id: $id) {
        id
        name
        description
      }
    }
  `;
  const { company } = await graphqlRequest(query, { id });
  return company;
}
```

#### 5. Use `loadCompany` function in the component CompanyDetail - client side

Last step is about updating the [CompanyDetail.js](examples/advanced/client/src/CompanyDetail.js) file. You can remove the `fake-data` import new `loadCompany` function from `./requests` after that let's update the constructor initially you can set the `company` to `null` in the state and, just like we did in the `JobDetail` component in the render method if the company data hasn't been loaded yet we return `null` to display nothing. Then we want to load the `company` and as usual for that we first need a `componentDidMount` method here we'll need the `companyId` and we can extracts it from the router params and fetch the `company` data from the server by calling the `loadCompany` function and passing `companyId` as an argument. Finally, we can update the component state with the company data we just fetched. It's pretty much the same code we wrote for the `JobDetail` component.<br>

```js
import React, { Component } from 'react';
// import { companies } from './fake-data'
import { loadCompany } from './requests';
import { JobList } from './JobList';

export class CompanyDetail extends Component {
  constructor(props) {
    super(props);
    this.state = {
      company: null
    };
  }

  async componentDidMount() {
    const { companyId } = this.props.match.params;
    const company = await loadCompany(companyId);
    this.setState({ company });
  }

  render() {
    const { company } = this.state;
    if (!company) {
      return null;
    }

    return (
      <div>
        <h1 className="title">{company.name}</h1>
        <div className="box">{company.description}</div>
      </div>
    );
  }
}
```

### Returning Jobs for a Company

#### 1. Create schema - server side

Let's go to [schema.graphql](./examples//advanced/server/schema.graphql) file and if I want to display on the company page the list of jobs currently available at that company just below the description how could you do that? If you look at query at the moment you can only request the company id, name, and description so if you want the jobs as well you'll need to start by modifying your schema here in the Company type you want to add a new field called `jobs` and its type is array of `Job` objects simple enough.<br>

```graphql
type Query {
  company(id: ID!): Company
  job(id: ID!): Job
  jobs: [Job]
}

type Company {
  id: ID!
  name: String
  description: String
  jobs: [Job] # <-- here I added a new field jobs
}

type Job {
  id: ID!
  title: String
  company: Company
  description: String
}
```

#### 2. Create resolver - server side

In [resolvers.js](././..examples/advanced/server/resolver.js) file first you need create `Company` variable and define it in the exported object straight away. Next you can write a resolver function for `jobs` and you need the first parameter that is the parent object in this case the parent is a `company`. It's kind of the opposite of what I have inside the `Job` object below where we resolve the `company`. For a `Job` variable here you need to resolve the `jobs` for a `company`, how could we do that? let's have a look at the data in [companies.json](././..examples/advanced/server/data/companies.json) each `company` object has only id, name, and description there's nothing about `jobs` you need to look at the `jobs` data instead and here each `job` object has a `companyId` property that you can use to find its associated `company` again. If you're familiar with relational databases you can think of this `companyId` as a foreign key so. Back to our resolver function, what we can do is to list all the `jobs` in the db, then -since `list()` returns an array- you can `filter` the `jobs` and only keep those where the `companyId` for this `job` is strictly equal to the id of the `company` you're looking for. This is a bit different from resolving a `company` for a `Job` because in that case, there's always a single `Company` associated with a given `Job` while in this case you can have multiple `jobs` associated with the same `Company`. If you're familiar with data modelling concepts, we have a one-to-many relationship between `Company` and `Job`.<br>

```js
const db = require('./db');

const Query = {
  company: (root, { id }) => db.companies.get(id),
  job: (root, { id }) => db.jobs.get(id),
  jobs: () => db.jobs.list()
};

const Company = {
  // <- resolver for Company new field jobs
  jobs: company => db.jobs.list().filter(job => job.companyId === company.id)
};

const Job = {
  company: job => db.companies.get(job.companyId)
};

module.exports = { Query, Company, Job };
```

#### 3. Test in GraphQl playground

Quick test in GraphQl playground. It should be accessible at this url: [http://localhost:9000/graphql](http://localhost:9000/graphql)...<br>

```graphql
query CompanyQuery($id: ID!) {
  company(id: $id) {
    id
    name
    description
    jobs {
      id
      title
    }
  }
}
```

in the `Query Variables` object you can set a value `id` like this:<br>

```graphql
{
  "id": "HJRa-DOuG"
}
```

#### 4. update function loadCompany with new field - client side

In the [requests.js](examples/advanced/client/src/requests.js) file when we load a company we need to add the new jobs field and for each job we want id and title.<br>

```js
const endpointURL = 'http://localhost:9000/graphql';

async function graphqlRequest(query, variables = {}) {
  const request = {
    method: 'POST',
    headers: { 'content-type': 'application/json' },
    body: JSON.stringify({ query, variables })
  };
  const response = await fetch(endpointURL, request);
  const responseBody = await response.json();
  if (responseBody.errors) {
    const message = responseBody.errors.map(error => error.message).join('\n');
    throw new Error(message);
  }
  return responseBody.data;
}

export async function loadJob(id) {
  const query = `query JobQuery($id: ID!) {
      job(id: $id) {
        id
        title
        description;
        company {
          id
          name
        }
      }
    }
  `;
  const { job } = await graphqlRequest(query, { id });
  return job;
}

export async function loadJobs() {
  const query = `
    {
      jobs {
        id
        title
        company {
          id
          name
        }
      }
    }
  `;
  const { jobs } = await graphqlRequest(query);
  return jobs;
}
// like this
export async function loadCompany(id) {
  const query = `query CompanyQuery($id: ID!) {
      company(id: $id) {
        id
        name
        description
        jobs {
          id
          title
        }
      }
    }
  `;
  const { company } = await graphqlRequest(query, { id });
  return company;
}
```

#### 5. update `loadCompany` function in the component CompanyDetail with new field - client side

I haven't described much about the user interface so far this project uses a CSS framework called Bulma which is like other CSS frameworks you might have used such as Bootstrap or Materialize so all these CSS classes used for styling come from the [Bulma](https://bulma.io/) stylesheet.<br>

we need to change the user interface a little bit it currently displays a title and then a box with the description we can show the jobs under the description.<br>

So we can use `JobList` component for display every company jobs and company name can be inside h5 html tag with bulma class `is-5`.<br>

```js
import React, { Component } from 'react';
import { loadCompany } from './requests';
import { JobList } from './JobList';

export class CompanyDetail extends Component {
  constructor(props) {
    super(props);
    this.state = {
      company: null
    };
  }

  async componentDidMount() {
    const { companyId } = this.props.match.params;
    const company = await loadCompany(companyId);
    this.setState({ company });
  }

  render() {
    const { company } = this.state;
    if (!company) {
      return null;
    }

    return (
      <div>
        <h1 className="title">{company.name}</h1>
        <div className="box">{company.description}</div>
        <h5 className="title is-5">Jobs at {company.name}</h5>
        <JobList jobs={company.jobs} />
      </div>
    );
  }
}
```

![1](./images/1.png)<br>

Getting the company for each job is obviously redundant because we know it will always be the same company we get as the top-level object but it proves that when we write a GraphQL query we have full flexibility in what we request as long as it matches the schema we can nest objects in any way we like we can go one level deeper by asking for the jobs of this company and then go on and ask for the company of each job and so on.<br>

GraphQL allows us to navigate our data in many different ways and with arbitrary level of nesting that's very powerful<br>

<hr>

### GraphQl Mutation - Creating a Job

`Mutation` is a root type just like Query.<br>

#### 1. Create schema - server side

How to modify data using GraphQL? The next feature we'll implement is posting a new job something that we can see in the frontend if we log in first we can log in. I'll use `bob@goobook.co` with password `bob123`. When you logged you will see a new link in the NavBar to `Post job` which takes us to this simple form where we can enter job title and description so when we click the Submit button we should call the GraphQL server and create a new job.<br>

![2](./images/2.png)<br>

In [schema.graphql](././..examples/advanced/server/schema.graphql) we currently have only queries we can get the list of `jobs` or a single `job`, or a `company` it's all about returning existing data there's nothing that modifies the data in GraphQL operations that modify the data are called `mutations`.<br>

Inside `Mutation` we can define `createJob` and mutations usually have arguments in this case we'll need to pass the data for the `job` to be created if we look at the `Job type` each `job` has an `id` however the id will be assigned by the server so we don't want to pass it from the client when creating a `job` we do want to supply a `title` so let's add that as an argument the `title` is a `String` similarly, we want the `description` that is another `String` what about the company? well Company is an object, that has many fields and we don't want to pass all these fields when creating a Job if we look at our jobs data, in the JSON file [jobs.json](././..examples/advanced/server/data/jobs.json) for each job we only need to save the companyId so in our schema when creating a job we can pass the `companyId`. that is of type `ID` finally, all mutations must return a result just like queries in this case at a minimum we may want to return the `job id` that's generated by the server so the result is of `type ID`.<br>

```graphql
type Query {
  company(id: ID!): Company
  job(id: ID!): Job
  jobs: [Job]
}

type Mutation { # <- here is mutation
  createJob(companyId: ID, title: String, description: String): ID
}

type Company {
  id: ID!
  name: String
  description: String
  jobs: [Job] # <-- here I added a new field jobs
}

type Job {
  id: ID!
  title: String
  company: Company
  description: String
}
```

#### 2. Create resolver - server side

In [resolver.js](././..examples/advanced/server/resolver.js) file we need to define a new object for the `Mutation type` and add it to the exported object at this point we can write a `Mutation` function called `createJob` the parameters we get are the same as any other resolver the first one is the parent object that in this case is the root and the second parameter is an object with the GraphQL arguments so in this case it contains `companyId` `title`, and `description` because these are the arguments we defined in the schema also we need a function body which create a new job and there is a `create` method in the `db.jobs`, so we can call `create` passing an object with all the properties we want to save i.e. `companyId`, `title`, and `description` same as the arguments we receive from GraphQL this will return a new `job id` so that's what we can return from our resolver function as well since in the schema we declared that this mutation returns an `ID`.<br>

```js
const db = require('./db');

const Query = {
  company: (root, { id }) => db.companies.get(id),
  job: (root, { id }) => db.jobs.get(id),
  jobs: () => db.jobs.list()
};

const Mutation = {
  // <- here
  createJob: (root, { companyId, title, description }) => {
    return db.jobs.create({ companyId, title, description });
  }
};

const Company = {
  jobs: company => db.jobs.list().filter(job => job.companyId === company.id)
};

const Job = {
  company: job => db.companies.get(job.companyId)
};

module.exports = { Query, Mutation, Company, Job };
```

#### 3. Test mutation query in GraphQl playground

```graphql
mutation {
  createJob(companyId: "SJV0-wdOM", title: "New Job", description: "TestA")
}
```

![3](./images/3.png)<br>

When we execute request a new job is created and the job id returned in the response in a client application after we create a job we may want to display the new job including all its fields so maybe it would better to return a Job object instead of just its id.<br>

### Best practices for mutations

#### 1. Update schema - server side

We can do changes for return a Job object instead of just its id in [schema.graphql](../../examples/advanced/server/schema.graphql) like this:<br>

```graphql
type Query {
  company(id: ID!): Company
  job(id: ID!): Job
  jobs: [Job]
}

type Mutation {
  # createJob(companyId: ID, title: String, description: String): ID
  createJob(companyId: ID, title: String, description: String): Job # <-- here
}

type Company {
  id: ID!
  name: String
  description: String
  jobs: [Job]
}

type Job {
  id: ID!
  title: String
  company: Company
  description: String
}
```

#### 2. Update resolver - server side

Now we need to update in [resolver.js](././..examples/advanced/server/resolver.js) file, the resolver function instead of returning the `id`. We need to return a `Job` object.<br>

So we can assign the `id` to a local variable first and then we can return the `job` in the `db` with that `id` and this will be a full `Job` object this should do.<br>

```js
const db = require('./db');

const Query = {
  company: (root, { id }) => db.companies.get(id),
  job: (root, { id }) => db.jobs.get(id),
  jobs: () => db.jobs.list()
};

const Mutation = {
  createJob: (root, { companyId, title, description }) => {
    // return db.jobs.create({ companyId, title, description });
    const id = db.jobs.create({ companyId, title, description }); // <-- here
    return db.jobs.get(id); // <-- here
  }
};

const Company = {
  jobs: company => db.jobs.list().filter(job => job.companyId === company.id)
};

const Job = {
  company: job => db.companies.get(job.companyId)
};

module.exports = { Query, Mutation, Company, Job };
```

#### 3. Test mutation query in GraphQl playground

This query can create new `job` with their details. If this job will be created successfully createJob will now return a `Job` object instead of just its `id`. Here we try return `id`, `title`, `comapny` object with `id`, `name`. So a client application could use this response data to display the `job` details without the need to make a separate query.<br>

```graphql
mutation {
  createJob(companyId: "SJV0-wdOM", title: "New Job", description: "TestB") {
    id
    title
    company {
      id
      name
    }
  }
}
```

![4](./images/4.png)

In response the object inside `data` is called `createJob` because that's the name of the mutation but if you look at this data it seems a bit strange because this object really is a `job` not a `createJob` so we can create graphql alias.<br>

#### 4. GraphQL supports aliases

It means we can give a different name to the result object like in this case we can tell it to call the result of createJob simply `job` and incidentally you can do this with both queries and mutations. The object in the reponse is called simply `job`<br>

```graphql
mutation {
  job: createJob(
    companyId: "SJV0-wdOM"
    title: "New Job"
    description: "TestB"
  ) {
    id
    title
    company {
      id
      name
    }
  }
}
```

![5](./images/5.png)

#### 5. Improve mutation arguments

I replaced 3 arguments `companyId`, `title` and `description` with a single argument `input` that is of type `CreateJobInput` in [schema.graphql](././..examples/advanced/server/schema.graphql).<br>

```graphql
type Query {
  company(id: ID!): Company
  job(id: ID!): Job
  jobs: [Job]
}

type Mutation {
  # createJob(companyId: ID, title: String, description: String): Job
  # best practice use input types
  createJob(input: CreateJobInput): Job
}

type Company {
  id: ID!
  name: String
  description: String
  jobs: [Job]
}

type Job {
  id: ID!
  title: String
  company: Company
  description: String
}

input CreateJobInput { # <- here
  companyId: ID
  title: String
  description: String
}
```

#### 6. Update resolver in createjob

in [resolver.js](././..examples/advanced/server/resolver.js) file we won't receive all defined arguments any more instead we get a single `input` argument so when we call `db.jobs.create` we can just pass `input` since `input` is already an object that contains `companyId`, `title`, and `description`.<br>

```js
const db = require('./db');

const Query = {
  company: (root, { id }) => db.companies.get(id),
  job: (root, { id }) => db.jobs.get(id),
  jobs: () => db.jobs.list()
};

const Mutation = {
  // createJob: (root, { companyId, title, description }) => {
  // const id = db.jobs.create({ companyId, title, description });
  createJob: (root, { input }) => {
    // <-- here
    const id = db.jobs.create(input); // <-- here
    return db.jobs.get(id);
  }
};

const Company = {
  jobs: company => db.jobs.list().filter(job => job.companyId === company.id)
};

const Job = {
  company: job => db.companies.get(job.companyId)
};

module.exports = { Query, Mutation, Company, Job };
```

#### 7. GraphQL test

now we can remove `companyId`, `title` and `description` arguments and we can use it as query variables.<br>

```graphql
mutation CreateJob($input: CreateJobInput) {
  job: createJob(input: $input) {
    id
    title
    company {
      id
      name
    }
  }
}
```

`Query Variables` object you can set a value `input` like this:<br>

```graphql
{
  "input": {
    "companyId": "SJV0-wdOM",
    "title": "New Job",
    "description": "TestC"
  }
}
```

![6](./images/6.png)<br>

#### 8. Create call the createJob mutation from the client

If we log in first as `bob@goobook.co` and with password `bob123`.<br>

![7](./images/7.png)<br>

We now see the Post Job link at the top and this is the form we want to modify so that it calls our GraphQL server on submit.

![8](./images/8.png)<br>

In [requests.js](././..examples/advanced/client/src/requests.js) file we can create a new async function `createJob(input)` and the data returned in the response will contain a job object. So we can extract the job that's the result of awaiting a call to `graphqlRequest` passing the `mutation` string and the `mutation` variables object that will contain the `input` parameter finally, we can return the job.<br>

```js
export async function createJob(input) {
  const mutation = `mutation CreateJob($input: CreateJobInput) {
      job: createJob(input: $input) {
        id
        title
        company {
          id
          name
        }
      }
    }
  `;
  const { job } = await graphqlRequest(mutation, { input });
  return job;
```

#### 9. Create call the createJob mutation from the client

We need define import for new function `createJob` from `request` and then we can use it in the handleClick method. We have access to the component state that contains title and description but when we call createJob we need to pass an input objet that also contains a companyId. For now we can define local variable called companyId. We will fix it later.<br>

we can get the title and the description from the component state and at this point when we call createJob we can pass an object with the companyId the title and the description createJob returns a promise so we can call then get the job data from the server. When we get the response
we display the JobDetail route.<br>

```js
import React, { Component } from 'react';
import { createJob } from './requests'; // <-- here

export class JobForm extends Component {
  constructor(props) {
    super(props);
    this.state = { title: '', description: '' };
  }

  handleChange(event) {
    const { name, value } = event.target;
    this.setState({ [name]: value });
  }

  handleClick(event) {
    event.preventDefault();
    // console.log('should post a new job:', this.state);
    const companyId = 'SJV0-wdOM'; // FIXME
    const { title, description } = this.state; // <-- here
    createJob({ companyId, title, description }).then(job => {
      // <-- here
      this.props.history.push(`/jobs/${job.id}`); // <-- here
    });
  }

  render() {
    const { title, description } = this.state;
    return (
      <div>
        <h1 className="title">New Job</h1>
        <div className="box">
          <form>
            <div className="field">
              <label className="label">Title</label>
              <div className="control">
                <input
                  className="input"
                  type="text"
                  name="title"
                  value={title}
                  onChange={this.handleChange.bind(this)}
                />
              </div>
            </div>
            <div className="field">
              <label className="label">Description</label>
              <div className="control">
                <textarea
                  className="input"
                  style={{ height: '10em' }}
                  name="description"
                  value={description}
                  onChange={this.handleChange.bind(this)}
                />
              </div>
            </div>
            <div className="field">
              <div className="control">
                <button
                  className="button is-link"
                  onClick={this.handleClick.bind(this)}
                >
                  Submit
                </button>
              </div>
            </div>
          </form>
        </div>
      </div>
    );
  }
}
```

### Authentication

#### 1. Enforcing authentication with the context

If we want to fix hard-coding `companyId`, we need add support for authentication which is a separate feature, but it's also a pre-requisite for knowing which company to associate to a new `job`. Support for authentication means that if we're not logged in then we shouldn't be able to create a new `job`. In the front-end if you're not logged in you don't see the `Post Job` link but that's not enough. Because somebody could call our GraphQL API directly like we can do from the Playground. we are able to create a job even without any authentication. We only want authenticated users to be able to post a job on our board. If we look at [server.js](./advanced/server/server.js) there's already some support for authentication here. We know that we can already log in in the frontend. In this file you can see a `/login` route at the bottom, that's what's called when we log in from the frontend. The code gets email and password from the request, looks in the db for a user with that email, if it can't find that user, or the password doesn't match, it sends a 401 Unauthorized response. Otherwise, if the login is valid, it sends back an access token generated with JWT. This project also uses the [express-jwt](https://github.com/auth0/express-jwt#readme) middleware to check if a user is authenticated and that depends on whether the request contains the token generated by the login endpoint. It's configured with `credentialsRequired` set to false, which means that by default.<br>

if `credentialsRequired` is `false` it means you don't have to be authenticated to access any route.
if you do send an access token the express-jwt middleware will check if it's valid, and if so extract your authentication details from the token.<br>

How can we access it in our resolvers? What we'd like to do is, in the `createJob` resolver, before we do anything else we want to check that the user is authenticated. How can we do that? The answer is that there's third parameter passed to a resolver function that is the `context`. This `context` can be used to access things that are not part of GraphQL itself but are provided by our application.<br>

```js
const db = require('./db');

const Query = {
  company: (root, { id }) => db.companies.get(id),
  job: (root, { id }) => db.jobs.get(id),
  jobs: () => db.jobs.list()
};

// so we can call this mutation and only log the context without actually modifying any data.
const Mutation = {
  createJob: (root, { input }, context) => {
    // <-- here
    // check user authenticated
    console.log('context:', context);
    // const id = db.jobs.create(input);
    // return db.jobs.get(id);
    return null;
  }
};

const Company = {
  jobs: company => db.jobs.list().filter(job => job.companyId === company.id)
};

const Job = {
  company: job => db.companies.get(job.companyId)
};

module.exports = { Query, Mutation, Company, Job };
```

If we go to the GraphQL Playground and run this mutation, we get "null" in the response, in line with our latest change. And if we look at our server logs what we see is that `context` is an object<br>

![9](./images/9.png)<br>

In the [server.js](..examples/advanced/server/server.js) when we create a new ApolloServer instance we pass our `typeDefs` and `resolvers`. Here we can pass another property that is called `context`. And this will be a function. As an argument to this function, here we'll receive an object that contains the request, or `req` for short. This is the Express request object, because we're using Apollo Server with Express. This function will return an object, which is the `context` object we'll get in the resolvers. And we can extract any information we're interested in from the HTTP request and make it available in the context visible to our resolvers. As a simple example, we could set a property called `method` with the value of the HTTP method from the request.<br>

```js
const fs = require('fs');
const { ApolloServer, gql } = require('apollo-server-express');

const bodyParser = require('body-parser');
const cors = require('cors');
const express = require('express');
const expressJwt = require('express-jwt');
const jwt = require('jsonwebtoken');
const db = require('./db');

const port = 9000;
const jwtSecret = Buffer.from('Zn8Q5tyZ/G1MHltc4F/gTkVJMlrbKiZt', 'base64');

const app = express();
app.use(
  cors(),
  bodyParser.json(),
  expressJwt({
    secret: jwtSecret,
    credentialsRequired: false // it means you don't have to be authenticated to access any route.
  })
);

const typeDefs = gql(fs.readFileSync('./schema.graphql', { encoding: 'utf8' }));
const resolvers = require('./resolvers');
const context = ({ req }) => ({ method: req.method }); // <-- here
const apolloServer = new ApolloServer({ typeDefs, resolvers, context });
apolloServer.applyMiddleware({ app, path: '/graphql' });

app.post('/login', (req, res) => {
  const { email, password } = req.body;
  const user = db.users.list().find(user => user.email === email);
  if (!(user && user.password === password)) {
    res.sendStatus(401);
    return;
  }
  const token = jwt.sign({ sub: user.id }, jwtSecret);
  res.send({ token });
});

app.listen(port, () =>
  console.info(`Server started on http://localhost:${port}/graphql`)
);
```

Right, let's save this file, reopen the terminal, and let's see what happens now when we submit our mutation request. You can see in the server logs that now the "context" contains a property called "method", with the value "POST".<br>
![10](./images/10.png)

Every HTTTP request has a method. If you have seen in the express-jwt documentation, that's set as the `req.user` property. Let's go try it.<br>

```js
const fs = require('fs');
const { ApolloServer, gql } = require('apollo-server-express');

const bodyParser = require('body-parser');
const cors = require('cors');
const express = require('express');
const expressJwt = require('express-jwt');
const jwt = require('jsonwebtoken');
const db = require('./db');

const port = 9000;
const jwtSecret = Buffer.from('Zn8Q5tyZ/G1MHltc4F/gTkVJMlrbKiZt', 'base64');

const app = express();
app.use(
  cors(),
  bodyParser.json(),
  expressJwt({
    secret: jwtSecret,
    credentialsRequired: false // it means you don't have to be authenticated to access any route.
  })
);

const typeDefs = gql(fs.readFileSync('./schema.graphql', { encoding: 'utf8' }));
const resolvers = require('./resolvers');
const context = ({ req }) => ({ user: req.user }); // <-- here
const apolloServer = new ApolloServer({ typeDefs, resolvers, context });
apolloServer.applyMiddleware({ app, path: '/graphql' });

app.post('/login', (req, res) => {
  const { email, password } = req.body;
  const user = db.users.list().find(user => user.email === email);
  if (!(user && user.password === password)) {
    res.sendStatus(401);
    return;
  }
  const token = jwt.sign({ sub: user.id }, jwtSecret);
  res.send({ token });
});

app.listen(port, () =>
  console.info(`Server started on http://localhost:${port}/graphql`)
);
```

So let's see what happens now when we send a GraphQL request.<br>

![11](./images/11.png)<br>

We get that `user` is `undefined`, which effectively means we're not authenticated. `user` will only be defined if we send a valid access token. And the access token is what is returned by the `/login` endpoint. It's created using the `jwt` library.<br>

When get that `user` is undefined, it means we're not authenticated. `user` will only be defined when we send a valid access token. if we log in in our aplication, `login` request POST in the response we can find the token generated by the server.<br>

Let's have a look at what happens when we log in in the frontend.<br>

![12](./images/12.png)<br>

Now, if we look at the `login` request, this is the POST request to `/login`.<br>

![13](./images/13.png)<br>

And in the response we can find the token generated by the server.<br>

![14](./images/14.png)<br>

What we should do with this token is send it back to the server in any other request we make to prove that we are authenticated. The same token is also available in the Local Storage, because the frontend application saves it there after a successful login response.<br>

![15](./images/15.png)<br>

The same token is also available in the Local Storage, because the frontend application saves it there after a successful login response. I'm going to copy this token from here, so we can send it with our request. Now, if we go back to the GraphQL Playground you can see that there's a tab called `HTTP Headers`, next to `Query Variables`. Here we can set a JSON object containing any HTTP header we want to send along with the GraphQL request. To authenticate ourselves we need to set the `authorization` header. And its value must be a string that starts with `Bearer`, a space and then the access token we obtained from the server after logging in. Let's run our mutation again.<br>

![16](./images/16.png)<br>

Open the terminal, and you can see that now the `context` contains a `user` object, with some properties like `sub` that is short for `subject` and is effectively the user id. So what happened is that the `express-jwt` middleware found the token we sent in the `authorization` header, it validate it, decoded it, put it on the request, and from there we set it on the context.<br>

So in [resolvers.js](././..examples/advanced/server/resolvers.js) we need to check if the user is authenticated. We can replace this comment with the actual implementation. If there is no `context.user` that means the user is not authenticated and therefore we can throw an Error. Let's just say `Unauthorized` for now. Throwing an error will cause the GraphQL server to also return an error response. Now, since the only thing we use from the `context` is `user`, I'm going to extract just that property from the parameter and simplify our code a tiny bit.<br>

```js
const db = require('./db');

const Query = {
  company: (root, { id }) => db.companies.get(id),
  job: (root, { id }) => db.jobs.get(id),
  jobs: () => db.jobs.list()
};

// so we can call this mutation and only log the context without actually modifying any data.
const Mutation = {
  // createJob: (root, { input }, context) => {
  // if (!context.user) {
  createJob: (root, { input }, { user }) => {
    // console.log('context:', context);
    if (!user) {
      throw new Error('Unathorized');
    }
    const id = db.jobs.create(input);
    return db.jobs.get(id);
    // return null;
  }
};

const Company = {
  jobs: company => db.jobs.list().filter(job => job.companyId === company.id)
};

const Job = {
  company: job => db.companies.get(job.companyId)
};

module.exports = { Query, Mutation, Company, Job };
```

When we not authenticated we get back an error response with a message saying `Unauthorized`.<br>

![17](./images/17.png)<br>

When we authenticated we get responseand this time it succeeded.<br>

![18](./images/18.png)<br>

#### 2. Passing authentication in HTTP requests

If we try posting a new job. I'll just enter some test values and click Submit as you can see this doesn't work any more we get an "Unauthorized" error this used to work but now that we added that check to our server it's refusing to create a job because we don't provide authentication we are logged in into this application otherwise we wouldn't even see the Post Job link in the NavBar<br>

![19](./images/19.png)<br>

![20](./images/20.png)<br>

We can modify the request in the [requests.js](././..examples/advanced/client/src/requests.js) file and if the user is logged in let's pretend we have this function defined already in this case we want to set a request header called `authorization` to be a string that start with `Bearer` and then we need the token again let's pretend we have a function that returns the access token so that's the sort of logic we need note that if the user is not logged in then we don't want to set any `authorization` header that's why I didn't simply add it to every request we need this if statement now, what about these two functions: isLoggedIn and getAccessToken how do we implement them? well, here's some good news: they already exist they're in the [auth.js](././..examples/advanced/client/src/auth.js) file. So all we need to do is import those functions into our requests file.<br>

```js
import { getAccessToken, isLoggedIn } from './auth'; // <-- here

const endpointURL = 'http://localhost:9000/graphql';

async function graphqlRequest(query, variables = {}) {
  const request = {
    method: 'POST',
    headers: { 'content-type': 'application/json' },
    body: JSON.stringify({ query, variables })
  };
  if (isLoggedIn()) {
    // <-- here
    request.headers['authoriyation'] = 'Bearer ' + getAccessToken();
  }
  const response = await fetch(endpointURL, request);
  const responseBody = await response.json();
  if (responseBody.errors) {
    const message = responseBody.errors.map(error => error.message).join('\n');
    throw new Error(message);
  }
  return responseBody.data;
}
```

#### 3. Extracting the Company from the Authenticated User

Here we will try to replace hard-coded `companyId` that happens to be the id for `Goobook`.
Let's go to our server project to the [requests](././..examples/advanced/server/resolvers.js). In the createJob resolver we now enforce authentication by checking if there is a `user` object on the context. Let's log this user object to the console to remind ourselves of what it contains. And then I'm just going to return null here, so it will skip the rest of the code for now.<br>

```js
const db = require('./db');

const Query = {
  company: (root, { id }) => db.companies.get(id),
  job: (root, { id }) => db.jobs.get(id),
  jobs: () => db.jobs.list()
};

const Mutation = {
  createJob: (root, { input }, { user }) => {
    console.log('user:', user); // <-- here
    return null; // <-- here
    if (!user) {
      throw new Error('Unathorized');
    }
    const id = db.jobs.create(input);
    return db.jobs.get(id);
  }
};

const Company = {
  jobs: company => db.jobs.list().filter(job => job.companyId === company.id)
};

const Job = {
  company: job => db.companies.get(job.companyId)
};

module.exports = { Query, Mutation, Company, Job };
```

Now if we send a new request, using the GraphQL Playground. Run this mutation.<br>

```graphql
mutation CreateJob($input: CreateJobInput) {
  job: createJob(input: $input) {
    id
    title
    company {
      id
      name
    }
  }
}
```

with query variables:<br>

```graphql
{
  "input": {
    "companyId": "SJV0-wdOM",
    "title": "New Job",
    "description": "TestC"
  }
}
```

and with http headers:<br>

```graphql
{
  "authorization": "Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJCSnJwLUR1ZEciLCJpYXQiOjE1ODA4MjcxNTl9.3Mlnu6KLEhH1aoHXySVwIqSlt9Ntg9ggoFrdrrnEOR4" # <-- here is generated id from local storage
}
```

In the server logs you can see that the `user` object contains the user id as the `sub` property. (Where `sub` is short for `subject`). Now, if we look at the user data in the JSON file. You can see that the id of the first user is the same as the one we saw in the log message.

![21](./images/21.png)<br>

Now we will try to create logic in the [resolvers.js](././..examples/advanced/server/resolvers.js). if I'm logged in as `alice@facegle` any job I create should have `Facegle` as the company. And if I am logged in as `bob@goobook` then I'll create jobs for the `Goobook` company.<br>

The only problem is that at the moment if we look at the user object logged to the console we don't have the `companyId` there. That's because the `user` property we put into the context right now is the `req.user` set by express-jwt which only contains information decoded from the token.<br>

It would be better if we could pass a user object loaded from the database instead. We can do that quite easily `req.user` contains a `sub` property that as we've seen in the server logs is effectively the user id. So we can load the user from the db with db.users.get passing the `sub`, that is the id, as argument. Now, there is a potential issue here: there's no guarantee that `req.user` will be set. If a user is not logged in on the frontend then the request will not include the access token and in that case `req.user` will be undefined. so this code would fail As a simple workaround, we can check if `req.user` is defined first and only in that case get the user from the db.<br>

```js
const context = ({ req }) => ({ user: req.user && db.users.get(req.user.sub) }); // <-- here
const apolloServer = new ApolloServer({ typeDefs, resolvers, context });
apolloServer.applyMiddleware({ app, path: '/graphql' });
app.post('/login', (req, res) => {
  const { email, password } = req.body;
  const user = db.users.list().find(user => user.email === email);
  if (!(user && user.password === password)) {
    res.sendStatus(401);
    return;
  }
  const token = jwt.sign({ sub: user.id }, jwtSecret);
  res.send({ token });
});
```
