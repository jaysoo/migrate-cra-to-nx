# Migrating CRA to Nx

We can migrate an existing CRA app to Nx by customizing the CRA setup using [`react-app-rewired`](https://www.npmjs.com/package/react-app-rewired), and wrapping the commands using `@nrwl/workspace:run-commands` builder.

1. Create Nx workspace

   ```
   create-nx-workspace acme --appName=webapp --preset=react --style=css --nx-cloud
   ```

   **Note:** Replace `acme` with your organization's npm scope. This will be used when importing workspace projects. You can also replace `webapp` with a different name -- you can have more than one apps in an Nx workspace.

2. Add npm packages to our workspace to support CRA

   ```
   yarn add --dev react-scripts @testing-library/jest-dom eslint-config-react-app react-app-rewired
   ```

   **Note:** The `react-app-rewired` package allows us to customize the webpack config without ejecting.

3. Replace code generated by Nx with the CRA app

   ```
   rm -rf apps/webapp/* apps/webapp/{.babelrc,.browserslistrc}
   cp -r /path/to/cra-app/{README.md,package.json,tsconfig.json,src,public} apps/webapp
   ```

4. Add CRA commands

   ```
   nx g @nrwl/workspace:run-commands serve \
   --project webapp \
   --command "node ../../node_modules/.bin/react-app-rewired start" \
   --cwd "apps/webapp"

   nx g @nrwl/workspace:run-commands build \
   --project webapp \
   --command "node ../../node_modules/.bin/react-app-rewired build" \
   --cwd "apps/webapp" \
   --outputs "dist/apps/webapp"

   nx g @nrwl/workspace:run-commands lint \
   --project webapp \
   --command "npx eslint src/**/*.tsx src/**/*.ts" \
   --cwd "apps/webapp"

   nx g @nrwl/workspace:run-commands test \
   --project webapp \
   --command "node ../../node_modules/.bin/react-app-rewired test --watchAll=false" \
   --cwd "apps/webapp"
   ```

   **Note:** The `--cwd` and `--outputs` options are not yet supported by the schematic. This will be fixed in a future release. For now, add the entries manually as follows:

   ```
   "webapp": {
     "root": "apps/webapp",
     "sourceRoot": "apps/webapp/src",
     "projectType": "application",
     "schematics": {},
     "architect": {
       "serve": {
         "builder": "@nrwl/workspace:run-commands",
         "options": {
           "cwd": "apps/webapp",
           "command": "node ../../node_modules/.bin/react-app-rewired start"
         }
       },
       "build": {
         "builder": "@nrwl/workspace:run-commands",
         "options": {
           "cwd": "apps/webapp",
           "command": "node ../../node_modules/.bin/react-app-rewired build",
           "outputs": "dist/apps/webapp"
         }
       },
       "lint": {
         "builder": "@nrwl/workspace:run-commands",
         "options": {
           "cwd": "apps/webapp",
           "command": "npx eslint src/**/*.tsx src/**/*.ts"
         }
       },
       "test": {
         "builder": "@nrwl/workspace:run-commands",
         "options": {
           "cwd": "apps/webapp",
           "command": "node ../../node_modules/.bin/react-app-rewired test --watchAll=false"
         }
       }
     }
   ```

5. Create a `apps/webapp/config-overrides.js` file so we can customize webpack and jest options for our CRA app.

   ```
   const path = require('path');
   const TsConfigPathsPlugin = require('tsconfig-paths-webpack-plugin');
   const ModuleScopePlugin = require('react-dev-utils/ModuleScopePlugin');

   module.exports = {
    webpack: (config) => {
      // Remove guard against importing modules outside of `src`.
      // Needed for workspace projects.
      config.resolve.plugins = config.resolve.plugins.filter(
        (plugin) => !(plugin instanceof ModuleScopePlugin)
      );

      // Add support for importing workspace projects.
      config.resolve.plugins.push(
        new TsConfigPathsPlugin({
          configFile: path.resolve(__dirname, 'tsconfig.json'),
          extensions: ['.ts', '.tsx', '.js', '.jsx'],
          mainFields: ['module', 'main'],
        })
      );

      // Replace include option for babel loader with exclude
      // so babel will handle workspace projects as well.
      config.module.rules.forEach((r) => {
        if (r.oneOf) {
          const babelLoader = r.oneOf.find(
            (rr) => rr.loader.indexOf('babel-loader') !== -1
          );
          babelLoader.exclude = /node_modules/;
          delete babelLoader.include;
        }
      });

      return config;
    },
    paths: (paths) => {
      // Rewrite dist folder to where Nx expects it to be.
      paths.appBuild = path.resolve(__dirname, '../../dist/apps/webapp');
      return paths;
    },
    jest: (config) => {
      config.resolver = '@nrwl/jest/plugins/resolver';
      return config;
    },
   };

   ```

6. Extend CRA `tsconfig.json` from base (e.g. `apps/webapp/tsconfig.json`)

   ```
   {
     "extends": "../../tsconfig.base.json",
     ...
   }
   ```

7. Add `tsconfig.base.json` and `tsconfig.spec.json` files for jest and eslint

   ```
   echo '{ "extends": "./tsconfig.json" }' > apps/webapp/tsconfig.base.json
   echo '{ "extends": "./tsconfig.json" }' > apps/webapp/tsconfig.spec.json
   ```

8. Skip CRA preflight check since Nx manages the monorepo.

   ```
   echo "SKIP_PREFLIGHT_CHECK=true" > .env
   ```

9. Add all `node_modules` to `.gitignore`

   ```
   echo "node_modules" >> .gitignore
   ```

10. Try the commands

    ```
    nx serve webapp
    nx build webapp
    nx lint webapp
    nx test webapp
    ```

    **Note:** If you add and use a new workspace project, then you'll have to restart the dev server to pick up the new module.
