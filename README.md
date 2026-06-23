# dataverse-auth

Cross-platform pure NodeJS On-behalf-of authentication against Microsoft dataverse Pro. Stores the token for use with NodeJS applications such as [dataverseify](https://github.com/scottdurow/dataverse-ify)

> **Note:** This is a public fork of [scottdurow/dataverse-auth](https://github.com/scottdurow/dataverse-auth) with updated dependencies and a modernized toolchain. It is **not published to npm**, so install it from source (see below) rather than with `npx`.

> **Note:** Version 2 of dataverse-auth is not compatible with version Version 1 of dataverse-ify and dataverse-gen.

## Installation

This fork is installed from the Git repository, not from npm. You need [Node.js](https://nodejs.org/) 18+ and `git`.

```bash
# 1. Clone the repository
git clone https://github.com/smanessis/dataverse-auth.git

cd dataverse-auth

# 2. Install dependencies (this also downloads the Electron binary)
npm install

# 3. Build the TypeScript into dist/
npm run build

# 4. Install the CLI globally so the `dataverse-auth` command is available everywhere
npm install -g .
```

After this, the `dataverse-auth` command is available in any terminal. To update later, `git pull` then re-run `npm install && npm run build && npm install -g .`.

## Usage

`~$ dataverse-auth [environment]`\
E.g.\
`~$ dataverse-auth contosoorg.crm.dynamics.com`

### Optional - specify tenant url

If you want to specify the tenant Url rather than have it looked up automatically:\
`~$ dataverse-auth [tenant] [environment]`\
E.g.\
`~$ dataverse-auth contoso.onmicrosoft.com contosoorg.crm.dynamics.com`\
For more information see the [dataverse-ify project](https://github.com/scottdurow/dataverse-ify)

## Other commands

- `dataverse-auth list` : Lists the currently authenticated environments
- `dataverse-auth [environmentUrl] test-connection` : Tests a previously authenticated environment
- `dataverse-auth [environmentUrl] remove` : Removes the stored token for an authenticated environment
- `dataverse-auth [environmentUrl] device-code` : Adds an authentication profile using the device-code flow. Use this if you are having trouble authenticating using the interactive prompt.

## Tested on

- Windows
  - ✔ 11

## MacOS, Apple Silicon usage

Version 2 is required to work on MacOS Apple Silicon. After installing from source (see [Installation](#installation)), run `dataverse-auth <org url>`.

### Build & Test

`dataverse-auth` uses electron which uses node-gyp. You will need to install Python and Visual Studio C++ core features.
To build & test locally, use:

```
npm run start org.api.crm3.dynamics.com
npm run start list
npm run start org.api.crm3.dynamics.com test-connection
```

### ADAL -> MSAL

As of version 2, dataverse-ify now uses MSAL for all authentication based on guidance given by https://docs.microsoft.com/en-us/azure/active-directory/develop/msal-node-migration
