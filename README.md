# CRA to Nx Migration

This is a repo showing the result of a CRA app that has been migrated to Nx.

The example `webapp` uses a shared buildable library that can be used by other apps in the workspace.


## Trying it out

```
npx nx lint webapp
npx nx test webapp

# Build webapp and all dependencies 
npx nx build webapp --withDeps

# Start dev server
npx nx serve webapp
```
