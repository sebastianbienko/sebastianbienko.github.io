---
layout: post
title: Customizing the Sitecore JSS CLI
subtitle: Adapting the Sitecore JSS CLI to a custom angular project structure (Sitecore 9.3)
gh-repo: sebastianbienko/cypressforsitecore
gh-badge: [star, fork, follow]
tags: [Sitecore JSS, Angular]
comments: true
---

I just started to work with a client on a Sitecore JSS project where one of our goals is to have a scalable Angular frontend application. They are already using [Nx (extensible dev tools for Monorepos)](https://nx.dev/angular) in a couple of other projects and therefore it made sense to use the same approach now. We also decided to start in the code-first workflow, so we needed to make sure that the manifest generation and other scripts are still running.

Nx gives you, alongside with a lot of other features, a specific folder structure at hand to work with. While the JSS CLI provides a couple of parameters to modify it's execution, I didn't find a way to use those alone to make it work with our application structure. Therefore, we decided to modify the CLI instead. For those of you who find themselves in a similar situation I will outline briefly how this can be easily done.

### What we want to end up with

Here is the the folder structure outlined in which we would like the CLI to work *(for the sake of this blog it is slightly simplified)*:

```
root/
├── apps/
│   ├── our-jss-app/
|	|	├── data					
|	|	|	├── component-content
|	|	|	├── content
|	|	|	├── dictionary
|	|	|	├── media
|	|	|	├── routes
|	|	├── scripts
|	|	|	├── manifest.ts
|	|	├── sitecore
│   │   |	├── config
│   │   |	├── definitions
│   │   |	├── manifest
│   |	├── src
│   │   |	├── app
|	|	├── sitecore.config.json
├── dist
├── libs/
│   ├── shared
│   │   ├── our-components
│   │   |	├── src
│   │   |	|	├── components
│   │   |	|	|	├── example
│   │   |	|	|	|	├── example.component.html
│   │   |	|	|	|	├── example.component.scss
│   │   |	|	|	|	├── example.component.spec.ts
│   │   |	|	|	|	├── example.component.ts
│   │   |	|	|	|	├── example.sitecore.ts
├── tools/
│   ├── schematics
|	|	├── jss-component
|	|	|	├── template-files
|	|	|	├── index.ts
|	|	|	├── schema.json
├── package.json
```

In case you opened a JSS solution before you will notice that the 'data' and 'sitecore' folders are still fully intact, just some levels further away from the solutions root folder. Another difference is that we placed our components in a shared library. Each component folder now also hold the component definition for the manifest.

### Scaffolding

Adapting the scaffolding is very straight forward. Because the JSS CLI uses [Angular schematics](https://angular.io/guide/schematics) internally we can simply use the templates and modify them according to our needs. The templates can be found here in the respective 'component-files' or 'manifest-files' folder: https://github.com/Sitecore/jss/tree/dev/packages/sitecore-jss-angular-schematics/src/jss-component

Simply take the templates and place them in the desired folder structure inside your schematic.

Almost all that is left for you to do is to write the script and schema to handle the file creation. I ended up with a very simple solution:

```typescript
// index.ts

import { chain, template, Rule, Tree, SchematicContext, apply, url, pathTemplate, mergeWith } from '@angular-devkit/schematics';
import { strings } from '@angular-devkit/core';

export default function (options: any): Rule {
  return chain([
    createFiles(options, "./template-files")
  ])
}

function createFiles (options: any, templatesPath: string): Rule{
  return (tree: Tree, context: SchematicContext) => {
    const sourceTemplates = url(templatesPath);

    const sourceParametrizedTemplates = apply(sourceTemplates, [
      pathTemplate({
        name: options.name,
        ...options.app,
        ...strings
      }),
      template({
        name: options.name,
        ...options.app,
        ...strings
      })
    ]);

    return mergeWith(sourceParametrizedTemplates)(tree, context);
  }
}
```

```json
// schema.json

{
  "$schema": "http://json-schema.org/schema",
  "id": "jss-component",
  "type": "object",
  "properties": {
    "name": {
      "type": "string",
      "description": "Name of the component.",
      "x-prompt": {
        "message": "Name of the component:",
        "type": "string"
      }
    },
    "app": {
      "type": "object",
      "x-prompt": {
        "message": "Select app:",
        "type": "list",
        "items": [
          {
            "label": "our-jss-app",
            "value": {
              "appId": "our-jss-app",
              "selector": "ofja"
            }
          }
        ]
      }
    }
  },
  "required": ["name","app"]
}
```

Of course you can also allow additional parameters to enable scaffolding for different variation like done in the [original JSS scaffolding script](https://github.com/Sitecore/jss/blob/dev/packages/sitecore-jss-angular-schematics/src/jss-component/index.ts), but since we now have full control about the generation there is not much need for it.

Be aware that for the above to work you will have to modify the templates slightly because they would usually expect additional parameters to be passed.

### Manifest generation

For the manifest generation to work we will once again turn to the JSS GitHub project where the manifest.ts script can be found: https://github.com/Sitecore/jss/blob/dev/packages/sitecore-jss-cli/src/scripts/manifest.ts We simply copy the file to the scripts folder of our app, where we can start to modify it. 

One reason why we have to modify the JSS CLI scripts is because they attempt to resolve the package JSON. The package.json is expected to be in the same directory in which context we run the command: `resolve('./package.json', { basedir: process.cwd() }, (error, packageJson) => {` In our case we like to run the scripts in a context where we don't have a package.json file. Instead we created a 'sitecore.config.json' file where the JSS settings are now placed separately. In our modified script we are reading this file instead.

Here is what we ended up with:

```typescript
// manifest.ts

import { clean } from '@sitecore-jss/sitecore-jss-dev-tools';
import { generateToFile } from '@sitecore-jss/sitecore-jss-manifest';
import chalk from 'chalk';
import { existsSync } from 'fs';
import readlineSync from 'readline-sync';

const path = require('path')

export const command = 'manifest';

export const describe =
  // tslint:disable-next-line:max-line-length
  'Generates a JSS manifest file which defines app assets to import into Sitecore. Nothing is deployed or added to a deployment package; this just collects assets. See `jss package`, which takes the manifest and turns it into a deployable package. `jss manifest --help` for options.';

export const builder = {
  appName: {
    requiresArg: false,
    type: 'string',
    describe: 'The name of the app. Defaults to the package.json config value.',
  },
  manifestSourceFiles: {
    requiresArgs: false,
    describe: 'The files or file patterns to parse to generate the manifest.',
    type: 'array',
  },
  require: {
    requiresArgs: false,
    type: 'string',
    describe:
      // tslint:disable-next-line:max-line-length
      'A JS module to require before processing the manifest. This may initialize a custom compiler (Babel, TypeScript), perform init tasks, etc.',
    default: './sitecore/definitions/config.js',
  },
  manifestOutputPath: {
    requiresArgs: false,
    type: 'string',
    describe: 'The path of the file to which manifest output will be written.',
  },
  includeContent: {
    requiresArgs: false,
    type: 'boolean',
    describe: 'Includes content and media items in the manifest output.',
    default: false,
    alias: 'c',
  },
  includeDictionary: {
    requiresArgs: false,
    type: 'boolean',
    describe: 'Includes dictionary items in the manifest output.',
    default: false,
    alias: 'd',
  },
  language: {
    requiresArgs: false,
    type: 'string',
    describe:
      'Defines the language the manifest represents. Defaults to the language config in the package.json.',
    alias: 'l',
  },
  rootPlaceholders: {
    requiresArgs: false,
    type: 'array',
    describe:
      // tslint:disable-next-line:max-line-length
      'Sets the root placeholder name(s) for the app. If set, overrides root placeholders set in the package.json',
    alias: 'p',
  },
  wipe: {
    requiresArgs: false,
    type: 'boolean',
    describe:
      // tslint:disable-next-line:max-line-length
      'Causes the JSS import to run as a wipe + recreate of any existing app items. Pass --unattendedWipe in addition to bypass interactive confirmation for CI scenarios.',
    alias: 'w',
    default: false,
  },
  unattendedWipe: {
    requiresArgs: false,
    hidden: true,
    type: 'boolean',
  },
  pipelinePatchFiles: {
    requiresArgs: false,
    type: 'array',
    describe: 'List of files or file patterns from which to load pipeline config patch files.',
    default: ['./sitecore/pipelines/**/*.patch.js', './sitecore/pipelines/**/*.patch.ts'],
  },
  debug: {
    requiresArgs: false,
    type: 'boolean',
    describe: 'If true, emits additional diagnostic information',
    default: false,
  },
  allowConflictingPlaceholderNames: {
    requiresArgs: false,
    type: 'boolean',
    describe: 'Enables using placeholder names that conflict with Sitecore or SXA',
    default: false,
    alias: 'a',
  },
};

export async function handler(argv: any) {
  const packageJson = require('../sitecore.config.json');

  let language = argv.language;
  if (!language && packageJson && packageJson.language) {
    language = packageJson.language;
  }
  if (!language) {
    // tslint:disable-next-line:no-string-throw
    throw 'Language was not defined as a parameter or in the package.json { config: { language: "en" } }';
  }

  let appName = argv.appName;
  if (!appName && packageJson && packageJson.appName) {
    appName = packageJson.appName;
  }
  if (!appName) {
    // tslint:disable-next-line:no-string-throw
    throw '--appName was not defined as a parameter or in the package.json { config: { appName: "myJssAppName" } }';
  }

  let rootPlaceholders = argv.rootPlaceholders;
  if (
    !rootPlaceholders &&
    packageJson &&
    packageJson.rootPlaceholders
  ) {
    rootPlaceholders = packageJson.rootPlaceholders;
  }
  if (!rootPlaceholders) {
    // tslint:disable-next-line:no-string-throw
    throw '--rootPlaceholders was not defined as a parameter or in the package.json { config: { rootPlaceholders: ["ph-name"] } }';
  }

  if (argv.wipe && !argv.unattendedWipe) {
    console.warn(chalk.yellow('Are you sure you want to wipe any existing app from Sitecore?'));
    if (
      !readlineSync.keyInYN(chalk.yellow('This will delete any content changes made in Sitecore'))
    ) {
      process.exit(1);
    }
  }

  const manifestSourceFiles = argv.manifestSourceFiles ?? packageJson.manifestSourceFiles;
  const manifestOutputPath = argv.manifestOutputPath ?? packageJson.manifestOutputPath;

  const generateArgs = {
    fileGlobs: manifestSourceFiles,
    requireArg: argv.require,
    appName,
    excludeItems: !argv.includeContent,
    excludeMedia: !argv.includeContent,
    excludeDictionary: !argv.includeDictionary,
    outputPath: `${manifestOutputPath}/sitecore-import.json`,
    language,
    pipelinePatchFileGlobs: argv.pipelinePatchFiles,
    debug: argv.debug,
    rootPlaceholders,
    wipe: argv.wipe,
    skipPlaceholderBlacklist: argv.allowConflictingPlaceholderNames,
  };

  console.log(`JSS is creating a manifest for ${appName} to ${manifestOutputPath}...`);

  if (existsSync(manifestOutputPath)) {
    clean({ path: manifestOutputPath });
  }

  const assetsPath = path.join(manifestOutputPath, 'assets');
  if (existsSync(assetsPath)) {
    clean({ path: assetsPath });
  }

  return generateToFile(generateArgs).catch((err) => {
    console.error('Error generating manifest', err);
    process.exit(1);
  });
}

const argv = require('yargs').options(builder).argv;

handler(argv);
```

If you compare this code to the original file you will see that no big changes are needed. We can proceed with all the other scripts which we want to use in the same manner.

Lastly we add the commands to our package.json and are good to go:

```json
{
  "scripts": {
    "jss:scaffold": "nx workspace-schematic jss-component",
    "jss:manifest": "cd apps/our-jss-app && ts-node -P scripts/tsconfig.json scripts/manifest.ts",
  }
}
```

