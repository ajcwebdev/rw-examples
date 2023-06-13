# RedwoodJS Weather Demo with Netlify Edge Functions

This is an example application demonstrating the use of the [RedwoodJS framework](https://redwoodjs.com/) and [Netlify Edge Functions](https://docs.netlify.com/netlify-labs/experimental-features/edge-functions/). The project uses an Edge Function to:

1. Determine your [geolocation](https://docs.netlify.com/netlify-labs/experimental-features/edge-functions/api/#netlify-specific-context-object)
2. Fetch your local weather forecast

It utilizes a standard [serverless function](https://redwoodjs.com/docs/serverless-functions) to query the OpenWeather API.

## Outline

- [Requirements](#requirements)
- [Set the OpenWeather API Key](#set-the-openweather-api-key)
  - [RedwoodJS Environment Variables](#redwoodjs-environment-variables)
  - [Netlify Environment Variables](#netlify-environment-variables)
- [Database Setup](#database-setup)
  - [Run Database Locally on MacOS](#run-database-locally-on-macos)
  - [Run Database on Railway](#run-database-on-railway)
  - [Populate Seed Data After Database Setup](#populate-seed-data-after-database-setup)
- [Configure Edge Functions](#configure-edge-functions)
  - [Develop Edge Functions Locally](#develop-edge-functions-locally)
- [Dataset](#dataset)
  - [WorldCity GraphQL Schema](#worldcity-graphql-schema)
- [GraphQL Queries](#graphql-queries)
  - [Get All World Cities](#get-all-world-cities)
  - [Search World Cities By Name](#search-world-cities-by-name)
  - [Get City By ID](#get-city-by-id)

## Requirements

- A Netlify account, [sign up for free here](https://app.netlify.com/signup)
- An OpenWeather API key, [get one here](https://openweathermap.org/api)
- A Postgres database, either by:
  - Setting one up [locally](#run-database-locally-on-macos)
  - Using [Railway](https://railway.app/) to [provision a temporary one](#run-database-on-railway)
  - Deploying an instance with any other PostgreSQL hosting provider such as [Supabase](https://supabase.com/), [Fly](https://fly.io/), or [Neon](https://neon.tech/)

## Set the OpenWeather API Key

Once you've gotten an API key from OpenWeather, you'll need to set it as an environment variable for your site (`OPEN_WEATHER_API_KEY`). This can be done through either [RedwoodJS environment variables](https://redwoodjs.com/docs/environment-variables) or [Netlify environment variables](https://docs.netlify.com/configure-builds/environment-variables/#declare-variables).

### RedwoodJS Environment Variables

```toml
[web]
  title = "Redwood App"
  port = 8910
  apiUrl = "/.netlify/functions"
  includeEnvironmentVariables = [ "OPEN_WEATHER_API_KEY" ]
```

### Netlify Environment Variables

Install the [`netlify-cli`](https://docs.netlify.com/cli/get-started/#installation) globally and run the following command to initialize a Netlify project (you will need to first initialize a Git repository if you have not already done so):

```bash
ntl init
```

Import your `.env` file with [`ntl env:import`](https://cli.netlify.com/commands/env#envimport):

```bash
ntl env:import .env
```

You can check that your environment variables where set correctly by running `ntl env:list` to list all variables.

## Database Setup

It's recommended to use [Postgres](https://www.postgresql.org/) for your database instead of SQLite because:

1. Postgres has stronger support for [types in our schema including `BigInt` and `Float`](https://stackoverflow.com/questions/7337882/what-is-the-difference-between-sqlite-integer-data-types-like-int-integer-bigi).
2. The seed script uses Prisma's [`createMany()`](https://www.prisma.io/docs/reference/api-reference/prisma-client-reference#createmany) to bulk load records in batches. This is significantly faster that loading data row by row which is important because we have ~43,000 world cities in the dataset. Unfortunately, `createMany` is not supported by SQLite. If you use a SQLite database, loading records will take significantly longer.

### Run Database Locally on MacOS

The easiest way to get Postgres up and running locally on OSX is by using the [Postgres.app](https://postgresapp.com) desktop client. After installation, you can set your database configuration settings to two different local databases like so:

```
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/redwoodjs-weather-demo-test
TEST_DATABASE_URL=postgresql://postgres:postgres@localhost:5432/redwoodjs-weather-demo-test
```

### Run Database on Railway

If you'd rather not worry about setting up a Postgres database on your computer, you can quickly provision one on [Railway for free](https://railway.app/). The database will be torn down after 24 hours unless claimed with an account. After provisioning a database on Railway:

1. Copy the 'database connection URL' that Railway provides
2. Use this as your `DATABASE_URL` value in your local environment

If you have a Railway account and have installed the Railway CLI you can run the following commands to set up a database in your project:

```bash
railway login
railway init
railway add
echo DATABASE_URL=`railway variables get DATABASE_URL` > .env
```

### Populate Seed Data After Database Setup

Regardless of where the database is set up, you'll need to also do the following:

1. Apply the Prisma schema to the provisioned database with `db push`
2. Run the seed script to populate the city data into the database with `db seed`

```bash
yarn
yarn rw prisma db push
yarn rw prisma db seed
```

## Configure Edge Functions

When a request is made to the Redwood GraphQL API, we want to invoke the `geolocate` Edge Function as part of the request. To achieve this, include the following `path` and `function` in your `netlify.toml` file:

```toml
# netlify.toml

[[edge_functions]]
  path = "/.netlify/functions/graphql"
  function = "geolocate"
```

To test your `geolocate` edge function, you will need to:

1. Place the Edge Function in `netlify/edge-functions`.
2. Set the `path` value to your serverless function's path in `netlify.toml`. Use the deploy path where the functions are located in your project (for example `/.redwood/functions/` or `/.netlify/functions/`).
3. Set the `function` value to the name of your Edge Function.

```toml
# netlify.toml

[[edge_functions]]
  path = "/.netlify/functions/weather"
  function = "geolocate"
```

Additional documentation on Edge Functions can be found [here](https://docs.netlify.com/netlify-labs/experimental-features/edge-functions/get-started/).

### Develop Edge Functions Locally

To test Netlify Edge Functions locally, run the project with [`ntl dev`](https://github.com/netlify/cli/blob/main/docs/netlify-dev.md) instead of `yarn rw dev`:

```bash
ntl dev
```

> *Note: If editing code while running `ntl dev`, you may get a warning that the port is in use when the API server tries to restart. You will need to stop and restart `ntl dev` manually.*

This will start and serve your app on [localhost:8888](http://localhost:8888).

<p align="center">
  <img
    width="400"
    alt="screenshot of localhost 8888 with working current weather query"
    src="https://user-images.githubusercontent.com/12433465/194445992-64394300-97b5-4d69-8cd7-20c9f714d582.png"
  >
</p>

The port is specified in `netlify.toml`. In this example, `targetPort` is set to the `web` side port as defined in `redwood.toml` (`8910`) and the Netlify Dev `port` is set to `8888`.

```toml
# netlify.toml

[dev]
  framework = "redwoodjs"
  targetPort = 8910
  port = 8888
```

See the [Netlify Dev](https://docs.netlify.com/configure-builds/file-based-configuration/#netlify-dev) section on Netlify's [File-based configuration](https://docs.netlify.com/configure-builds/file-based-configuration/) documentation for more information on configuring your local development environment.

## Dataset

This project uses the [World Cities Dataset](https://simplemaps.com/data/world-cities) from [SimpleMaps](https://simplemaps.com) which "contains demographic details of about 15,000 cities around the world. The location of the cities, the countries to which the City belongs to, its populations etc."

See [Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/) for information related to the license.

### WorldCity GraphQL Schema

See the `worldCities.sdl.ts` file in the `api/src/graphql` directory for the entire schema including input types.

```graphql
type WorldCity {
  id: String!
  createdAt: DateTime!
  updatedAt: DateTime!
  simpleMapsId: BigInt!
  city: String!
  cityAscii: String!
  lat: Float!
  lng: Float!
  country: String!
  iso2: String!
  iso3: String!
  adminName: String
  capital: String
  population: Int
  WeatherReport: [WeatherReport]!
}
```

Our queries and mutations are based on the standard SDL convention generated by Redwood's CLI.

```graphql
type Query {
  worldCities: [WorldCity!]!
    @requireAuth
  worldCity(id: String!): WorldCity
    @requireAuth
}

type Mutation {
  createWorldCity(input: CreateWorldCityInput!): WorldCity!
    @requireAuth
  updateWorldCity(id: String!, input: UpdateWorldCityInput!): WorldCity!
    @requireAuth
  deleteWorldCity(id: String!): WorldCity!
    @requireAuth
}
```

## GraphQL Queries

While RedwoodJS ships with a developer GraphQL Playground, the playground doesn't launch properly when running the app under `netlify cli`. Instead, you can try using either:
- A GraphQL IDE like [Altair](https://altairgraphql.dev/).
- A general API client with GraphQL integration like [Insomnia](https://docs.insomnia.rest/insomnia/graphql-queries), [Postman](https://learning.postman.com/docs/sending-requests/graphql/graphql/), or [RapidAPI](https://docs.rapidapi.com/docs/graphql-apis).

It is also possible to send GraphQL queries directly through cURL with a POST request like so:

```bash
curl \
  --header 'content-type: application/json' \
  --url 'http://localhost:8888/.netlify/functions/graphql' \
  --data '{"query":"{ worldCity(id:\"1de7d874-cffb-4884-9473-c102f91e28de\") { city } }"}'
```

This will output the following:

```json
{
	"data": {
		"worldCity": {
			"city": "Chicago"
		}
	}
}
```

If you want to test a deployed endpoint, replace the localhost URL in the previous command with your endpoint URL.

### Get All World Cities

```graphql
query AllCities {
  worldCities {
    id
    city
    cityAscii
    country
    lat
    lng
  }
}
```

### Search World Cities By Name

```graphql
query CityByName {
  searchWorldCities(search: { city: "Boston" }) {
    city
    country
    lat
    lng
    adminName
    population
  }
}
```

### Get City By ID

```graphql
query CityById {
  worldCity(id: "aa7d246e-eb86-44ac-8cdb-ecf9c78542de") {
    city
    country
    lat
    lng
    adminName
    population
  }
}
```