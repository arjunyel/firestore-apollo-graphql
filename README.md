# firebase-firestore-graphql

An example of a [GraphQL](https://graphql.org/) setup with a Firebase Firestore backend. Uses [Apollo Engine/Server 2.0](https://www.apollographql.com/) and deployed to Google App Engine.

## Initial setup

```bash
npm init --yes
npm install apollo-server@beta firebase-admin graphql graphql-tag
npm install --save-dev typescript tslint
```

You'll also want to set up some scripts and other settings, as of writing here is what the package.json looks like

```json
{
  "name": "firebase-firestore-graphql",
  "scripts": {
      "build": "tsc",
      "serve": "npm run build && node lib/index.js",
      "start": "node lib/index.js",
      "deploy": "npm run build && gcloud app deploy"
  },
  "main": "lib/index.js",
  "dependencies": {
      "apollo-server": "^2.0.0-beta.10",
      "firebase-admin": "^5.12.1",
      "graphql": "^0.13.2",
      "graphql-tag": "^2.9.2"
  },
  "devDependencies": {
      "tslint": "^5.10.0",
      "typescript": "^2.9.1"
  }
}
```

## Firebase setup

Download Firebase service account as `service-account.json` and put in root of this directory.

In your firestore database setup two collections, one of tweets and one of users. The userId in tweets should point to a user Id that the tweet came from.

```typescript
interface User {
  id: string;
  name: string;
  screenName: string;
  statusesCount: number;
}

interface Tweet {
  id: string;
  name: string;
  screenName: string;
  statusesCount: number;
  userId: string;
}
```

## Typescript

Copy the tslint and tsconfig json files from this repo into your own.

## GraphQL

Make a src directory and a index.ts file inside. Setup the imports

```typescript
import * as admin from 'firebase-admin';

const serviceAccount = require('../service-account.json');

admin.initializeApp({
  credential: admin.credential.cert(serviceAccount)
});

import { ApolloServer, ApolloError, ValidationError, gql } from 'apollo-server';

interface User {
  id: string;
  name: string;
  screenName: string;
  statusesCount: number;
}

interface Tweet {
  id: string;
  name: string;
  screenName: string;
  statusesCount: number;
  userId: string;
}
```

## Schema

Now we setup our GraphQL schema

```typescript
const typeDefs = gql`
  # A Twitter User
  type User {
    id: ID!
    name: String!
    screenName: String!
    statusesCount: Int!
    tweets: [Tweets]!
  }

  # A Tweet Object
  type Tweets {
    id: ID!
    text: String!
    userId: String!
    user: User!
    likes: Int!
  }

  type Query {
    tweets: [Tweets]
    user(id: String!): User
  }
`;
```

The ! signifies that this property is guaranteed to not be null. You'll notice that a user has an array of Tweets and a tweet has a user object in it, despite them being separate collections in our database. This is the magic of GraphQL, we can combine things across collections.

For the purpose of this tutorial we have two queries, an array of all tweets and a specific user based on their ID.

## Resolver

Next we setup our resolver, this turns GraphQL queries into data. First we setup our resolver for the base queries

```typescript
const resolvers = {
  Query: {
    async tweets() {
      const tweets = await admin
        .firestore()
        .collection('tweets')
        .get();
      return tweets.docs.map(tweet => tweet.data()) as Tweet[];
    },
    async user(_: null, args: { id: string }) {
      try {
        const userDoc = await admin
          .firestore()
          .doc(`users/${args.id}`)
          .get();
        const user = userDoc.data() as User | undefined;
        return user || new ValidationError('User ID not found');
      } catch (error) {
        throw new ApolloError(error);
      }
    }
  }
};
```

This will get an array of tweets or a user but how do we add the graph part of GraphQL and interconnect different collections such as all the Tweets a user has made or the details of a user that made a certain tweet?

```typescript
const resolvers = {
  Query: {
    ...
  },
  User: {
    async tweets(user) {
      try {
        const userTweets = await admin
          .firestore()
          .collection('tweets')
          .where('userId', '==', user.id)
          .get();
        return userTweets.docs.map(tweet => tweet.data()) as Tweet[];
      } catch (error) {
        throw new ApolloError(error);
      }
    }
  },
  Tweets: {
    async user(tweet) {
      try {
        const tweetAuthor = await admin
          .firestore()
          .doc(`users/${tweet.userId}`)
          .get();
        return tweetAuthor.data() as User;
      } catch (error) {
        throw new ApolloError(error);
      }
    }
  }
};
```

Take getting all the tweets a user has made as an example. You can see in our resolver we have a user object with a tweets property. Because tweets is a child of user, we can use the parent user to then query the tweets collection for all the tweets with that user ID.

### Apollo Server

Finally we setup our Apollo server

```typescript
const server = new ApolloServer({
  typeDefs,
  resolvers,
  introspection: true
});

server.listen().then(({ url }) => {
  console.log(`🚀  Server ready at ${url}`);
});
```

If you setup your npm scripts you should be able to run

```bash
npm run serve
```

If you navigate to the URL you shoud be able to see a GraphQL playground where you can query your API, congrats!

## Apollo Engine

[Apollo Engine](https://www.apollographql.com/engine) gives use awesome features such as caching, tracing, and error logging. First get an [Apollo Engine API key](https://engine.apollographql.com/) then change your Apollo server config to turn on engine

```typescript
const server = new ApolloServer({
  typeDefs,
  resolvers,
  engine: {
    apiKey: "<APOLLO ENGINE API KEY HERE>"
  },
  introspection: true
});
```

Now when you npm serve and run some queries you should see some data populate the Apollo Engine dashboard with things like how fast your queries resolved. Cool!

## App Engine

Finally we can deploy to App engine so the world can access our GraphQL endpoint. In the root project folder create a file app.yaml. Inside is just one line

```yaml
runtime: nodejs8
```

Also add the .gcloudignore file from this repo to your folder. Setup the gcloud SDK then point it to your Firebase project.

```bash
gcloud config set project <projectID>
npm run build
gcloud app deploy
```

You should get a deployed URL, you can then query that using an GraphQL tool. I personally use [Insomnia’s GraphQL mode](https://support.insomnia.rest/article/61-graphql).

Congratulations, you've setup a GraphQL server!
