# Steps to reproduce

- Use: npm 7+ (tested on Node.js 16)
- Run: `npm run reset` (or `rm -rf node_modules package-lock.json`)
- Run: `npm i --before=2021-10-09T20:46:00.000Z`
- Run: `npm run lint` - linter should pass
- Run: `npm run reset` (or `rm -rf node_modules package-lock.json`)
- Run: `npm i --before=2021-10-09T20:47:00.000Z`
- Run: `npm run lint` - linter will fail like so:

```
Oops! Something went wrong! :(

ESLint: 7.32.0

ESLint couldn't find the plugin "eslint-plugin-filenames".

(The package "eslint-plugin-filenames" was not found when loaded as a Node module from the directory "/home/runner/work/inventory/inventory".)

It's likely that the plugin isn't installed correctly. Try reinstalling by running the following:

    npm install eslint-plugin-filenames@latest --save-dev

The plugin "eslint-plugin-filenames" was referenced from the config file in "package.json » @architect/eslint-config".

If you still can't figure out the problem, please stop by https://eslint.org/chat/help to chat with the team.
```

# What's happening

- This project has two dependencies: `eslint@7` (which has many subdependencies) + `@architect/eslint-config`, which has 3 subdependencies:
  - `eslint-plugin-filenames`
  - `eslint-plugin-fp`
  - `eslint-plugin-import`
- When deps before `2021-10-09T20:46:00.000Z` are installed:
  - The 3 `eslint-plugin-$name` dependencies above are installed flatly into `node_modules/` (e.g. `node_modules/eslint-plugin/filenames`)
- When deps before `2021-10-09T20:47:00.000Z` are installed (one minute later):
  - The 2 of 3 plugins are installed into `node_modules/@architect/eslint-config/node_modules/` (specifically `eslint-plugin-filenames`, `eslint-plugin-fp`)
  - The 3rd plugin is still installed flatly into `node_modules/`
- eslint then runs, and cannot find `eslint-plugin-filenames` (nor `eslint-plugin-fp`, but it fails on the first plugin) because they are both installed in `node_modules/@architect/eslint-config/node_modules/`, away from the reach of Node's module resolution
- **Note:** `eslint@8` DOES NOT appear in this dependency tree anywhere, however it was published at `2021-10-09T20:46:13.874Z`
  - This implies that Arborist / npm is reifying module locations based on the existence of a package that is not even in use in this project

## Affected module `package.json`

Relevant fields from the two dependencies placed in the wrong file heirarchy:

```json
{
  "name": "eslint-plugin-filenames",
  "version": "1.3.2",
  "dependencies": {
    "lodash.camelcase": "4.3.0",
    "lodash.kebabcase": "4.1.1",
    "lodash.snakecase": "4.1.1",
    "lodash.upperfirst": "4.3.1"
  },
  "peerDependencies": {
    "eslint": "*"
  },
  ...
}
```
```json
{
  "name": "eslint-plugin-fp",
  "version": "2.3.0",
  "dependencies": {
    "create-eslint-index": "^1.0.0",
    "eslint-ast-utils": "^1.0.0",
    "lodash": "^4.13.1",
    "req-all": "^0.1.0"
  },
  "peerDependencies": {
    "eslint": ">=3"
  },
  ...
}
```
