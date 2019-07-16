# Server-side rendering in ASP.NET Core and Angular

- Create a new ASP.NET Core project (Angular template)
- Open the csproj file and set BuildServerSideRenderer=true
- Remove the ClientApp folder
- `ng new ClientApp --style=scss --routing`
- `ng add @nguniversal/express-engine --clientProject ClientApp`
- `npm install --save aspnet-prerendering`
- SPA prerendering middleware:
```
spa.UseSpaPrerendering(options => {
  options.BootModulePath = $"{spa.Options.SourcePath}/dist/server/main.js";
  options.BootModuleBuilder = env.IsDevelopment()
    ? new AngularCliBuilder(npmScript: "build:ssr")
    : null;
  options.ExcludeUrls = new[] { "/sockjs-node" };
});
```
- Set build-type of all tsconfig files to **Content**
- Modify tsconfig.server.json:
  - compilerOptions:module = "commonjs"
  - compilerOptions:types = [ "node" ]
- main.ts:
```
import { enableProdMode } from '@angular/core';
import { platformBrowserDynamic } from '@angular/platform-browser-dynamic';

import { AppModule } from './app/app.module';
import { environment } from './environments/environment';

export function getBaseUrl() {
  return document.getElementsByTagName('base')[0].href;
}

const providers = [
  { provide: 'BASE_URL', useFactory: getBaseUrl, deps: [] }
];

if (environment.production) {
  enableProdMode();
}

document.addEventListener('DOMContentLoaded', () => {
  platformBrowserDynamic(providers).bootstrapModule(AppModule)
  .catch(err => console.error(err));
});
```
- main.server.ts
```
import 'zone.js/dist/zone-node';
import 'reflect-metadata';
import { renderModule, renderModuleFactory } from '@angular/platform-server';
import { APP_BASE_HREF } from '@angular/common';
import { enableProdMode } from '@angular/core';
import { provideModuleMap } from '@nguniversal/module-map-ngfactory-loader';
import { createServerRenderer } from 'aspnet-prerendering';
export { AppServerModule } from './app/app.server.module';

enableProdMode();

export default createServerRenderer(params => {
  const { AppServerModule, AppServerModuleNgFactory, LAZY_MODULE_MAP } = (module as any).exports;

  const options = {
    document: params.data.originalHtml,
    url: params.url,
    extraProviders: [
      provideModuleMap(LAZY_MODULE_MAP),
      { provide: APP_BASE_HREF, useValue: params.baseUrl },
      { provide: 'BASE_URL', useValue: params.origin + params.baseUrl }
    ]
  };

  // Bypass ssr api call cert warnings in development
  process.env.NODE_TLS_REJECT_UNAUTHORIZED = "0";

  const renderPromise = AppServerModuleNgFactory
    ? /* AoT */ renderModuleFactory(AppServerModuleNgFactory, options)
    : /* dev */ renderModule(AppServerModule, options);

  return renderPromise.then(html => ({ html }));
});
```
