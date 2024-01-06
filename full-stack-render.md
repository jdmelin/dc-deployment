# Full Stack App Deployment on Render

This guide assumes the use of the PERN stack (PostgreSQL, Express, React, Node), along with Sequelize.

## Project Structure

Structure your project this way:

```
client
node_modules
server
.env
.gitignore
package-lock.json
package.json
README.md
```

Your `client` folder is the entire React app (created with `create-react-app`, for example).

Your `server` folder is the entire Node/Express app.

(`client` and `server` aren't crucial directory names if you prefer something else)

Your `.env` file should have any environment variables you need to successfully run the app locally.  We'll get into the details a little later on. The `.env` will only exist locally. You will store the corresponding environment variables in the Render web app so that your production app can make use of them. 

Your `.gitignore` file should look something like this:

```
node_modules
build
.env
.env.local
.DS_Store
```

Details for your `.gitignore`:

`node_modules`: we don't want to track them with git

`build`: this is where the React production version lives, so if it gets built locally, we don't want to track it

`.env`: **crucial** -- we don't want any sensitive info tracked

`.env.local`: **crucial** -- if you're using this in your React app, we don't want any sensitive info tracked

`.DS_Store`: if you're using macOS

Your `package.json` file should look something like this:

```
{
  "name": "my-cool-project",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1",
    "setup": "npm i; cd server && npm i; cd ../client && npm i; npm run build",
    "dev": "concurrently \"cd server && npm run dev\" \"cd client && npm start\"",
    "start": "cd server && npm start"
  },
  "repository": {
    "type": "git",
    "url": ""
  },
  "author": "",
  "license": "ISC",
  "bugs": {
    "url": ""
  },
  "homepage": "",
  "dependencies": {
    "concurrently": "^8.0.1"
  }
}
```

The reason we have a `package.json` at the root level of our app structure is that we want to have scripts that can run the React app & the Node app at the same time. To do so, we'll use `concurrently` (`npm i concurrently`).

When developing, you can run `npm run dev`, which will simultaneously change directories into `server` and `client`, respectively, and run the relevant scripts. Keep in mind, each subdirectory has its own `package.json` (and scripts) since they're essentially standalone apps put in one main app that we're controlling here.

We'll come back to the `setup` and `start` scripts, which are written specifcally for setting up and running the app in Render. 

---

## Sign Up for Render

[Sign up](https://dashboard.render.com/register) for a Render account.

---

## Create a New PostgreSQL Instance

### In Render

- Click `New` and select `PostgreSQL` from the dropdown
- Fill out the form with a `Name` and `Database`
- Select a region that's closest to you geographically
- Make sure the Instance type is set to `Free`
- Click `Create Database`

### Locally

The following steps for connecting to the PostgreSQL database on Render assume you're using Sequelize.

First, you'll need to change the `config.json` file to `config.js` so that we can make some necessary adjustments to the code. You `config.js` should look something like this:

```
require('dotenv').config({
  path: '../.env',
});
module.exports = {
  development: {
    username: process.env.DEV_DB_USER,
    password: process.env.DEV_DB_PASSWORD,
    database: process.env.DEV_DB_NAME,
    host: process.env.DEV_DB_HOST,
    dialect: 'postgres',
  },
  test: {
    username: process.env.TEST_DB_USER,
    password: process.env.TEST_DB_PASSWORD,
    database: process.env.TEST_DB_NAME,
    host: process.env.TEST_DB_HOST,
    dialect: 'postgres',
  },
  production: {
    username: process.env.PROD_DB_USER,
    password: process.env.PROD_DB_PASSWORD,
    database: process.env.PROD_DB_NAME,
    host: process.env.PROD_DB_HOST,
    dialect: 'postgres',
    dialectOptions: {
      ssl: {
        rejectUnauthorized: false,
      },
    },
  },
};
```

(You'll also need to update `config.json` to `config.js` in `server/models/index.js`)

Now, run `npm i dotenv` to get your app to play nice with the `.env` file created earlier.

Next, put the environment variables relevant to PostgreSQL into your `.env` file.

For example:

```
DEV_DB_USER=postgres
DEV_DB_PASSWORD=null
DEV_DB_NAME=[your local database name]
DEV_DB_HOST=127.0.0.1
PROD_DB_USER=[your Render database user]
PROD_DB_PASSWORD=[your Render database password]
PROD_DB_NAME=[your Render database name]
PROD_DB_HOST=[your Render database host]
```

**Note**: Your `PROD_DB_HOST` value should be formatted similar to `abc-abc12abcdefghij1ab2c-a.ohio-postgres.render.com`.

Now, the config file will read in the `development` credentials when running locally and the `production` credentials when running in production on Render.

To get the PostgreSQL database instance on Render to actually have the tables and data to match your local PostgreSQL instance, you'll need to run these commands from your `server` directory:

```
npx sequelize-cli db:migrate --env production
npx sequelize-cli db:seed:all --env production
```

**Note**: Since the production database had already been created, you'll only need to migrate & seed. However, if you need to reset tables, reseed, etc., you'll need to use other sequelize-cli or SQL commands.

The PostgreSQL database instance on Render should now be ready for your app to use.

Also, if you want a local interface for your database, connect with Beekeeper Studio using the credentials created in Render.

---

## Prepare Codebase for Render

Assuming you followed the earlier steps for your app structure, there are only a handful of further configurations to make to get things up and running. We'll tweak our code a bit and then get things up in Render.

### Client

If your `client` React app runs successfully on its own, there's nothing you need to change inside that directory at this point. It's worth noting here that if you're using environment variables (which should be the case if you have sensitive data such as an API key), your `client` directory would have a `.env.local` file, for instance, where you've got something like this:

```
REACT_APP_BASE_URL=http://localhost:3001/api
REACT_APP_FINNHUB_API_KEY=123456
```

In the above example, the `client` React app makes use of two different variables: a base URL for the locally running API (which will be set differently in Render), and an API key. Your app should probably at least make use of some sort of URL variable so that your code can run smoothly whether locally or in production.

When running in production, our app is actually going to use the `.env` file at the root of the app (which will include variables that we'll later set in Render).

### Server

Regarding the `server` directory, we need to adjust our `index.js` file so that our app runs both locally and in production. We'll need to either add or adjust our `dotenv` configuration to look like this:

```
require('dotenv').config({
  path: '../.env',
});
```

We'll also need to change our PORT to not be hardcoded as `3001`, for example. So, we can do:

```
const port = process.env.PORT;
```

This way, Render can set the port automatically in production while we can just set it to `3001` (or whatever) in our `.env` file at the root of the entire app.

Next, we'll need to change this line: `app.use(express.static(path.join(__dirname, '../client/public')))`

Because our React app will be served from a `build` directory, we can write something like this:

```
let staticPath = '../client/public';

if (process.env.NODE_ENV === 'production') {
  staticPath = '../client/build';
}

app.use(express.static(path.join(__dirname, staticPath)));
```

In doing so, we're ensuring that it uses the `public` directory in development mode but `build` in production.

Additionally, in our catch all route, we can change `res.sendFile(path.join(__dirname, '../client/public/index.html'))` to `res.sendFile(path.join(__dirname, staticPath, 'index.html'))` to make use of the conditional logic that sets the `staticPath`.

### Last Local Steps

Make sure the app runs locally by running the `dev` script at the root of the project. By using `concurrently`, it runs both apps at the same time. 

Once everything is good to go, push your latest code to GitHub (or run through the whole Git flow if you haven't up to this point).

**Note**: If you haven't already done so... to get your server & client apps to play nice together, use the `cors` library on the server and make sure you have `app.use(cors())` in your `index.js`.

## Create New Web Service in Render

Now that your app is ready to go and in GitHub, you can hook it up with Render.

- Click `New` and select `Web Service` from the dropdown
- You can either connect Render with GitHub repos of your choice, or you can paste in the URL for your repo at the bottom of the page
- Next, create a `Name` for you service
- Select a region that's closest to you geographically
- For `Build command`, enter `npm run setup`
- For `Start command`, enter `npm start`
- Make sure the Instance type is set to `Free`
- Click `Create Web Service`

To explain the build & start commands, we want `npm run setup` to fire off whenever the app gets built. This script ensures that we install everything we need in the root directory, the `client` directory, and the `server` directory. It also builds the production version of the React app and puts it in a `build` folder for the server app to serve. The start command uses `npm start` to simply go into the `server` directory and start up the server app. 

Next, you'll need to enter the correct environment variables into Render:

- Click on the `Environment` tab on the sidebar
- Add `PROD_DB_HOST`, `PROD_DB_NAME`, `PROD_DB_PASSWORD`, and `PROD_DB_USER`, and copy the values from the `Connections` section in your PostgreSQL instance on Render (or from your local `.env` file).
- Add the React environment variables: `REACT_APP_BASE_URL` and `REACT_APP_WHATEVER_API_KEY` (whatever you have called yours), and any other variables you may have.

Finally, let your app auto-build or kick it off again with the `Manual Deploy` button. Once the build has completed (if it doesn't have errors), visit your masterpiece using the URL provided near the top left of the screen.

---

Congratulations! You successfully deployed a full stack app.