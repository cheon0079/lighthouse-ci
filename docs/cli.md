# CLI

## Overview

The CLI is the primary API you'll use when setting up Lighthouse CI. To install:

```bash
npm install -g @lhci/cli
```

## Commands

All commands support configuration via a JSON file passed in via `--config=./path/to/`. Any argument on the CLI can also be passed in via environment variable. For example, `--config=foo` can be replaced with `LH_CONFIG=foo`. Learn more about [configuration](./configuration.md).

* [`healthcheck`](#healthcheck)
* [`autorun`](#autorun)
* [`collect`](#collect)
* [`upload`](#upload)
* [`assert`](#assert)
* [`open`](#open)
* [`wizard`](#wizard)
* [`server`](#server)

### Global Options

```txt
  --help             Show help                                         [boolean]
  --version          Show version number                               [boolean]
  --no-lighthouserc  Disables automatic usage of a .lighthouserc file. [boolean]
  --config           Path to JSON config file.
```

---------------------

### `healthcheck`

Runs diagnostics to ensure a valid configuration, useful when setting up Lighthouse CI for the first time to test your configuration.

```bash
Options:
  --fatal            Exit with a non-zero status code when a component fails the status check.
                                                                                           [boolean]
  --checks           The list of opt-in checks to include in the fatality check.             [array]
```

**Examples**

```bash
lhci healthcheck --fatal --checks=githubToken
```

---

### `autorun`

Automatically run, with sensible defaults, [`lhci collect`](#collect), [`lhci assert`](#assert) and [`lhci upload`](#upload), depending on the options. Options for individual commands can be set by prefixing the option with the command name.

**Examples**

```bash
lhci autorun --collect.numberOfRuns=5
lhci autorun --upload.target=temporary-public-storage
```

#### Starting a webserver

To allow autorun to automatically start your webserver, add an npm script named `serve:lhci`.

example `package.json` excerpt:

```json
  "scripts": {
    "serve:lhci": "NODE_ENV=production npm run server"
  }
```

Alternatively, you can supply the custom server initalization command from [`collect`](#collect) as a flag directly to `autorun`:

```bash
lhci autorun --collect.startServerCommand="rails server -e production"
```

---

### `collect`

Runs Lighthouse n times and stores the LHRs in a local `.lighthouseci/` folder.

```bash
Options:
  --method                                              [string] [choices: "node"] [default: "node"]
  --headful                  Run with a headful Chrome                                     [boolean]
  --additive                 Skips clearing of previous collect data                       [boolean]
  --url                      A URL to run Lighthouse on. Use this flag multiple times to evaluate
                             multiple URLs.
  --staticDistDir            The build directory where your HTML files to run Lighthouse on are
                             located.
  --isSinglePageApplication  If the application is created by Single Page Application, enable
                             redirect to index.html.
  --chromePath               The path to the Chrome or Chromium executable to use for collection.
             [default: "/Applications/Google Chrome Canary.app/Contents/MacOS/Google Chrome Canary"]
  --puppeteerScript          The path to a script that manipulates the browser with puppeteer before
                             running Lighthouse, used for auth.
  --puppeteerLaunchOptions   The object of puppeteer launch options
  --startServerCommand       The command to run to start the server.
  --startServerReadyPattern  String pattern to listen for started server.
                                                                  [string] [default: "listen|ready"]
  --startServerReadyTimeout  The number of milliseconds to wait for the server to start before
                             continuing                                    [number] [default: 10000]
  --settings                 The Lighthouse settings and flags to use when collecting
  --numberOfRuns, -n         The number of times to run Lighthouse.            [number] [default: 3]
```

**Examples**

```bash
# Basic usage
lhci collect --numberOfRuns=5 --url=https://example.com
# Have LHCI start a server before running
lhci collect --start-server-command="yarn serve" --url=http://localhost:8080/
# Have LHCI use the built-in server on a static directory
lhci collect --staticDistDir=./dist
# Have LHCI use the built-in server and audit a specific URL
lhci collect --staticDistDir=./public --url=http://localhost/products/pricing/
# Run on multiple URLs
lhci collect --url=https://example-1.com --url=https://example-2.com
# Have LHCI start a server and login with puppeteer before running
lhci collect --start-server-command="yarn serve" --url=http://localhost:8080/ --puppeteer-script=./path/to/login-with-puppeteer.js
```

#### Using Puppeteer Scripts

**NOTE:** In order to use puppeteer scripts, you need to install `puppeteer` yourself, i.e. `npm install --save puppeteer`. Any version you like that has a `.launch` method compatible with v1.x or v2.x and works with the Chrome version installed in your environment should work.

When running Lighthouse CI on a page with behind authentication, you'll need to authorize the browser that Lighthouse CI will be using. While there are [several](./configuration.md#page-behind-authentication) different [options](https://github.com/GoogleChrome/lighthouse/blob/v5.6.0/docs/authenticated-pages.md) to accomplish this, `--puppeteer-script` is one of the most flexible and convenient.

`--puppeteer-script` accepts a path to a JavaScript file that exports a function that will be invoked by Lighthouse CI before navigating, e.g. a script that will login to your site.

**Example `puppeteer-script.js`**

```js
/**
 * @param {puppeteer.Browser} browser
 * @param {{url: string, options: LHCI.CollectCommand.Options}} context
 */
module.exports = async (browser, context) => {
  const page = await browser.newPage();
  await page.goto('http://localhost:8080/login');
  await page.type('#username', 'admin');
  await page.type('#password', 'password');
  await page.click('[type="submit"]');
  await page.waitForNavigation();
};
```

Lighthouse CI will then use the browser that this script sets up when running Lighthouse. Note that if you store your credentials in `localStorage` or anything other than a cookie you might want to pair this option with `--settings.disableStorageReset` to force Lighthouse to keep the cache state.

For more information on how to use puppeteer, read up on [their API docs](https://github.com/puppeteer/puppeteer/blob/v2.0.0/docs/api.md#class-browser).

---

### `upload`

Saves the runs in the `.lighthouseci/` folder to desired target and sets a GitHub status check when token is available.

```bash
Options:
  --target                       The type of target to upload the data to. If set to anything other
                                 than "lhci", some of the options will not apply.
                            [string] [choices: "lhci", "temporary-public-storage"] [default: "lhci"]
  --token                        [lhci only] The Lighthouse CI server token for the project.[string]
  --ignoreDuplicateBuildFailure  [lhci only] Whether to ignore failures (still exit with code 0)
                                 caused by uploads of a duplicate build.                   [boolean]
  --githubToken                  The GitHub token to use to apply a status check.           [string]
  --githubApiHost                The GitHub host to use for the status check API request. Modify
                                 this when using on a GitHub Enterprise server.
                                                        [string] [default: "https://api.github.com"]
  --githubAppToken               The LHCI GitHub App token to use to apply a status check.  [string]
  --githubStatusContextSuffix    The suffix of the GitHub status check context label.       [string]
  --extraHeaders                 [lhci only] Extra headers to use when making API requests to the
                                 LHCI server.
  --basicAuth.username           [lhci only] The username to use on a server protected with HTTP
                                 Basic Authentication.                                      [string]
  --basicAuth.password           [lhci only] The password to use on a server protected with HTTP
                                 Basic Authentication.                                      [string]
  --serverBaseUrl                [lhci only] The base URL of the LHCI server where results will be
                                 saved.                          [default: "http://localhost:9001/"]
  --uploadUrlMap                 [temporary-public-storage only] Whether to post links to historical
                                 base results to storage or not. Defaults to true only on master
                                 branch.                                  [boolean] [default: false]
  --urlReplacementPatterns       [lhci only] sed-like replacement patterns to mask non-deterministic
                                 URL substrings.                                   [array] [default:
  ["s#:[0-9]{3,5}/#:PORT/#","s/[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}/UUID/ig"
                                                                                                  ]]
```

**Examples**

```bash
lhci upload --config=./lighthouserc.json
lhci upload --target=temporary-public-storage
lhci upload --serverBaseUrl=http://lhci.my-custom-domain.com/
lhci upload --extraHeaders.Authorization='Bearer X92sEo3n1J1F0k1E9'
lhci upload --basicAuth.username="customuser" --basicAuth.password="LighthouseRocks"
lhci upload --githubToken="$MY_ENTERPRISE_GITHUB_TOKEN" --githubApiHost="https://custom-github-server.example.com/api/v3"
```

#### Build Context

When uploading to the Lighthouse CI server, the CLI will attempt to automatically infer the build context such as git hash, author, message, ancestor, etc. In most cases, there's nothing you need to change about this, but if you're running without a git repo, in a Jenkins environment, a CI provider we haven't documented yet, or are just running into errors, you can control the build context yourself.

The following environment variables override the inferred build context settings.

| Name                                     | Example Format                                  |
| ---------------------------------------- | ----------------------------------------------- |
| `LHCI_BUILD_CONTEXT__GIT_REMOTE`         | `git@github.com:GoogleChrome/lighthouse-ci.git` |
| `LHCI_BUILD_CONTEXT__GITHUB_REPO_SLUG`   | `GoogleChrome/lighthouse-ci`                    |
| `LHCI_BUILD_CONTEXT__CURRENT_HASH`       | `e7f1b0fa3aebb6ef95e44c0d0b820433ffdd2e63`      |
| `LHCI_BUILD_CONTEXT__ANCESTOR_HASH`      | `78bafdcaf40e204c0d0b81b90bb76b9aa5834e11`      |
| `LHCI_BUILD_CONTEXT__COMMIT_TIME`        | `2019-11-29T16:43:39-05:00`                     |
| `LHCI_BUILD_CONTEXT__CURRENT_BRANCH`     | `dev_branch_1234`                               |
| `LHCI_BUILD_CONTEXT__COMMIT_MESSAGE`     | `Daily run of Lighthouse`                       |
| `LHCI_BUILD_CONTEXT__AUTHOR`             | `Patrick Hulce <patrick.hulce@example.com>`     |
| `LHCI_BUILD_CONTEXT__AVATAR_URL`         | https://example.com/patrickhulce.jpg            |
| `LHCI_BUILD_CONTEXT__EXTERNAL_BUILD_URL` | https://my-jenkins.example.com/jobs/123         |

---

### `assert`

Asserts the conditions in the Lighthouse CI config and exits with the appropriate status code if there were any failures. See the [assertion docs](./assertions.md) for more.

```bash
Options:
  --preset                   The assertions preset to extend
                          [choices: "lighthouse:all", "lighthouse:recommended", "lighthouse:no-pwa"]
  --assertions               The assertions to use.
  --budgetsFile              The path (relative to cwd) to a budgets.json file.
  --includePassedAssertions  Whether to include the results of passed assertions in the output.
                                                                                           [boolean]
```

**Examples**

```bash
lhci assert --config=./lighthouserc.json
lhci assert --budgetsFile=./budgets.json
lhci assert --preset=lighthouse:recommended --assertions.speed-index=off
lhci assert --preset=lighthouse:recommended --includePassedAssertions
```

---

### `open`

Open a local lighthouse report that has been created using `collect`.

```bash
lhci open
```

---

### `wizard`

Runs an interactive step-by-step wizard to either A) create a new project on the LHCI server or B) reset a project's admin token.

```bash
lhci wizard
```

---

### `server`

Starts the LHCI server. This command is unique in that it is likely run on infrastructure rather than in your build process. Learn more about the [LHCI server](./getting-started.md#the-lighthouse-ci-server) and [how to setup your personal LHCI server](./recipes/heroku-server/README.md) accessible over the internet.

```bash

Run Lighthouse CI server

Options:
  --logLevel                            [string] [choices: "silent", "verbose"] [default: "verbose"]
  --port, -p                                                                [number] [default: 9001]
  --storage.storageMethod                      [string] [choices: "sql", "spanner"] [default: "sql"]
  --storage.sqlDialect         [string] [choices: "sqlite", "postgres", "mysql"] [default: "sqlite"]
  --storage.sqlDatabasePath              The path to a SQLite database on disk.
  --storage.sqlConnectionUrl             The connection url to a postgres or mysql database.
  --storage.sqlConnectionSsl             Whether the SQL connection should force use of SSL
                                                                          [boolean] [default: false]
  --storage.sqlDangerouslyResetDatabase  Whether to force the database to the required schema.
                                         WARNING: THIS WILL DELETE ALL DATA
                                                                          [boolean] [default: false]
  --basicAuth.username                   The username to protect the server with HTTP Basic
                                         Authentication.                                    [string]
  --basicAuth.password                   The password to protect the server with HTTP Basic
                                         Authentication.                                    [string]
```

**Examples**

```bash
lhci server --storage.sqlDatabasePath=./lhci.db
lhci server --storage.sqlDialect=postgres --storage.sqlConnectionUrl="postgres://user@localhost/lighthouse_ci_db"
lhci server --storage.sqlDatabasePath=./lhci.db --basicAuth.username="customuser" --basicAuth.password="LighthouseRocks"
```

## Configuration

Configuring settings for Lighthouse CI is most easily done through a `lighthouserc.json` file. The file supports configuration for the four major commands, `collect`, `upload`, `assert`, and `server`. Learn more about [configuration](./configuration.md).

**Usage**

```bash
# Your config file is at `./lighthouserc.json` and will be picked up automatically
lhci <command>

# Specify a config file via command-line flag
lhci <command> --config=path/to/different/rc/file
```

**Example Project Config File**

```json
{
  "ci": {
    "collect": {
      "numberOfRuns": 5
    },
    "assert": {
      "preset": "lighthouse:recommended",
      "assertions": {
        "offscreen-images": "off",
        "uses-webp-images": "off"
      }
    },
    "upload": {
      "serverBaseUrl": "http://my-custom-lhci-server.example.com/"
    }
  }
}
```

**Example Server RC File**

```json
{
  "ci": {
    "server": {
      "port": 9009,
      "storage": {
        "sqlDatabasePath": "/home/lhci/lhci.db"
      }
    }
  }
}
```
