# Redwood and Next.js

## Setup

```bash
curl \
  --request POST \
  --header 'content-type: application/json' \
  --url 'http://localhost:8911/graphql' \
  --data '{"query":"{ redwood { version currentUser prismaVersion } }"}'
```

## Run API Development Server

```bash
yarn rw dev api
```