# Step 15: Using a REST API

[//]: # (head-end)


Despite using GraphQL throughout all our app, we will soon meet the need to use some external API and chances are it will be REST.
Our first idea could be to bridge the REST API through GraphQL, reproposing the very same API to the client. This approach is wrong, because our first concern should always be to provide the client with ready to use data in the best possible shape.
The client don’t need to know that our GraphQL API is backed by a REST API, it doesn’t have to pass headers required by the underlying API or do any kind of special considerations: our backend should take care of everything.

## Retrieve a profile picture from a REST API

In this chapter we will discuss how to use an external API called Unsplash to retrieve random profile pictures for the users who didn’t set any.

Start heading to https://unsplash.com/developers and clicking on “Register as a developer”. After registering you will have to create a new app: take note of the Access Key because we’re going to need it.

If you look at the Documentation (https://unsplash.com/documentation#get-a-random-photo) you’ll notice that in order to retrieve a random photo we have to query the /photos/random endpoint (GET method). We also have to pass some headers for the authent
cation and some params for the search term and the orientation.

On the browser we would probably use the fetch api, but since on we node we would need a polyfill it’s better to just use a full fledged library like axios:

    yarn add axios
    yarn add -D @types/axios

Before we start implementing, we want to create some typings for our endpoint, because ideally we would like to be aided by those typings during the development.
In order to do so we can use a Chrome extension like Advanced Rest Client to retrieve the response.
Set the Method to GET, the Headers to Authorization: 'Client-ID 4d048cfb4383b407eff92e4a2a5ec36c0a866be85e64caafa588c110efad350d' and the Request URL to https://api.unsplash.com/photos/random, along with the params to query: 'portrait' and orientation: 'squarish'.
Copy the response, create a new file called types/unsplash.ts in your vscode editor and run the command “Past JSON as Types” (you need to install the Past JSON as Code extension and press CTRL+P to open the run command prompt). That would be enough to automatically create the typings for the random photo endpoint.

Now we can finally implement the REST API call in our picture resolver:

[{]: <helper> (diffStep "12.1" files="schema/resolvers.ts" module="server")

#### [__Server__ Step 12.1: Retrieve profile picture from REST API](https://github.com/Urigo/WhatsApp-Clone-Server/commit/cb3da1cc8cd68cf69912eeac8e1c5c105b9ba1c5)

##### Changed schema&#x2F;resolvers.ts
```diff
@@ -7,6 +7,8 @@
 ┊ 7┊ 7┊import jwt from 'jsonwebtoken';
 ┊ 8┊ 8┊import { validateLength, validatePassword } from '../validators';
 ┊ 9┊ 9┊import sql from 'sql-template-strings';
+┊  ┊10┊import axios from 'axios';
+┊  ┊11┊import { RandomPhoto } from '../types/unsplash';
 ┊10┊12┊
 ┊11┊13┊const resolvers: Resolvers = {
 ┊12┊14┊  Date: DateTimeResolver,
```
```diff
@@ -71,7 +73,26 @@
 ┊71┊73┊
 ┊72┊74┊      const participant = rows[0];
 ┊73┊75┊
-┊74┊  ┊      return participant ? participant.picture : null;
+┊  ┊76┊      if (participant && participant.picture) return participant.picture;
+┊  ┊77┊
+┊  ┊78┊      try {
+┊  ┊79┊        return (await axios.get<RandomPhoto>(
+┊  ┊80┊          'https://api.unsplash.com/photos/random',
+┊  ┊81┊          {
+┊  ┊82┊            params: {
+┊  ┊83┊              query: 'portrait',
+┊  ┊84┊              orientation: 'squarish',
+┊  ┊85┊            },
+┊  ┊86┊            headers: {
+┊  ┊87┊              Authorization:
+┊  ┊88┊                'Client-ID 4d048cfb4383b407eff92e4a2a5ec36c0a866be85e64caafa588c110efad350d',
+┊  ┊89┊            },
+┊  ┊90┊          }
+┊  ┊91┊        )).data.urls.small;
+┊  ┊92┊      } catch (err) {
+┊  ┊93┊        console.error('Cannot retrieve random photo:', err);
+┊  ┊94┊        return null;
+┊  ┊95┊      }
 ┊75┊96┊    },
 ┊76┊97┊
 ┊77┊98┊    async messages(chat, args, { db }) {
```

[}]: #

In order to test it, we have to remove the picture from one of the users and re-run the server with the `RESET_DB=true` environment variable:

[{]: <helper> (diffStep "12.1" files="db.ts" module="server")

#### [__Server__ Step 12.1: Retrieve profile picture from REST API](https://github.com/Urigo/WhatsApp-Clone-Server/commit/cb3da1cc8cd68cf69912eeac8e1c5c105b9ba1c5)

##### Changed db.ts
```diff
@@ -123,6 +123,10 @@
 ┊123┊123┊    sql`SELECT setval('users_id_seq', (SELECT max(id) FROM users))`
 ┊124┊124┊  );
 ┊125┊125┊
+┊   ┊126┊  await pool.query(
+┊   ┊127┊    sql`SELECT setval('users_id_seq', (SELECT max(id) FROM users))`
+┊   ┊128┊  );
+┊   ┊129┊
 ┊126┊130┊  await pool.query(sql`DELETE FROM chats`);
 ┊127┊131┊
 ┊128┊132┊  const sampleChats = [
```

[}]: #


## Track the API

Even if our typings are working pretty well so far, not all REST APIs are versioned and the shape we’ve got from the server could potentially change.
In order to keep an eye on it we could use the safe-api middleware in order to check for abnormal answers coming from the server and log them. We can also generate the typings automatically based on the response we get.
First let’s install the safe-api middleware:

    yarn add @safe-api/middleware

Then let’s use it inside our resolver:

[{]: <helper> (diffStep "12.2" files="schema/resolvers.ts" module="server")

#### [__Server__ Step 12.2: Use safe-api](https://github.com/Urigo/WhatsApp-Clone-Server/commit/c5d349827bcfbb22056a125878c65f9d8ce6f7c1)

##### Changed schema&#x2F;resolvers.ts
```diff
@@ -9,6 +9,8 @@
 ┊ 9┊ 9┊import sql from 'sql-template-strings';
 ┊10┊10┊import axios from 'axios';
 ┊11┊11┊import { RandomPhoto } from '../types/unsplash';
+┊  ┊12┊import { trackProvider } from '@safe-api/middleware';
+┊  ┊13┊import { resolve } from 'path';
 ┊12┊14┊
 ┊13┊15┊const resolvers: Resolvers = {
 ┊14┊16┊  Date: DateTimeResolver,
```
```diff
@@ -75,20 +77,38 @@
 ┊ 75┊ 77┊
 ┊ 76┊ 78┊      if (participant && participant.picture) return participant.picture;
 ┊ 77┊ 79┊
+┊   ┊ 80┊      interface RandomPhotoInput {
+┊   ┊ 81┊        query: string;
+┊   ┊ 82┊        orientation: 'landscape' | 'portrait' | 'squarish';
+┊   ┊ 83┊      }
+┊   ┊ 84┊
+┊   ┊ 85┊      const trackedRandomPhoto = await trackProvider(
+┊   ┊ 86┊        async ({ query, orientation }: RandomPhotoInput) =>
+┊   ┊ 87┊          (await axios.get<RandomPhoto>(
+┊   ┊ 88┊            'https://api.unsplash.com/photos/random',
+┊   ┊ 89┊            {
+┊   ┊ 90┊              params: {
+┊   ┊ 91┊                query,
+┊   ┊ 92┊                orientation,
+┊   ┊ 93┊              },
+┊   ┊ 94┊              headers: {
+┊   ┊ 95┊                Authorization:
+┊   ┊ 96┊                  'Client-ID 4d048cfb4383b407eff92e4a2a5ec36c0a866be85e64caafa588c110efad350d',
+┊   ┊ 97┊              },
+┊   ┊ 98┊            }
+┊   ┊ 99┊          )).data,
+┊   ┊100┊        {
+┊   ┊101┊          provider: 'Unsplash',
+┊   ┊102┊          method: 'RandomPhoto',
+┊   ┊103┊          location: resolve(__dirname, '../logs/main'),
+┊   ┊104┊        }
+┊   ┊105┊      );
+┊   ┊106┊
 ┊ 78┊107┊      try {
-┊ 79┊   ┊        return (await axios.get<RandomPhoto>(
-┊ 80┊   ┊          'https://api.unsplash.com/photos/random',
-┊ 81┊   ┊          {
-┊ 82┊   ┊            params: {
-┊ 83┊   ┊              query: 'portrait',
-┊ 84┊   ┊              orientation: 'squarish',
-┊ 85┊   ┊            },
-┊ 86┊   ┊            headers: {
-┊ 87┊   ┊              Authorization:
-┊ 88┊   ┊                'Client-ID 4d048cfb4383b407eff92e4a2a5ec36c0a866be85e64caafa588c110efad350d',
-┊ 89┊   ┊            },
-┊ 90┊   ┊          }
-┊ 91┊   ┊        )).data.urls.small;
+┊   ┊108┊        return (await trackedRandomPhoto({
+┊   ┊109┊          query: 'portrait',
+┊   ┊110┊          orientation: 'squarish',
+┊   ┊111┊        })).urls.small;
 ┊ 92┊112┊      } catch (err) {
 ┊ 93┊113┊        console.error('Cannot retrieve random photo:', err);
 ┊ 94┊114┊        return null;
```

[}]: #

Now launch the client in order to retrieve the picture field multiple times.

If you look inside the logs directory you will notice that it generated some graphql schema to represent the REST API. You will notice that each time we call the REST endpoint it generates a new schema, because a single response isn’t generic enough to account for all possible responses. Ideally safe-api should be able to average multiple esponses in order to generate the least generic schema matching the given responses.

Now we need to remove `types/unsplash.ts` and generate some Typescript typings out of the schema. Do do so we can use the graphql-code-generator:

[{]: <helper> (diffStep "12.3" files=".gitignore, codegen.yml" module="server")

#### [__Server__ Step 12.3: Generate typings from safe-api](https://github.com/Urigo/WhatsApp-Clone-Server/commit/d46cd070541e6e4d401fd550fd3ae41335b334e7)

##### Changed .gitignore
```diff
@@ -1,4 +1,5 @@
 ┊1┊1┊node_modules
 ┊2┊2┊npm-debug.log
 ┊3┊3┊test-results/
-┊4┊ ┊types/graphql.d.ts🚫↵
+┊ ┊4┊types/graphql.d.ts
+┊ ┊5┊types/unsplash.d.ts🚫↵
```

##### Changed codegen.yml
```diff
@@ -1,7 +1,7 @@
-┊1┊ ┊schema: ./schema/typeDefs.graphql
 ┊2┊1┊overwrite: true
 ┊3┊2┊generates:
 ┊4┊3┊  ./types/graphql.d.ts:
+┊ ┊4┊    schema: ./schema/typeDefs.graphql
 ┊5┊5┊    plugins:
 ┊6┊6┊      - typescript
 ┊7┊7┊      - typescript-resolvers
```
```diff
@@ -17,3 +17,7 @@
 ┊17┊17┊        # e.g. Message.createdAt will be of type Date
 ┊18┊18┊        Date: Date
 ┊19┊19┊        URL: string
+┊  ┊20┊  ./types/unsplash.d.ts:
+┊  ┊21┊    schema: ./logs/main/Unsplash.RandomPhoto.graphql
+┊  ┊22┊    plugins:
+┊  ┊23┊      - typescript
```

[}]: #

    yarn codegen


## Apollo DataSources

We’re not done yet, there is still room for improvement. Instead of using axios, we could use Apollo’s Data Sources and take advantage of the built-in support for caching, deduplication and error handling.

    yarn remove axios @types/axios
    yarn add apollo-datasource-rest

[{]: <helper> (diffStep "12.4" files="schema/unsplash.api.ts" module="server")

#### [__Server__ Step 12.4: Use Apollo DataSources](https://github.com/Urigo/WhatsApp-Clone-Server/commit/86e1e39459fdaa093f2d6e364dd62caa2fabf83d)

##### Added schema&#x2F;unsplash.api.ts
```diff
@@ -0,0 +1,45 @@
+┊  ┊ 1┊import { RESTDataSource, RequestOptions } from 'apollo-datasource-rest';
+┊  ┊ 2┊import { resolve } from 'path';
+┊  ┊ 3┊import { trackProvider } from '@safe-api/middleware';
+┊  ┊ 4┊import { RandomPhoto } from '../types/unsplash';
+┊  ┊ 5┊
+┊  ┊ 6┊interface RandomPhotoInput {
+┊  ┊ 7┊  query: string;
+┊  ┊ 8┊  orientation: 'landscape' | 'portrait' | 'squarish';
+┊  ┊ 9┊}
+┊  ┊10┊
+┊  ┊11┊export class UnsplashApi extends RESTDataSource {
+┊  ┊12┊  constructor() {
+┊  ┊13┊    super();
+┊  ┊14┊    this.baseURL = 'https://api.unsplash.com/';
+┊  ┊15┊  }
+┊  ┊16┊
+┊  ┊17┊  willSendRequest(request: RequestOptions) {
+┊  ┊18┊    request.headers.set(
+┊  ┊19┊      'Authorization',
+┊  ┊20┊      'Client-ID 4d048cfb4383b407eff92e4a2a5ec36c0a866be85e64caafa588c110efad350d'
+┊  ┊21┊    );
+┊  ┊22┊  }
+┊  ┊23┊
+┊  ┊24┊  async getRandomPhoto() {
+┊  ┊25┊    const trackedRandomPhoto = await trackProvider(
+┊  ┊26┊      ({ query, orientation }: RandomPhotoInput) =>
+┊  ┊27┊        this.get<RandomPhoto>('photos/random', { query, orientation }),
+┊  ┊28┊      {
+┊  ┊29┊        provider: 'Unsplash',
+┊  ┊30┊        method: 'RandomPhoto',
+┊  ┊31┊        location: resolve(__dirname, '../logs/main'),
+┊  ┊32┊      }
+┊  ┊33┊    );
+┊  ┊34┊
+┊  ┊35┊    try {
+┊  ┊36┊      return (await trackedRandomPhoto({
+┊  ┊37┊        query: 'portrait',
+┊  ┊38┊        orientation: 'squarish',
+┊  ┊39┊      })).urls.small;
+┊  ┊40┊    } catch (err) {
+┊  ┊41┊      console.error('Cannot retrieve random photo:', err);
+┊  ┊42┊      return null;
+┊  ┊43┊    }
+┊  ┊44┊  }
+┊  ┊45┊}
```

[}]: #

We created the UnsplashApi class, which extends RESTDataSource. In the constructor you need to set the baseUrl (after calling super() to run the constructor of the base class). You also need to create a willSendRequest method to set the authentication headers for each call. Then it’s simply a matter of creating a getRandomPhoto method to perform the actual REST API call. Instead of calling axios you will have to call the get method of the class (which in turn gets inherited from its RESTDataSource base class): the API is very similar to the axios one.

In order to access the data source from the resolvers we need to tell Apollo to put them on the context for every request. We shouldn’t use the context field, because that would lead to circular dependencies. Instead we need to use the dataSources field:

[{]: <helper> (diffStep "12.4" files="index.ts" module="server")

#### [__Server__ Step 12.4: Use Apollo DataSources](https://github.com/Urigo/WhatsApp-Clone-Server/commit/86e1e39459fdaa093f2d6e364dd62caa2fabf83d)

##### Changed index.ts
```diff
@@ -8,6 +8,7 @@
 ┊ 8┊ 8┊import schema from './schema';
 ┊ 9┊ 9┊import { MyContext } from './context';
 ┊10┊10┊import sql from 'sql-template-strings';
+┊  ┊11┊import { UnsplashApi } from './schema/unsplash.api';
 ┊11┊12┊const { PostgresPubSub } = require('graphql-postgres-subscriptions');
 ┊12┊13┊
 ┊13┊14┊const pubsub = new PostgresPubSub({
```
```diff
@@ -67,6 +68,9 @@
 ┊67┊68┊
 ┊68┊69┊    return res;
 ┊69┊70┊  },
+┊  ┊71┊  dataSources: () => ({
+┊  ┊72┊    unsplashApi: new UnsplashApi(),
+┊  ┊73┊  }),
 ┊70┊74┊});
 ┊71┊75┊
 ┊72┊76┊server.applyMiddleware({
```

[}]: #

Now we need to update the typings for our context and run the graphq-code-generator again:

[{]: <helper> (diffStep "12.4" files="context.ts" module="server")

#### [__Server__ Step 12.4: Use Apollo DataSources](https://github.com/Urigo/WhatsApp-Clone-Server/commit/86e1e39459fdaa093f2d6e364dd62caa2fabf83d)

##### Changed context.ts
```diff
@@ -2,10 +2,14 @@
 ┊ 2┊ 2┊import { User } from './db';
 ┊ 3┊ 3┊import { Response } from 'express';
 ┊ 4┊ 4┊import { PoolClient } from 'pg';
+┊  ┊ 5┊import { UnsplashApi } from './schema/unsplash.api';
 ┊ 5┊ 6┊
 ┊ 6┊ 7┊export type MyContext = {
 ┊ 7┊ 8┊  pubsub: PubSub;
 ┊ 8┊ 9┊  currentUser: User;
 ┊ 9┊10┊  res: Response;
 ┊10┊11┊  db: PoolClient;
+┊  ┊12┊  dataSources: {
+┊  ┊13┊    unsplashApi: UnsplashApi;
+┊  ┊14┊  };
 ┊11┊15┊};
```

[}]: #

    yarn codegen

Now it should be pretty easy to modify our resolver in order to use our just created datasource:

[{]: <helper> (diffStep "12.4" files="schema/resolvers.ts" module="server")

#### [__Server__ Step 12.4: Use Apollo DataSources](https://github.com/Urigo/WhatsApp-Clone-Server/commit/86e1e39459fdaa093f2d6e364dd62caa2fabf83d)

##### Changed schema&#x2F;resolvers.ts
```diff
@@ -7,10 +7,6 @@
 ┊ 7┊ 7┊import jwt from 'jsonwebtoken';
 ┊ 8┊ 8┊import { validateLength, validatePassword } from '../validators';
 ┊ 9┊ 9┊import sql from 'sql-template-strings';
-┊10┊  ┊import axios from 'axios';
-┊11┊  ┊import { RandomPhoto } from '../types/unsplash';
-┊12┊  ┊import { trackProvider } from '@safe-api/middleware';
-┊13┊  ┊import { resolve } from 'path';
 ┊14┊10┊
 ┊15┊11┊const resolvers: Resolvers = {
 ┊16┊12┊  Date: DateTimeResolver,
```
```diff
@@ -64,7 +60,7 @@
 ┊64┊60┊      return participant ? participant.name : null;
 ┊65┊61┊    },
 ┊66┊62┊
-┊67┊  ┊    async picture(chat, args, { currentUser, db }) {
+┊  ┊63┊    async picture(chat, args, { currentUser, db, dataSources }) {
 ┊68┊64┊      if (!currentUser) return null;
 ┊69┊65┊
 ┊70┊66┊      const { rows } = await db.query(sql`
```
```diff
@@ -75,44 +71,9 @@
 ┊ 75┊ 71┊
 ┊ 76┊ 72┊      const participant = rows[0];
 ┊ 77┊ 73┊
-┊ 78┊   ┊      if (participant && participant.picture) return participant.picture;
-┊ 79┊   ┊
-┊ 80┊   ┊      interface RandomPhotoInput {
-┊ 81┊   ┊        query: string;
-┊ 82┊   ┊        orientation: 'landscape' | 'portrait' | 'squarish';
-┊ 83┊   ┊      }
-┊ 84┊   ┊
-┊ 85┊   ┊      const trackedRandomPhoto = await trackProvider(
-┊ 86┊   ┊        async ({ query, orientation }: RandomPhotoInput) =>
-┊ 87┊   ┊          (await axios.get<RandomPhoto>(
-┊ 88┊   ┊            'https://api.unsplash.com/photos/random',
-┊ 89┊   ┊            {
-┊ 90┊   ┊              params: {
-┊ 91┊   ┊                query,
-┊ 92┊   ┊                orientation,
-┊ 93┊   ┊              },
-┊ 94┊   ┊              headers: {
-┊ 95┊   ┊                Authorization:
-┊ 96┊   ┊                  'Client-ID 4d048cfb4383b407eff92e4a2a5ec36c0a866be85e64caafa588c110efad350d',
-┊ 97┊   ┊              },
-┊ 98┊   ┊            }
-┊ 99┊   ┊          )).data,
-┊100┊   ┊        {
-┊101┊   ┊          provider: 'Unsplash',
-┊102┊   ┊          method: 'RandomPhoto',
-┊103┊   ┊          location: resolve(__dirname, '../logs/main'),
-┊104┊   ┊        }
-┊105┊   ┊      );
-┊106┊   ┊
-┊107┊   ┊      try {
-┊108┊   ┊        return (await trackedRandomPhoto({
-┊109┊   ┊          query: 'portrait',
-┊110┊   ┊          orientation: 'squarish',
-┊111┊   ┊        })).urls.small;
-┊112┊   ┊      } catch (err) {
-┊113┊   ┊        console.error('Cannot retrieve random photo:', err);
-┊114┊   ┊        return null;
-┊115┊   ┊      }
+┊   ┊ 74┊      return participant && participant.picture
+┊   ┊ 75┊        ? participant.picture
+┊   ┊ 76┊        : dataSources.unsplashApi.getRandomPhoto();
 ┊116┊ 77┊    },
 ┊117┊ 78┊
 ┊118┊ 79┊    async messages(chat, args, { db }) {
```

[}]: #


[//]: # (foot-start)

[{]: <helper> (navStep)

| [< Previous Step](https://github.com/Urigo/WhatsApp-Clone-Tutorial/tree/master@next/.tortilla/manuals/views/step14.md) | [Next Step >](https://github.com/Urigo/WhatsApp-Clone-Tutorial/tree/master@next/.tortilla/manuals/views/step16.md) |
|:--------------------------------|--------------------------------:|

[}]: #
