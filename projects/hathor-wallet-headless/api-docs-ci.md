# Summary
This feature automates the process of uploading the updated API Docs for the Headless Wallet, while giving the developer a preview of the contents, error checking and linting.

A script is made available for the developer to make all the necessary checks locally and a GitHub Workflow is implemented to:
- Validate and lint the `api-docs.js` file on every PR to the `dev` branch
- Deploy the updated docs on every new successful release

# Motivation
Currently this upload is made manually by a team member, extracting data from the `src/api-docs.js` file and processing it to finally upload its contents to the [Headless Hathor Wallet API](https://wallet-headless.docs.hathor.network/) site.

# Guide-level explanation
### JS to JSON conversion
The platform that manages the OpenAPI Documentation static website, [Redoc](https://github.com/Redocly/redoc), only interacts with `.json` files. So we need to convert our `src/api-docs.js` file into a `.json` one whenever we want to use this platform.

A script should be built to make this conversion on the `scripts` folder, outputting the results to the `tmp/` folder. All following steps will be executed on this converted file. Some specific conditions to execute:
- Remove the `securitySchemes` from the `components` property
- Ensure the `default` property normally exported by the javascript `.toJSON()` is not present on the output json

This script will be referred to as `scripts/convert-docs.js` from now on.

### Linting and Preview
A developer should be able to run a `npm run docs` command with three outcomes:
- A validation check should be executed on the generated docs, alerting any errors found
- A lint should be executed on the generated docs, displaying its analysis results
- A local server should be fired, allowing the developer to visualize the results of the doc changes

This would be an automated version of manually executing the following commands:
```sh
# Validation and conversion
node ./scripts/convert-docs.js

# Linting
npx @redocly/cli lint tmp/api-docs.json

# Local server for validating
npx @redocly/cli preview-docs tmp/api-docs.json
xdg-open http://127.0.0.1:8080
```

> *Note:* The linter throws some errors on our current documentation. The first implementation of this script will certainly require a refactoring PR.

### PR Validation
A GitHub Workflow, configured to run on every PR merging to `dev`, should run the conversion script and linter to check for errors on the generated documentation. This should be part of the CI process.

### Deployment
On the deployment pipeline, namely when there is a new `v*` label on the `master` branch, a new workflow should also upload the generated `json` file to the production documentation S3 bucket. This will update the website to the community.

> *Note:* All connection data with the S3 should be implemented as secrets. Refer to the [`docker.yml`](https://github.com/HathorNetwork/hathor-wallet-headless/blob/master/.github/workflows/docker.yml) file for reference.

# Task breakdown

### Milestone: Developer feedback - 0.5 dev days
- Write the conversion script - 0.2 dev days
- Implement the local validation environment - 0.3 dev days
  - Write the `shell` command to convert and lint the api docs
  - Also start the local server for manual conference.
  - Associate it with the `docs` script on `package.json`
- Merge this on the `dev` branch

### Milestone: CI workflow - 0.5 dev days
- Implement a workflow for validating and lint checking when merging to `dev` - 0.1 dev days
- Implement a workflow to deploy when making a new release - 0.1 dev days
- Test these new workflows on PR merges and releases - 0.3 dev days
