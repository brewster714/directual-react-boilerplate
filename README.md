This project was bootstrapped with [Create React App](https://github.com/facebook/create-react-app).

# React boilerplate for integrating with Directual as a backend

This is an example React project integrated with [Directual](https://www.directual.com/). 
- Here is a [demo](http://react.directual.app/)
- You can also download the [snapshot](https://api.alfa.directual.com/fileUploaded/React-boilerplate/c52983be-d61f-48d9-b5ec-5e5aba1eeb4a.json
) of Directual app.

## Step-by-step instruction for creating your app from scratch
### 1. Create a react bootstrap project and bind required dependencies

`npm install -g create-react-app`

go to home directory and run command:

`npm init react-app directual-example`

go to the your new project folder:

`cd directual-example`

and install necessary libs: react-router-dom, directual-api:

`npm install react-router-dom --save `

`npm install directual-api --save `

`npm install http-proxy-middleware --save `

### 2. Run you app

The command: `npm run start` runs the app in the development mode.<br />
Open [http://localhost:3000](http://localhost:3000) to view it in the browser.

The page will reload automatically if you make edits.<br />
You will also see lint errors in the console.


### 3. Create middleware proxy to directual.api, for resolving problem linked with CORS

create file `.env` in root directory with you APP_ID,
example:
`.env`
`APP_ID=050e77bb-b0e6-4685-8712-a85774fad272`

You can find APP_ID in Api -> API keys section on [Directual](https://my.directual.com), 


create `src/setupProxy.js` file in you src directory, and insert this text

```javascript
const { createProxyMiddleware } = require('http-proxy-middleware');
const API_HOST = 'https://api.alfa.directual.com/'
// !Important, set APP_ID in , env file or set you APP ID here
const APP_ID = process.env.APP_ID

module.exports = function(app) {
  app.use(
    '/good/api',
    createProxyMiddleware({
      target: API_HOST,
      changeOrigin: true,
      pathRewrite(pathReq, req) {
        const pathname = pathReq.split('?')[0];
        let url = `${pathname}?appID=${APP_ID}`;
        url = Object
          .entries(req.query)
          .reduce(
            (newUrl, [key, value]) => `${newUrl}&${key}=${encodeURI(value)}`,
            url,
          );
        return url;
      }
    })
  );
};
```

### 4. Create a simple site structure
create `pages` folder and insert 3 files

`src/pages/PublicPage.js`
```javascript
import React from 'react'

export default function PublicPage () {
  return (
    <div>
      <h2>Public Page</h2>
    </div>
  )
}
```
`src/pages/PrivatePage.js`
```javascript
import React from 'react'

export default function PrivatePage () {
  return (
    <div>
      <h2>Private Page</h2>
    </div>
  )
}
```
`src/pages/AdminPage.js`
```javascript
import React from 'react'

export default function AdminPage () {
  return (
    <div>
      <h2>Admin Page</h2>
    </div>
  )
}
```

`src/pages/LoginPage.js`
```javascript
import { useHistory, useLocation } from 'react-router-dom'
import React, { useState, useEffect } from 'react';
import { ProvideAuth, useAuth } from "./../auth";

export default function LoginPage () {
  let history = useHistory()
  let location = useLocation()

  const [username, setUsername] = useState('');
  const [password, setPassword] = useState('');
  const [error, setError] = useState('');

  let { from } = location.state || { from: { pathname: '/' } }
  const auth = useAuth();

  useEffect(() => {
    if(auth.isAutorised()){
      history.replace(from);
    }
  })
  let login = () => {
    auth.login(username, password).then(()=>{
      history.replace(from);
    }).catch(e=>{
      setError("You login or password incorrect")
    })
  }

  return (
    <div>
      <p>You must log in to view the the page {from.pathname}</p>
      <input onChange={(e)=> {
        setUsername(e.target.value)
      }}/><br />
      <input onChange={(e)=> {
        setPassword(e.target.value)
      }}/>
      {error}
      <button onClick={login}>Log in</button>
    </div>
  )
}
```


create `src/LogInLogOutButton.js` file and 
`src/LogInLogOutButton.js`
```javascript

import React from 'react'
import { useAuth } from './auth'
import {
    Link, useHistory
} from 'react-router-dom'

export function LogInLogOutButton() {
    let history = useHistory()
    const auth = useAuth();
    return (
        <React.Fragment>
            {auth.user ? (
                <button
                    onClick={() => {
                        auth.signout(() => history.push('/'))
                    }}
                >
                    Log out
                </button>
            ) : (
                    <Link to="/login">
                        <button>Log in</button></Link>
                )}
        </React.Fragment>)
}
```


create `src/auth.js` file and 

`src/auth.js`

```javascript
import React, { useState, useEffect, useContext, createContext } from "react";
import Directual from 'directual-api';

const api = new Directual({apiHost: '/'});

export const authContext = createContext();

export function ProvideAuth({ children }) {
  const auth = useProvideAuth();
  return <authContext.Provider value={auth}>{children}</authContext.Provider>;
}

export const useAuth = () => {
  return useContext(authContext);
};

// Provider hook that creates auth object and handles state
function useProvideAuth() {
  const [user, setUser] = useState(null);
  const [sessionID, setSessionID] = useState(null);
  const [role, setRole] = useState(null);

  const login = (username, password) => {
    return api.auth.login(username, password).then(res=>{
      setUser(res.username)
      setSessionID(res.sessionID)
      setRole(res.role)
      window.localStorage.setItem('sid', res.sessionID)
    })
  };

  const signout = (cb) => {
    return api.auth.logout('').then(res=>{
      setUser(null)
      setSessionID(null)
      window.localStorage.setItem('sid', null)
      cb()
    })
  };

  const isAutorised = () => {
    return !!user
  }

  const hasRole = (roleCheck) => {
    return role === roleCheck
  }

  useEffect(() => {
    let sid = window.localStorage.getItem('sid') || ''
    api.auth.isAuthorize(sid, (status, token)=>{
      if(status === true){
        setUser(token.username)
        setSessionID(token.sessionID)
        setRole(token.role)
      }
    })
  }, []);

  return {
    user,
    sessionID,
    login,
    isAutorised,
    signout,
    hasRole
  };
}
```


change you `src/App.js`

`src/App.js`

```javascript
import React, { useState, useEffect } from 'react';
import {
  BrowserRouter as Router,
  Switch,
  Route,
  Link,
  Redirect,
  useHistory,
  useLocation
} from 'react-router-dom'
import './App.css'
import LoginPage from './pages/LoginPage'
import PrivatePage from './pages/PrivatePage'
import PublicPage from './pages/PublicPage'
import AdminPage from './pages/AdminPage'
import { ProvideAuth, useAuth } from "./auth";
import { LogInLogOutButton } from {./LogInLogOutButton}


export function PrivateRoute({ children, hasRole, ...rest }) {
    const auth = useAuth();
    return (
        <Route
            {...rest}
            render={({ location }) =>
                auth.isAutorised() && auth.hasRole(hasRole) ? (
                    children
                ) : auth.isAutorised() && !auth.hasRole(hasRole) ? <AccessDenied /> : (
                    <Redirect
                        to={{
                            pathname: '/login',
                            state: { from: location }
                        }}
                    />
                )
            }
        />
    )
}

//example how use standart React components:

class MainMenu extends React.Component{
  render() {
    const authContext = this.context;
    return  <div> 
      user is auth {authContext.isAutorised() ? 'true' : 'false'} 
      user has role admin : {authContext.hasRole('admin') ? 'true' : 'false'}
    </div>
  }
}
MainMenu.contextType = authContext


function App () {
  useEffect( ()=>{

  })
  return ( <ProvideAuth>
    <Router>
      <div>
        <ul>
          <li>
            <Link to="/">Public Page</Link>
          </li>
          <li>
            <Link to="/private">Private Page</Link>
          </li>
          <li>
            <Link to="/admin">Admin Page</Link>
          </li>
        </ul>

        <hr />
        <MainMenu />

        <Switch>
          <Route path="/login">
            <LoginPage />
          </Route>
          <Route exact path="/">
            <PublicPage />
          </Route>
          <PrivateRoute path="/private" >
            <PrivatePage />
          </PrivateRoute>
          <PrivateRoute path="/admin" hasRole={'admin'} >
            <AdminPage />
          </PrivateRoute>
        </Switch>
      </div>
    </Router>
    </ProvideAuth>
  )
}

export default App

```

### 5. Connect directual-api

Open [ApiEndpoints](https://directual.gitbook.io/directual-documentation/api-integrations/api-endpoints-security-layer) section on Directual, 
choose any endpoint
and press 

`Endpoint respond preview`

after copy code in section `How it use? step 3` to useEffect function.


Example result page: `src/pages/DashboardPage.js`

```javascript
import React, { useEffect, useState } from 'react'
import Directual from 'directual-api';
import { useAuth } from '../auth'
const api = new Directual({apiHost: '/'});

export default function DashBoardPage () {
  const [payload, setPayload] = useState([]);
  const [loading, setLoading] = useState(true);
  const auth = useAuth();

  useEffect( ()=>{
    api
      .structure('test')
      .getData('test', {sessionID:auth.sessionID})
      .then((response) => {
        setPayload(response.payload)
        setLoading(false)
      })
      .catch((e) => {
        setLoading(false)
        if(!e.response){
          //check you API endpoint, you must enable CORS header in settings
        }
        if(e.response && e.response.status === 403){
          //todo: api endpoint required authorisation
        }
      })
  }, [])
  return (
    <div>
      <h2>Dashboard</h2>
      {loading && <span>loading</span>}
      {payload.map((data)=>{
        return <div>{JSON.stringify(data)}</div>
      })}
    </div>
  )
}
```

### 6. Build and pack you product in docker container

#### Create Docker file in root directory
Create `Dockerfile` with following body:

```
FROM node:14.0.0-buster-slim
ENV BUILD_PATH /usr/src/app
ENV COMMIT_SHORT COMMIT_PLACEHOLDER
ENV DOCKER_TAG TAG_PLACEHOLDER

RUN mkdir -p $BUILD_PATH
WORKDIR $BUILD_PATH
COPY . $BUILD_PATH
RUN npm install
RUN npm run build

FROM node:14.0.0-buster-slim

ENV SERVER_FOLDER /opt/app/
RUN mkdir -p $SERVER_FOLDER
WORKDIR $SERVER_FOLDER

ARG NODE_ENV=production
ENV NODE_ENV $NODE_ENV

RUN npm install --global express && npm link express && \
npm install --global http-proxy-middleware && npm link http-proxy-middleware

ENV NODE_SERVER_PORT=${NODE_SERVER_PORT:-8080}

COPY server/ $SERVER_FOLDER/server
RUN mkdir -p $SERVER_FOLDER/src
COPY src/setupProxy.js $SERVER_FOLDER/src/setupProxy.js
COPY --from=0 /usr/src/app/build $SERVER_FOLDER/build

EXPOSE $NODE_SERVER_PORT $NODE_INSPECT_PORT

ENTRYPOINT exec node ./server/server.js

```


Create proxy server in server directory:

`server/server.js`


```
const path = require('path');
const express = require('express');
const proxy = require('../src/setupProxy')
const SERVER_PORT = 8080;

const app = express();
app.use(express.static(path.join(__dirname, '../build')));
proxy(app)

const server = app.listen(SERVER_PORT, () => {
  console.log(`start webserver: http://localhost:${SERVER_PORT} \nAPP ID: ${process.env.APP_ID} `);
});

module.exports = app;
```

#### Build docker image
run command: `docker build -t final_image_name .`

#### Run you image

`docker run -d -p 8080:8080 -e APP_ID='__YOU_APP_ID__' final_image_name`

