# Deploying a Rails-React App to Render

## Learning Goals

- Understand the React build process and how to serve a React app from Rails
- Understand challenges of client-side routing in a deployed application
- Deploy a Rails API with a React frontend to Render

## Introduction

Earlier in this section, we deployed a small Rails API application to Render to
learn how the deployment process works in general, and what steps are required
to take the code from our machine and get it to run on a server.

In this lesson, we'll be tackling a more complex app with a modern React-Rails
stack, and explore some of the challenges of getting these two apps to run
together on a single server.

## Setup

To follow along with this lesson, we have a pre-built React-Rails application
that you'll be deploying to Render. To start, head to this link, and **fork**
and **clone** the repository there:

- [https://github.com/learn-co-curriculum/phase-4-deploying-demo-app-render](https://github.com/learn-co-curriculum/phase-4-deploying-demo-app-render)

After downloading the code, set up the repository locally:

```console
$ bundle install
$ rails db:create db:migrate
$ npm install --prefix client
```

This application has a Rails API with session-based authentication, a React
frontend using React Router for client-side routing, and PostgreSQL for the
database.

Spend some time familiarizing yourself with the code for the demo app before
proceeding. We'll be walking through its setup and why certain choices were made
through the course of this lesson.

## React Production Build

One of the great features that Create React App provides to developers is the
ability to build different versions of a React application for different
environments.

When working in the **development** environment, a typical workflow for adding
new features to a React application is something like this:

- Run `npm start` to run a development server
- Make changes to the app by editing the files
- View those changes in the browser

To enable this _excellent_ developer experience, Create React App uses
[webpack](https://webpack.js.org/) under the hood to create a development server
with hot module reloading, so any changes to the files in our application will
be instantly visible to us in the browser. It also has a lot of other nice
features in development mode, like showing us good error and warning messages
via the console.

Create React App is _also_ capable of building an entirely different version of
our application for **production**, also thanks to webpack. The end goal of our
application is to get it into the hands of our users via our website. For our
app to run in production, we have a different set of needs:

- **Build** the static HTML, JavaScript and CSS files needed to run our app in
  the browser, keeping them as small as possible
- **Serve** the application's files from a server hosted online, rather than a
  local webpack development server
- Don't show any error messages/warnings that are meant for developers rather
  than our website's users

### Building a Static React App

When developing the frontend of a site using Create React App, our ultimate goal
is to create a **static site** consisting of pre-built HTML, JavaScript, and CSS
files, which can be served by Rails when a user makes a request to the server to
view our frontend. To demonstrate this process of **building** the production
version of our React app and **serving** it from the Rails app, follow these
steps.

**1.** Build the production version of our React app:

```console
$ npm run build --prefix client
```

This command will generate a bundled and minified version of our React app in
the `client/build` folder.

Check out the files in that directory, and in particular the JavaScript files.
You'll notice they have very little resemblance to the files in your `src`
directory! This is because of that **bundling** and **minification** process:
taking the source code you wrote, along with any external JavaScript libraries
your code depends on, and squishing it as small as possible.

**2.** Move our static frontend files to the `/public directory`:

```console
$ mv client/build/* public
```

This command will move all of the files and folders that are inside the
`client/build` directory into to the `public` directory. The `public` directory
is used by Rails to serve **static** assets, so when we run the Rails server, it
will be able to display the files from our production version of the React
application. When a user visits `http://localhost:3000`, Rails will return the
`index.html` file from this directory.

**3.** Run the Rails server:

```console
$ rails s
```

Visit [http://localhost:3000](http://localhost:3000) in the browser. You should
see the production version of the React application!

Explore the React app in the browser using the React dev tools. What differences
do you see between this version of the app and what you're used to when running
in development mode?

Now you've seen how to build a production version of the React application
locally, and some of the differences between this version and the development
version you're more familiar with.

There is one other issue with our React application to dive into before we
deploy it: how can we deal with client-side routing?

### Configuring Rails for Client-Side Routing

In our React application, we're using React Router to handle [client-side
routing][]. Client-side routing means that a user should be able to navigate to
the React application, load all the HTML/CSS/JavaScript code just **once**, and
then click through links in our site to navigate to different pages without
making another request to the server for a new HTML document.

We have two client-side routes defined:

```jsx
// client/src/components/App.js
<Switch>
  <Route path="/new">
    <NewRecipe user={user} />
  </Route>
  <Route path="/">
    <RecipeList />
  </Route>
</Switch>
```

When we run the app using `npm start` and webpack is handling the React server,
it can handle these client-side routing requests just fine! **However**, when
we're running React within the Rails application, we also have routes defined
for our Rails API, and Rails will be responsible for all the routing logic in
our application. So let's think about what will happen from the point of view of
**Rails** when a user makes a request to these routes.

- `GET /`: Rails will respond with the `public/index.html` file
- `GET /new`: Rails will look for a `GET /new` route in the `config/routes.rb`
  file. If we don't have this route defined, it will return a 404 error.

Any other client-side routes we define in React will have the same issue as
`/new`: since Rails is handling the routing logic, it will look for routes
defined in the `config/routes.rb` file to determine how to handle all requests.

We can solve this problem by setting up a **custom route** in our Rails
application, and handle any requests that come through that **aren't** requests
for our API routes by returning the `public/index.html` file instead.

Here's how it works:

```rb
# config/routes.rb
Rails.application.routes.draw do
  namespace :api do
    resources :recipes, only: [:index, :create]
    post "/signup", to: "users#create"
    get "/me", to: "users#show"
    post "/login", to: "sessions#create"
    delete "/logout", to: "sessions#destroy"
  end

  get "*path", to: "fallback#index", constraints: ->(req) { !req.xhr? && req.format.html? }
end
```

All the routes for our API are defined **first** in the `routes.rb` file. We use
the [namespacing][] to differentiate the API requests from other requests.

The last method in the `routes.rb` file handles all other `GET` requests by
sending them to a special `FallbackController` with an `index` action:

```rb
# app/controllers/fallback_controller.rb
class FallbackController < ActionController::Base
  def index
    render file: 'public/index.html'
  end
end
```

This action has just one job: to render the HTML file for our React application!

> **Note**: It's important that this `FallbackController` inherits from
> `ActionController::Base` instead of `ApplicationController`, which is what
> your API controllers inherit from. Why? The `ApplicationController` class in a
> Rails API inherits from the [`ActionController::API`
> class][actioncontroller api], which doesn't include the methods for rendering
> HTML. For our other controllers, this isn't a problem, since they only need to
> render JSON responses. But for the `FallbackController`, we need the ability
> to render an HTML file for our React application.

[actioncontroller api]:
  https://api.rubyonrails.org/classes/ActionController/API.html

Experiment with the code above. Run `rails s` to run the application. Try
commenting out the last line of the `routes.rb` file, and visit
[http://localhost:3000/new](http://localhost:3000/new). You should see a 404
page. Comment that line back in, and make the same request. Success!

Now that you've seen how to create a production version of our React app
locally, and tackled some thorny client-side routing issues, let's talk about
how to deploy the application to Render.

## Render Build Process

Think about the steps to build our React application locally. What did we have
to do to build the React application in such a way that it could be served by
our Rails application? Well, we had to:

- Run `npm install --prefix client` to install any dependencies
- Use `npm run build --prefix client` to create the production app
- Move the code from the `client/build` folder to the `public` folder
- Run `rails s`

We would also need to repeat these steps any time we made any changes to the
React code, i.e., to anything in the `client` folder. Ideally, we'd like to be
able to **automate** those steps when we deploy this app to Render, so we can
just push up new versions of our code to GitHub and deploy them like we were
able to do in the earlier lesson.

Luckily, we already have something that will help us do that! Recall from the
earlier lesson that one of the deployment steps was to create a build script in
the `bin` directory. If you look in the `bin` directory of our demo app, you'll
see a `recipes-build.sh` file that contains the following:

```bash
#!/usr/bin/env bash
# exit on error
set -o errexit

bundle install
bundle exec rake db:migrate
```

This file runs the commands to build the Rails API portion of our app when
changes are pushed to GitHub. All we need to do is add the corresponding
commands for the front end:

```bash
#!/usr/bin/env bash
# exit on error
set -o errexit

# Add build commands for front end
rm -rf public
npm install --prefix client && npm run build --prefix client
cp -a client/build/. public/

bundle install
bundle exec rake db:migrate
```

The code we added does the following whenever a build is launched:

- Removes the `public` folder that contains the current front end production
  build.
- Installs the dependencies for the app.
- Runs the production build.
- Recreates the `public` directory and copies the new production build files
  into it.

Finally, we need to run the following command in the terminal to make sure the
script is executable:

```sh
$ chmod a+x bin/recipes-build.sh
```

With this build script, any time we make a change to our app — in either the
front end or the back end — all we need to do is push the changes to GitHub and
relaunch the build from the Render dashboard!

## Creating the Master Key File

You may recall that, when we set up our Rails API for deployment earlier in this
section, we had to make edits to several files. Those edits have all been made
to the demo app's files, so you won't need to do those steps again. However,
there is one thing missing.

Recall that when we created the Web Service on Render, we added an environment
variable called `RAILS_MASTER_KEY` and pasted in the value that's in the
`config/master.key` file. If you look in the `config` folder for the demo app,
you'll see that file isn't there.

When you create a Rails app from scratch, the `master.key` file is automatically
created. However, this file contains secure information so it should not be
pushed to GitHub. If you look in the `.gitignore` file, you'll see it listed
there. As a result, when you fork and clone a repo from GitHub (as you did with
the demo app), the `master.key` file will not be present. So we need to create
it.

To do that, first delete the `config/credentials.yml.enc` file, which holds the
encrypted version of the key. Then run the following command in the terminal:

```sh
$ EDITOR="code --wait" bin/rails credentials:edit
```

**Note**: if you use a different text editor than VS Code, you will need to
replace `code` with the appropriate command.

The command above will open a file in VS Code and "wait" for you to close it
before completing the process of creating the credential files. You can edit the
secret key base in the file if you choose, but we'll leave it as is, so go ahead
and close the file. The command in the terminal will now complete, and you
should see both the `credentials.yml.enc` and `master.key` files in the `config`
folder.

See the Rails Guide on [Environmental Security][rails security] if you'd like
more information about these files.

## Deploy to Render

We're now ready to create a new database and Web Service and deploy the app to
Render. If you need a refresher on any of the steps below, look back at the
earlier lesson.

1. Commit and push your latest changes to GitHub.
2. Go to the Render dashboard, click on your PostgreSQL instance, scroll down to
   the "Connection" section, and copy the "PSQL Command."
3. Run the command in the terminal to open the PSQL terminal, then use the SQL
   command `CREATE DATABASE database_name;` to create a new database for the app
   (e.g., `recipe_app_db`). Exit PSQL with the `\q` command.
4. Return to the Render dashboard, click the "New +" button, and select "Web
   Service."
5. Select the GitHub repo you want to deploy then, on the next page: a) give the
   app a name, and b) set the Environment to Ruby.
6. Scroll down and enter the build command (`./bin/recipes-build.sh`) and start
   command (`bundle exec puma -C config/puma.rb`)
7. Scroll down and click "Advanced" then "Add Environment Variable." Add the
   `DATABASE_URL` environment variable. Go to the database page on the Render
   dashboard, scroll down to the "Connection" section, copy the "Internal
   Database URL" and paste it in as the value; be sure to change the database
   name at the end of the URL to `recipe_app_db` (or whatever you named it).
8. Add the `RAILS_MASTER_KEY` environment variable; enter the key contained in
   the `master.key` file we created above as the value.
9. Click the "Create Web Service" button at the bottom of the page.

## Making an Update

Try making a minor change in the app's front end. Commit and push the changes,
then launch a new build from the Render dashboard. Once the deploy is complete,
refresh the page and verify that our updated build script has deployed the
change.

## Conclusion

Creating a website out of multiple applications working together adds a
significant amount of complexity when it comes time to deploy our application.
The upside to this approach is we get to leverage the strengths of each of the
tools we're using: React for a speedy, responsive user interface, and Rails for
a robust, well-designed backend to communicate with the database.

By spending some time up front to understand and automate parts of the
deployment process, we can make future deployments simpler.

For your future projects using a React frontend and Rails API backend, we'll
provide a template project to use so you don't have to worry about configuring
the tricky parts of the deployment process yourself. However, it's helpful to
have an understanding of this configuration should you wish to customize it or
troubleshoot issues related to deployments in the future.

## Check For Understanding

Before you move on, make sure you can answer the following questions:

1. Why does deploying the production version of our Rails/React app lead to
   routing problems? What can we do to fix the issue?

## Resources

- [Render Rails-React Setup](https://blog.Render.com/a-rock-solid-modern-web-stack)
- [Demo App](https://github.com/learn-co-curriculum/phase-4-deploying-demo-app-render)

[namespacing]:
  https://guides.rubyonrails.org/routing.html#controller-namespaces-and-routing
[rails security]:
  https://guides.rubyonrails.org/security.html#environmental-security
[client-side routing]:
  https://render.com/docs/deploy-create-react-app#using-client-side-routing
