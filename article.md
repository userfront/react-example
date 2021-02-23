# React authentication, simplified

Authentication is one of those things that just always seems to take a lot more effort than we want it to. To set it up, you have to re-research topics you haven’t thought about since the last time you did authentication, and the fast-paced nature of the space means things have often changed in the meantime. New threats, new options, and new updates may have kept you guessing and digging through docs in your past projects.

In this article, we lay out a different approach to authentication (and access control, SSO, and more) in React applications. Rather than add a static library that you have to keep up to date or re-research each time you want to implement auth, we’ll use a service that stays up to date automatically and is a much simpler alternative to Auth0, Okta, and others.

## React authentication

We typically use a similar approach when writing authentication in React: our React app makes a request to our authentication server, which then returns an access token. That token is saved in the browser and can be used in subsequent requests to your server (or other servers, if needed). Whether writing standard email & password authentication or using magic links or single sign on (SSO) logins like Google, Azure, or Facebook, we want our React app to send an initial request to an authentication server and have that server handle all the complexity of generating a token.

So React’s responsibility in authentication is to:

1. Send the initial request to the authentication server
2. Receive and store the access token
3. Send the access token to your server with each request

## JWT access tokens

JSON Web Tokens (JWTs) are compact, URL-safe tokens that can be used for authentication and access control in React applications. Each JWT has a simple JSON-object as its “payload” and is signed such that your server can verify that the payload is authentic. An example JWT would look like:

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOjEsImF1dGhvcml6YXRpb24iOiJhZG1pbiJ9.f7iKN-xi24qrQ5NQtOe0jiriotT-rve3ru6sskbQXnA
```

The payload for this token is the middle section (separated by periods):

```
eyJ1c2VySWQiOjEsImF1dGhvcml6YXRpb24iOiJhZG1pbiJ9
```

The JWT payload can be decoded from base64 to yield the JSON object:

```js
JSON.parse(atob("eyJ1c2VySWQiOjEsImF1dGhvcml6YXRpb24iOiJhZG1pbiJ9"));

// =>
{
  “userId”: 1,
  “authorization”: “admin”
}
```

It’s important to note that this payload is readable by anyone with the JWT, including your React application or a third party.

Anyone that has the JWT can read its contents. However, only the authentication server can generate valid JWTs -- your React application, your application server, or a malicious third party cannot generate valid JWTs. So in addition to reading the JWT, your server also needs to verify the JWT as authentic by checking it against a public key. This allows your application server to verify incoming JWTs and reject any tokens that were not created by the authentication server or that have expired.

The flow for using a JWT in a React application looks like this:

1. Your React app requests a JWT whenever the user wants to sign on.
2. The authentication server generates a JWT using a private key and then sends the JWT back to your React app.
3. Your React app sends this JWT to your application server whenever your user needs to make a request.
4. Your application server verifies the JWT using a public key and then read the payload to determine which user is making the request.

Each of these steps is simple to write down, but each step has its own pitfalls when you actually want to implement it and keep it secure. Especially over time, as new threat vectors emerge and new platforms need to be patched or supported, the security overhead can add up quickly.

## Userfront removes auth complexity in React apps

Userfront makes it much easier to work with authentication to a React app and, perhaps most importantly, keeps the auth protocols updated automatically over time. Their underlying philosophy is that world-class auth should not take effort – it should be easy to start and maintain. Userfront has the bells and whistles of authentication, SSO, access control, and multi-tenancy, with a production-ready free tier for 10,000 monthly active users. For most modern React applications, it’s a no-brainer.

### Setting up authentication in React

Use your favorite boilerplate to set up your React application and get your build pipeline in order. In this article, we’ll use [Create React App](https://reactjs.org/docs/create-a-new-react-app.html), which does a lot of the setup work for us, and we’ll also add [React Router](https://reactrouter.com/web/guides/quick-start) for our client-side routing. Start by installing Create React App and React Router:

```js
npx create-react-app my-app
cd my-app
npm install react-router-dom --save
npm start
```

Now our React application is available at http://localhost:3000

![Create React App authentication](https://res.cloudinary.com/component/image/upload/v1612896738/permanent/create-react-app.gif)

Just like it says, we can now edit the `src/App.js` file to start working.

Replace the contents of `src/App.js` with the following, based on the React Router quickstart:

```jsx
// src/App.js

import React from "react";
import { BrowserRouter as Router, Switch, Route, Link } from "react-router-dom";

export default function App() {
  return (
    <Router>
      <div>
        <nav>
          <ul>
            <li>
              <Link to="/">Home</Link>
            </li>
            <li>
              <Link to="/login">Login</Link>
            </li>
            <li>
              <Link to="/reset">Reset</Link>
            </li>
            <li>
              <Link to="/dashboard">Dashboard</Link>
            </li>
          </ul>
        </nav>

        <Switch>
          <Route path="/login">
            <Login />
          </Route>
          <Route path="/reset">
            <PasswordReset />
          </Route>
          <Route path="/dashboard">
            <Dashboard />
          </Route>
          <Route path="/">
            <Home />
          </Route>
        </Switch>
      </div>
    </Router>
  );
}

function Home() {
  return <h2>Home</h2>;
}

function Login() {
  return <h2>Login</h2>;
}

function PasswordReset() {
  return <h2>Password Reset</h2>;
}

function Dashboard() {
  return <h2>Dashboard</h2>;
}
```

Now we have a very simple app with routing:

| Route        | Description                              |
| :----------- | :--------------------------------------- |
| `/`          | Home page                                |
| `/login`     | Login page                               |
| `/reset`     | Password reset page                      |
| `/dashboard` | User dashboard, for logged in users only |

![React Router authentication](https://res.cloudinary.com/component/image/upload/v1614094607/permanent/react-router-basic.gif)

This is all the structure we need to start adding authentication.

### Signup, login, and password reset with Userfront

First, create a Userfront account at https://userfront.com. This will give you a signup form, login form, and password reset form you can use for the next steps.

In the Toolkit section of your Userfront dashboard, you can find the instructions for installing your signup form:

![Userfront installation instructions](https://res.cloudinary.com/component/image/upload/v1614094834/permanent/instructions-react.png)

Follow the instructions by installing the Userfront react package with:

```js
npm install @userfront/react --save
npm start
```

Then, add the form to your home page by importing and initializing Userfront, then updating the `Home()` function to render the form.

```js
// src/App.js

import React from "react";
import { BrowserRouter as Router, Switch, Route, Link } from "react-router-dom";
import Userfront from "@userfront/react";

Userfront.init("demo1234");

const SignupForm = Userfront.build({
  toolId: "nkmbbm",
});

export default function App() {
  return (
    <Router>
      <div>
        <nav>
          <ul>
            <li>
              <Link to="/">Home</Link>
            </li>
            <li>
              <Link to="/login">Login</Link>
            </li>
            <li>
              <Link to="/reset">Reset</Link>
            </li>
            <li>
              <Link to="/dashboard">Dashboard</Link>
            </li>
          </ul>
        </nav>

        <Switch>
          <Route path="/login">
            <Login />
          </Route>
          <Route path="/reset">
            <PasswordReset />
          </Route>
          <Route path="/dashboard">
            <Dashboard />
          </Route>
          <Route path="/">
            <Home />
          </Route>
        </Switch>
      </div>
    </Router>
  );
}

function Home() {
  return (
    <div>
      <h2>Home</h2>
      <SignupForm />
    </div>
  );
}

function Login() {
  return <h2>Login</h2>;
}

function PasswordReset() {
  return <h2>Password Reset</h2>;
}

function Dashboard() {
  return <h2>Dashboard</h2>;
}
```

Now the home page has your signup form. Try signing up a user:

![React signup form](https://res.cloudinary.com/component/image/upload/v1614095453/permanent/react-router-signup.png)

The form is in "Test mode" by default, which will create user records in a test environment you can view separately in your Userfront dashboard:

![Userfront test mode](https://res.cloudinary.com/component/image/upload/v1612980797/permanent/create-react-app-2.png)

Continue by adding your login and password reset forms in the same way that you added your signup form:

```js
// src/App.js

import React from "react";
import { BrowserRouter as Router, Switch, Route, Link } from "react-router-dom";
import Userfront from "@userfront/react";

Userfront.init("demo1234");

const SignupForm = Userfront.build({
  toolId: "nkmbbm",
});
const LoginForm = Userfront.build({
  toolId: "alnkkd",
});
const PasswordResetForm = Userfront.build({
  toolId: "dkbmmo",
});

export default function App() {
  return (
    <Router>
      <div>
        <nav>
          <ul>
            <li>
              <Link to="/">Home</Link>
            </li>
            <li>
              <Link to="/login">Login</Link>
            </li>
            <li>
              <Link to="/reset">Reset</Link>
            </li>
            <li>
              <Link to="/dashboard">Dashboard</Link>
            </li>
          </ul>
        </nav>

        <Switch>
          <Route path="/login">
            <Login />
          </Route>
          <Route path="/reset">
            <PasswordReset />
          </Route>
          <Route path="/dashboard">
            <Dashboard />
          </Route>
          <Route path="/">
            <Home />
          </Route>
        </Switch>
      </div>
    </Router>
  );
}

function Home() {
  return (
    <div>
      <h2>Home</h2>
      <SignupForm />
    </div>
  );
}

function Login() {
  return (
    <div>
      <h2>Login</h2>
      <LoginForm />
    </div>
  );
}

function PasswordReset() {
  return (
    <div>
      <h2>Password Reset</h2>
      <PasswordResetForm />
    </div>
  );
}

function Dashboard() {
  return <h2>Dashboard</h2>;
}
```

At this point, your signup, login, and password reset should all be functional.

Your users can sign up, log in, and reset their password.

![React signup, login, password reset](https://res.cloudinary.com/component/image/upload/v1614095875/permanent/react-router-3.gif)
