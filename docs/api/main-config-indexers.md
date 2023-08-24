---
title: 'indexers'
---

(⚠️ **Experimental**)

<div class="aside">

🧪 While this feature is experimental, it must be specified by the `experimental_indexers` property of [`StorybookConfig`](./main-config.md).

</div>

Parent: [main.js|ts configuration](./main-config.md)

Type: `(existingIndexers: Indexer[]) => Promise<Indexer[]>`

Indexers are responsible for building Storybook's index of stories—the list of all stories and a subset of their metadata like `id`, `title`, `tags`, and more. The index can be read at the `/index.json` route of your Storybook.

The indexers API is an advanced feature that allows you to customize Storybook's indexers, which dictate how Storybook indexes and parses files into story entries. This adds more flexibility to how you can write stories, including which language stories are defined in or where to get stories from.

They are defined as a function that returns the full list of indexers, including the existing ones. This allows you to add your own indexer to the list, or to replace an existing one:

<!-- prettier-ignore-start -->

<CodeSnippets
  paths={[
    'common/main-config-indexers.js.mdx',
    'common/main-config-indexers.ts.mdx',
  ]}
/>

<!-- prettier-ignore-end -->

For more information, see [Creating an indexer](#creating-an-indexer), below.

## `Indexer`

Type:

```ts
{
  test: RegExp;
  index: (fileName: string, options: IndexerOptions) => Promise<IndexInput[]>;
}
```

Specifies which files to index and how to index them as stories.

### `test`

(Required)

Type: `RegExp`

A regular expression run against filenames included in the [`stories`](./main-config-stories.md) configuration that should match all files to be handled by this indexer.

### `index`

(Required)

Type: `(fileName: string, options: IndexerOptions) => Promise<IndexInput[]>`

Function to index a single story file and return a list of entries.

#### `fileName`

Type: `string`

The name of the file to index.

#### `IndexerOptions`

Type:

```ts
{
  makeTitle: (userTitle?: string) => string;
}
```

Options for indexing the file.

##### `makeTitle`

Type: `(userTitle?: string) => string`

A function that takes a user-provided title and returns a title for the file. This is used to generate the title for the file in the sidebar.

#### `IndexInput`

Type:

```ts
{
  exportName: string;
  importPath: string;
  type: 'story';
  metaId?: string;
  name?: string;
  tags?: string[];
  title?: string;
  __id?: string;
};
```

An object representing a story to be added to the Storybook sidebar.

##### `exportName`

(Required)

Type: `string`

The name of the export to import.

##### `importPath`

(Required)

Type: `string`

The file to import from, e.g. the story file.

##### `type`

(Required)

Type: `'story'`

The type of entry.

##### `metaId`

Type: `string`

Default: Auto-generated from [`title`](#title)

Define the custom id for meta of the entry.

If specified, the export default (meta) in the CSF file _must_ have a matching `id` property, to be correctly matched.

##### `name`

Type: `string`

Default: Auto-generated from [`exportName`](#exportname)

The name of the entry.

##### `tags`

Type: `string[]`

Tags for filtering entries in Storybook and its tools.

##### `title`

Type: `string`

Default: Auto-generated from [`importPath`](#importpath)

The location in the sidebar.

##### `__id`

Type: `string`

Default: Auto-generated from [`title`](#title)/[`metaId`](#metaid) and [`exportName`](#exportname)

Define the custom id for the story of the entry.

If specified, the story in the CSF file _must_ have a matching `__id` property, to be correctly matched.

Only use this if you need to override the auto-generated id.

## Creating an indexer

All indexers must be defined in the function assigned to the `experimental_indexer` property of `StorybookConfig`. The full [Indexer API](#indexer) is detailed above.

If an indexer parses files that are not written in [CSF](./csf.md), then you also need to define a builder plugin that transpiles those files to regular CSF that Storybook can read in the browser.

<details>

<summary>Transpiling to CSF</summary>

Transpiling the custom source format to CSF is beyond the scope of this documentation. This transpilation is often done at the builder level ([Vite](../builders/vite.md) and/or [Webpack](../builders/webpack.md)), and we recommend using [unplugin](https://github.com/unjs/unplugin) to create plugins for multiple builders.

Consider an example where the source files provide an easy way to generate multiple stories, which are then transpiled into CSF. First, here's an example of a source file:

```ts
// ./Button.stories-generator.ts

import { variantsFromComponent, createStoryFromVariant } from '../utils';
import { Button } from './Button';

/**
 * returns raw strings representing stories via component props, eg.
 * 'export const PrimaryVariant = {
 *    args: {
 *      primary: true
 *    },
 *  };'
 */
export const generateStories = () => {
  const variants = variantsFromComponentProps(Button);
  return variants.map((variant) => createStoryFromVariant(variant));
};
```

The builder plugin would then:

1. Receive and read the source file
2. Import the exported `generateStories` function
3. Run the function to generate the stories
4. Write the stories to a CSF file

That CSF file would then be indexed by Storybook. It would something look like this:

```js
// virtual:./Button.stories-generated

import { Button } from './Button';

export default {
  component: Button,
};

export const PrimaryVariant = {
  args: {
    primary: true,
  },
};
```

</details>

### Examples

Some example usages of custom indexers include:

<details open>

<summary>Generating stories dynamically from fixture data or API endpoints</summary>

This indexer generates stories for components based on JSON fixture data. It looks for `jsonstories.js|jsx|ts|tsx` files in the project and then for each such file, looks for `.json` files in the same directory, each of which is added as a story.

<!-- TK: Make snippet -->

```ts
// .storybook/main.ts

import type { StorybookConfig } from '@storybook/your-framework';

import fs from 'fs/promises';
import path from 'path';

const config: StorybookConfig = {
  framework: '@storybook/your-framework',
  stories: [
    '../src/**/*.mdx',
    '../src/**/*.stories.@(js|jsx|mjs|ts|tsx)',
    // 👇 Make sure files to index are included in `stories`
    '../src/**/*.jsonstories.@(js|jsx|mjs|ts|tsx)',
  ],
  experimental_indexers: async (existingIndexers) => {
    const jsonFixtureIndexer = {
      test: /jsonstories\.[tj]sx?$/,
      index: async (fileName) => {
        // JSON files in the current directory
        const jsonFiles = (await fs.readdir(path.dirname(fileName)))
          .filter(f => f.endsWith('.json'))

        // Sidebar entries for each file
        return jsonFiles
          .map(jsonFile => ({
            type: 'story',
            importPath: fileName,
            exportName: f.split('.')[0]
          }));
      }
    return [...existingIndexers, jsonFixtureIndexer];
  }
};
```

</details>

<details>

<summary>Generating stories based on custom combinatorial logic</summary>

This is an indexer in a fictional addon that provides combinatorial testing for files that end with `.combos.js|jsx|ts|tsx`. It generates extra named exports (stories) of the form `Combo0 ... ComboN` for each combination of args.

<!-- TK: Make snippet -->

```ts
// addon-arg-combos/preset.ts

import fs from 'fs/promises';
import path from 'path';

import { parseCombos } from '../utils';

const CombosIndexer: Indexer = {
  test: /\.combos\.[tj]sx?$/,
  index: async (fileName) => {
    const code = await fs.readFile(fileName);
    const combos = parseCombos(code);

    // Sidebar entries for each file
    return combos.map((combo, idx) => ({
      type: 'story',
      importPath: fileName,
      exportName: `Combo${idx}`,
    }];
  }
}

export default {
  experimental_indexers: (existingIndexers) => [...existingIndexers, CombosIndexer],
};
```

</details>

<details>

<summary>Defining stories in template languages like Vue or Svelte native syntax instead of CSF</summary>

TK

</details>
