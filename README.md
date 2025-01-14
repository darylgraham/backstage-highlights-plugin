# @rsc-labs/backstage-highlights-plugin

<img src='./docs/highlighter.png' width='100' height='150' alt='Highlights screenshot'>

Backstage Highlights Plugin is configurable and customizable plugin for viewing the most important information about your entity.

### Why?

We have a lot information from different plugins and also in Overview tab, but sometimes:

- we want to see some short summary from couple of plugins
- we do not want to jump to every card to get such information

The "Highlights" shall provide you possibility to create such small, useful view.

# Getting started

If you haven't already, check out the [Backstage docs](https://backstage.io/docs/getting-started/) and create a Backstage application with

```
npx @backstage/create-app
```

Then, you will need to install and configure the highlights plugins for the frontend and the backend.

# Frontend plugin

Install:

```bash
cd packages/app
yarn add @rsc-labs/backstage-highlights-plugin
```

### Card:

Add the card to `packages/app/src/components/catalog/EntityPage.tsx`:

```jsx
// import:
import { isGitlabHighlightsAvailable, EntityHighlightsCard } from '@rsc-labs/backstage-highlights-plugin';

// use it in entity view
const overviewContent = (
  <Grid container
  ...
    <Grid item md={12} xs={12}>
      <EntitySwitch>
          <EntitySwitch.Case if={isGitlabHighlightsAvailable}>
            <EntityHighlightsCard />
          </EntitySwitch.Case>
        </EntitySwitch>
    </Grid>
  </Grid>
)
```

For the best UX we <b> strongly recommend </b> to use as much horizontal space as possible. Thanks to that you will have your highlights on top of your page as a bar.

<img src='./docs/built_in_example.PNG' alt='Built-in example'>

Of course, you can also make it smaller and near the other card.

### Built-in fields

At this moment, "highlights plugin" comes with built-in support of basic information about Git. As you can see in above picture, we support following fields:

- latest tag
- number of branches
- latest commit
- date of latest commit
- author of latest commit
- clone button

You can click at the field and get more information. For example, when you click on latest tag you will get longer history:

<img src='./docs/commit_table.PNG' alt='Built-in example'>

Other fields can have similar functionality, but it depends on the provider (Github API provides more information)

At this moment built-in fields supports Github and Gitlab (see: Configuration of Backend).

## Frontend configuration

By default, you can use EntityHighlights without any parameter - it gives you above built-in fields.
However, you may want change a behaviour or implement your custom fields.
Below you can find an interface:

```typescript
/** @public */
export interface EntityHighlightsProps {
  fields?: EHighlightFields[];
  customFields?: HighlightCustomField[];
}
```

1. fields - this parameter describes what built-in you would like to see and in what order
2. customFields - this parameter can let you define your own field. Every custom field contains:
   - fieldLabel - it is a title of the field (you can see it in built-in fields). It is optional parameter as your field can be also without a title (example: Clone button in built-in fields)
   - field - it is simple React component

# Backend plugin

Install:

```bash
cd packages/backend
yarn add @rsc-labs/backstage-highlights-plugin-backend
```

Create a file `packages/backend/src/plugins/highlights.ts`:

```typescript
import { createRouter } from '@rsc-labs/backstage-highlights-plugin-backend';
import { Router } from 'express';
import { PluginEnvironment } from '../types';

export default async function createPlugin(
  env: PluginEnvironment,
): Promise<Router> {
  return await createRouter({
    discovery: env.discovery,
    tokenManager: env.tokenManager,
    logger: env.logger,
    config: env.config,
  });
}
```

Add the plugin to `packages/backend/src/index.ts`:

```typescript
// import:
import highlights from './plugins/highlights';
...

async function main() {
  ...
  // add env
  const highlightsEnv = useHotMemoize(module, () => createEnv('highlights'));
  ...
  // add to router
  apiRouter.use('/highlights', await highlights(highlightsEnv));
  ...
}
```

## Catalog-info.yaml

Backend plugin supports two providers - Github and Gitlab. They are providing information for built-in fields mentioned in Frontend plugin.
Plugin uses following annotations from catalog-info.yaml:

```yaml
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: example-website
  annotations:
    github.com/project-slug: rsc-labs/backstage-changelog-plugin
    gitlab.com/project-slug: owner/project
    gitlab.com/instance: gitlab.instance.com
```

Both <b>/project-slug</b> are supported (so your component can be in github or gitlab). In theory case if you have both annotations, github takes precedence.

<b>gitlab.com/instance</b> annotation is used for [Multiple Gitlab instances](#multiple-gitlab-instances)

## App-config

To have properly working Github or Gitlab, you need also provide information about token and potentially about base url.
You have two options how to provide it

1. Custom highlights configuration
2. Integration configuration

Below you can find implemented both options:

```yaml
highlights:
  gitlab:
    token: ${GITLAB_TOKEN}
    apiBaseUrl: https://gitlab.com/api/v4
  github:
    token: ${GITHUB_TOKEN}

integrations:
  gitlab:
    - token: ${GITLAB_TOKEN}
  github:
    - token: ${GITHUB_TOKEN}
```

If provided, "highlights" configuration takes precendece over "integrations".
<b>Note:</b> "highlights" configuration requires providing "apiBaseUrl", while it is not needed in "integrations" (if you are using default one)

### Multiple Gitlab instances

We start supporting also multiple Gitlab instances for both highlights and integrations in app-config. Below you can find instruction and example of configuration.

1. If you would like to use "highlights" configuration, then you need to change it to a list instead of one object. Additionally, we need to be able to figure out which instance is used for which entity, so <b>host</b> annotation is needed (see example below). For backward compatibility, we still support single object.
2. Similarly with <b>integrations</b>. We already support a list, but to use multiple Gitlab instances, you need to also provide <b>host</b> and <b>apiBaseUrl</b> (see example below)

Example of both options in one configuration:

```yaml
highlights:
  gitlab:
    - host: gitlab.com
      token: ${GITLAB_TOKEN}
      apiBaseUrl: https://gitlab.com/api/v4
    - host: gitlab1.com
      token: ${GITLAB2_TOKEN}
      apiBaseUrl: https://gitlab1.com/api/v4
  github:
    token: ${GITHUB_TOKEN}

integrations:
  gitlab:
    - host: gitlab.com
      token: ${GITLAB_TOKEN}
    - host: gitlab1.com
      token: ${GITLAB2_TOKEN}
      apiBaseUrl: https://gitlab1.com/api/v4
  github:
    - host: github.com
      token: ${GITHUB_TOKEN}
```

At the same time, your entity shall have proper annotations, for example:

```yaml
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: example-website
  annotations:
    github.com/project-slug: rsc-labs/backstage-changelog-plugin
    gitlab.com/project-slug: owner/project
    gitlab.com/instance: gitlab1.com
```

Taking above example - component <b>example-website</b> will use <b>gitlab1.com</b>, which maps to values:

- host: gitlab1.com
- token: ${GITLAB2_TOKEN}
- apiBaseUrl: https://gitlab1.com/api/v4

Please remember that "highlights" configuration (if present) takes precendence over "integrations".

## TODO

[ ] Unit tests

[ ] More fields to support

## Contribution

Contributions are welcome and they are greatly appreciated!

## License

Licensed under the Mozilla Public License, Version 2.0: https://www.mozilla.org/en-US/MPL/2.0/

---

© 2023 RSC https://rsoftcon.com/
