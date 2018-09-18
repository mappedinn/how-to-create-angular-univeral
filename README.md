# How to convert an Angular project to an Angular Universal project?

Reference url: [https://angular.io/guide/universal](https://angular.io/guide/universal)

## 1. Start up

```sh
npm i
ng serve
```

Just keep in mind that a regular angular application is downloaded by the client. Then, the client browser will be running it to render the page.
This assumes that:

* the client is able to run javascript
* the client has enough resources to run javascript code (old mobile devices will struggle to run the application).
* Crawlers (like Google, Bing, Twitter, etc...) are expected to run the javascript code to render the pages and index them.

In order to avoid these issues, it would be preferable to convert the angular projet to an angular universal project.

## 2. Conversion to Angular Universal

### 2.1. Install the tools

To get started, install these packages.

* `@angular/platform-server` - Universal server-side components.
* `@nguniversal/module-map-ngfactory-loader` - For handling lazy-loading in the context of a server-render.
* `@nguniversal/express-engine` - An express engine for Universal applications.
* `ts-loader` - To transpile the server application

```sh
npm install --save @angular/platform-server @nguniversal/module-map-ngfactory-loader ts-loader @nguniversal/express-engine
```


### 2.2. Modify the client app

A Universal app can act as a dynamic, content-rich "splash screen" that engages the user. It gives the appearance of a near-instant application.

Meanwhile, the browser downloads the client app scripts in background. Once loaded, Angular transitions from the static server-rendered page to the dynamically rendered views of the interactive client app.

You must make a few changes to your application code to support both server-side rendering and the transition to the client app.

**a. src/app/app.module.ts**

Open file src/app/app.module.ts and find the BrowserModule import in the NgModule metadata. Replace that import with this one:

```ts
BrowserModule.withServerTransition({ appId: 'tour-of-heroes' }),
```

Angular adds the `appId` value (which can be any string) to the style-names of the server-rendered pages, so that they can be identified and removed when the client app starts.

You can get runtime information about the current platform and the appId by injection.

```ts
// src/app/app.module.ts
// ...
import { PLATFORM_ID, APP_ID, Inject } from '@angular/core';
import { isPlatformBrowser } from '@angular/common';
// ...
  constructor(
    @Inject(PLATFORM_ID) private platformId: Object,
    @Inject(APP_ID) private appId: string) {
    const platform = isPlatformBrowser(platformId) ?
      'in the browser' : 'on the server';
    console.log(`Running ${platform} with appId=${appId}`);
  }
// ...
```

**b. Build Destination**

A Universal app is distributed in two parts: 

* the server-side code that serves up the initial application
* the client-side code that's loaded in dynamically.

The Angular CLI outputs the client-side code in the dist directory by default, so you modify the outputPath for the build target in the `angular.json` to keep the **client-side build outputs** separate from the **server-side code**. The client-side build output will be served by the Express server.

```json
...
"build": {
  "builder": "@angular-devkit/build-angular:browser",
  "options": {
    "outputPath": "dist/browser", // modified to "dist/browser"
    ...
  }
}
...
```

**c. Absolute HTTP URLs**

The tutorial's `HeroService` and `HeroSearchService` delegate to the Angular HttpClient module to fetch application data. These services send requests to **relative URLs** such as `api/heroes`.

In a Universal app, HTTP URLs must be **absolute**, for example, https://my-server.com/api/heroes even when the Universal web server is capable of handling those requests.

You'll have to change the services to make requests with absolute URLs when running on the server and with relative URLs when running in the browser.

One solution is to provide the server's runtime origin under the Angular `APP_BASE_HREF` token, inject it into the service, and prepend the origin to the request URL.

Start by changing the `HeroService` constructor to take a second origin parameter that is optionally injected via the `APP_BASE_HREF` token.


```ts
// src/app/hero.service.ts (constructor with optional origin)
// ...
import { Injectable, Optional, Inject } from '@angular/core';
import {APP_BASE_HREF} from '@angular/common';
// ...
constructor(
  private http: HttpClient,
  private messageService: MessageService,
  @Optional() @Inject(APP_BASE_HREF) origin: string) {
    this.heroesUrl = `${origin}${this.heroesUrl}`;
  }
// ...
```

### 2.3. Create the server app

To run an Angular Universal application, you'll need a server that accepts client requests and returns rendered pages.

**a. App server module**

The app server module class (conventionally named `AppServerModule`) is an Angular module that wraps the application's root module (`AppModule`) so that Universal can mediate between your application and the server. `AppServerModule` also tells Angular how to bootstrap your application when running as a Universal app.

Create an `app.server.module.ts` file in the `src/app/` directory with the following `AppServerModule` code:

```ts
import { NgModule } from '@angular/core';
import { ServerModule } from '@angular/platform-server';
import { ModuleMapLoaderModule } from '@nguniversal/module-map-ngfactory-loader';

import { AppModule } from './app.module';
import { AppComponent } from './app.component';

@NgModule({
  imports: [
    AppModule,
    ServerModule,
    ModuleMapLoaderModule
  ],
  providers: [
    // Add universal-only providers here
    // it my opinion, it should be: HeroService, MessageService
    // but may be it is already imported from AppModule
    // Needs checking
  ],
  bootstrap: [ AppComponent ],
})
export class AppServerModule {}
```
Notice that it imports first the client app's `AppModule`, the Angular Universal's `ServerModule` and the `ModuleMapLoaderModule`.

The `ModuleMapLoaderModule` is a server-side module that allows lazy-loading of routes.

This is also the place to register providers that are specific to running your app under Universal.

**b. App server entry point**

The Angular CLI uses the `AppServerModule` to build the server-side bundle.

Create a `main.server.ts` file in the `src/` directory that exports the `AppServerModule`:

```ts
// src/main.server.ts
export { AppServerModule } from './app/app.server.module';
```

The `main.server.ts` will be referenced later to add a server target to the Angular CLI configuration.

**c. Universal web server**

A Universal web server responds to application page requests with static HTML rendered by the Universal template engine.

It receives and responds to HTTP requests from clients (usually browsers). It serves static assets such as scripts, css, and images. It may respond to data requests, perhaps directly or as a proxy to a separate data server.

The sample web server for this guide is based on the popular Express framework.

## 2.4. Configure for Universal

**a. Universal TypeScript configuration**

Create a `tsconfig.server.json` file in the project root directory to configure:

* TypeScript 
* AOT compilation of the universal app.

```json
// src/tsconfig.server.json
{
    "extends": "../tsconfig.json",
    "compilerOptions": {
        "outDir": "../out-tsc/app",
        "baseUrl": "./",
        "module": "commonjs",
        "types": []
    },
    "exclude": [
        "test.ts",
        "**/*.spec.ts"
    ],
    "angularCompilerOptions": {
        "entryModule": "app/app.server.module#AppServerModule"
    }
}
```

This config extends from the root's `tsconfig.json` file. Certain settings are noteworthy for their differences.

* The module property must be commonjs which can be required into our server application.

* The angularCompilerOptions section guides the AOT compiler:

  - entryModule - the root module of the server application, expressed as `path/to/file#ClassName`

**b. Universal Webpack configuration**

Universal applications doesn't need any extra Webpack configuration, the CLI takes care of that for you, but since the server is a typescript application, you will use Webpack to transpile it.

Create a `webpack.server.config.js` file in the project root directory with the following code.

```js
// webpack.server.config.js
const path = require('path');
const webpack = require('webpack');

module.exports = {
  entry: { server: './server.ts' },
  resolve: { extensions: ['.js', '.ts'] },
  target: 'node',
  mode: 'none',
  // this makes sure we include node_modules and other 3rd party libraries
  externals: [/node_modules/],
  output: {
    path: path.join(__dirname, 'dist'),
    filename: '[name].js'
  },
  module: {
    rules: [{ test: /\.ts$/, loader: 'ts-loader' }]
  },
  plugins: [
    // Temporary Fix for issue: https://github.com/angular/angular/issues/11580
    // for 'WARNING Critical dependency: the request of a dependency is an expression'
    new webpack.ContextReplacementPlugin(
      /(.+)?angular(\\|\/)core(.+)?/,
      path.join(__dirname, 'src'), // location of your src
      {} // a map of your routes
    ),
    new webpack.ContextReplacementPlugin(
      /(.+)?express(\\|\/)(.+)?/,
      path.join(__dirname, 'src'),
      {}
    )
  ]
};
```

Webpack configuration is a rich topic beyond the scope of this guide.

## 2.5. Angular CLI configuration

The CLI provides builders for different types of targets. Commonly known targets such as `build` and `serve` already exist in the `angular.json` configuration. To target a server-side build, add a `server` target to the architect configuration object.

* The `outputPath` tells where the resulting build will be created.
* The `main` provides the `main` entry point to the previously created `main.server.ts`
* The `tsConfig` uses the `tsconfig.server.json` as configuration for the TypeScript and AOT compilation.

```json
"architect": {
  ...
  "server": {
    "builder": "@angular-devkit/build-angular:server",
    "options": {
      "outputPath": "dist/server",
      "main": "src/main.server.ts",
      "tsConfig": "src/tsconfig.server.json"
    }
  }
  ...
}
```

## 2.6. scripts

Now that you've created the TypeScript and Webpack config files and configured the Angular CLI, you can build and run the Universal application.

First add the `build` and `serve` commands to the scripts section of the `package.json`:

```json

```

