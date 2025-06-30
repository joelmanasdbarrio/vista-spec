It uses [OpenAPI/Swagger](https://swagger.io/specification/) to define the project's API specification and work as the single source of truth for all endpoints and data structures that Vista has.

This package is published to a private registry of GitHub Packages. To be able to access this private registry in your project, update your .npmrc file with the registry URL and a valid auth token:

```bash
@joelmanasdbarrio:registry=https://npm.pkg.github.com/
//npm.pkg.github.com/:_authToken=${NODE_AUTH_TOKEN}
```

> ⚠️ Replace `${NODE_AUTH_TOKEN}` with a valid PAT token (classic token) that has read/write permissions on the repository and packages.

> ⚠️ Do NOT upload the `.npmrc` file with your token to the repository. If it's not already there, add this file to the `.gitignore` or use the `.npmrc` file from your personal folder.

## Installation

<Steps>

1. Install all dependencies:
    ```bash
    npm ci
    ```

2. Validate specification:
    ```bash
    npm run validate
    ```
  
3. Generate TypeScript interfaces:
    ```bash
    npm run generate:types
    ```

</Steps>

## Versions

The `@joelmanasdbarrio/vista-spec` package leverages semantic versioning and dist-tags from NPM to publish new packages. Packages published from:

> ⚠️ Package and specification version must remain the same to avoid inconsistencies.

- `dev`: `1.3.0-dev.N` in `dev` tag (experimental), where `N` is the commit number. From a `feature` branch, create a new merge-request into `dev` once the implementation is finished to trigger the GitHub workflow that validates and publishes a new experimental version of the package. Developers can commit as many changes as they need without having to update the final version of the package/specification while having an active merge-request since it will autoincrement its value based on the commit number on each run.
    <details>
    <summary>Install experimental version</summary>
    
    Install latest experimental version:
    
    ```bash
    npm install @joelmanasdbarrio/vista-spec@dev
    ```
    
    Install specific experimental version:
    
    ```bash
    npm install @joelmanasdbarrio/vista-spec@1.0.0-dev.N
    ```
    
    </details>
    
- `release`: `1.2.3-rc` in `rc` tag (release candidate). Combines multiple features into a single `release` branch. Create a new merge-request into `main` to trigger the GitHub workflow that validates and publishes a new release candidate version of the package. Developers can merge as many features as they need without having to update the final version of the package/specification while having an active merge-request since it will autoincrement its value based on the commit number on each run.
    <details>
    <summary>Install release candidate version</summary>
    
    Install latest release candidate version:
    
    ```bash
    npm install @joelmanasdbarrio/vista-spec@rc
    ```
    
    Install specific release candidate version:
    
    ```bash
    npm install @joelmanasdbarrio/vista-spec@1.0.0-rc.N
    ```
    
    </details>
    
- `main`: `1.2.3` in `latest` tag (stable). Complete a merge-request from a `release` branch into `main` to trigger the GitHub workflow that validates and publishes a new latest stable version of the package.
    <details>
    <summary>Install stable version</summary>
    
    Install latest stable version:
    
    ```bash
    npm install @joelmanasdbarrio/vista-spec@latest
    ```
    
    or just
    
    ```bash
    npm install @joelmanasdbarrio/vista-spec
    ```
    
    Install specific stable version:
    
    ```bash
    npm install @joelmanasdbarrio/vista-spec@1.0.0
    ```
    
    </details>
    

### Semantic versioning

NPM uses SemVer (Semantic Versioning) to define the values for its versions: MAJOR.MINOR.PATCH.

For example `1.2.3`:

- 1 = major changes (breaking changes).
- 2 = new functionalities that do not affect compatibility.
- 3 = fixes and improvements that do not change the current definition.

|     |     |
| --- | --- |
| Command | What it does |
| `npm version patch` | Upgrades `1.2.3` → `1.2.4` |
| `npm version minor` | Upgrades `1.2.3` → `1.3.0` |
| `npm version major` | Upgrades `1.2.3` → `2.0.0` |
| `npm version prerelease --preid=dev` | Upgrades `1.2.3` → `1.2.4-dev.0` (prerelease with suffix) |
| `npm version prepatch --preid=beta` | Upgrades `1.2.3` → `1.2.4-beta.0` |

## Workflows

They are stored inside `.github/workflows`.

### Deployment

`.github/workflows/deployment.yml`. This file runs all jobs in order:

```yaml
name: CI & Publish OpenAPI Package

permissions:
  contents: write
  packages: write

on:
  pull_request:
    branches: [main, release, dev]
    types: [opened, synchronize, reopened]
  push:
    branches: [main]

jobs:
```

<details>
<summary>validate-specification</summary>

Checks if a major/minor version number has already been released:

```yaml
validate-specification:
  if: github.event_name == 'pull_request'
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v3
    - uses: actions/setup-node@v4
      with:
        node-version: 22
        registry-url: https://npm.pkg.github.com
    - run: npm ci
    - run: npm run validate
      name: Validate OpenAPI spec
```

</details>
<details>
<summary>check-version</summary>

Compares the specification and package versions to assert they are equal:

```yaml
check-version:
  if: github.event_name == 'pull_request'
  runs-on: ubuntu-latest
  outputs:
    base_version: ${{ steps.pkg.outputs.version }}
  steps:
    - uses: actions/checkout@v3
    - uses: actions/setup-node@v4
      with:
        node-version: 22
        registry-url: https://npm.pkg.github.com
    - run: npm ci
    - name: Read package.json version
      id: pkg
      uses: ActionsTools/read-json-action@main
      with: { file_path: package.json }
    - name: Read OpenAPI info.version
      id: spec
      uses: actions-tools/yaml-outputs@v2.1
      with:
        file-path: openapi/openapi-rest.yaml
        separator: '__'
    - run: |
        if [ "${{ steps.pkg.outputs.version }}" != "${{ steps.spec.outputs.info__version }}" ]; then
          echo "::error ::Version mismatch: package.json (${{
          steps.pkg.outputs.version }}) vs OpenAPI spec (${{
          steps.spec.outputs.info__version }})"
          exit 1
        fi
      name: Ensure versions match
```

</details>
<details>
<summary>generate-types</summary>

Generates TypeScript interfaces based on the specification:

```yaml
generate-types:
  needs: check-version
  if: github.event_name == 'pull_request' && (github.base_ref == 'dev' || github.base_ref == 'release' || github.base_ref == 'main')
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v3
    - uses: actions/setup-node@v4
      with:
        node-version: 22
        registry-url: https://npm.pkg.github.com
    - run: npm ci
    - run: rm -rf types/
      name: Clean previous types
    - run: npm run generate:types
      name: Generate TS types
    - uses: EndBug/add-and-commit@v9
      with:
        author_name: github-actions[bot]
        author_email: 41898282+github-actions[bot]@users.noreply.github.com
        message: "regenerate OpenAPI types"
```

</details>
<details>
<summary>publish-package-\[dev, release, main\]</summary>

Creates and publishes a new version of the `@joelmanasdbarrio/vista-spec` package with a specific version, depending on the environment:

```yaml
publish-package-dev:
  needs: [generate-types, check-version]
  if: github.event.action != 'closed' && (github.base_ref == 'dev' && startsWith(github.head_ref, 'feature/'))
  runs-on: ubuntu-latest
  env:
    NODE_AUTH_TOKEN: ${{ secrets.DEPLOYMENT_TOKEN }}
  steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-node@v4
      with:
        node-version: 22
        registry-url: https://npm.pkg.github.com
    - name: Calculate version
      id: version
      run: |
        BASE_VERSION=${{ needs.check-version.outputs.base_version }}
        VERSION=${BASE_VERSION}-dev.$GITHUB_RUN_NUMBER
        echo "VERSION=$VERSION" >> $GITHUB_ENV
        echo "new_version=$VERSION" >> $GITHUB_OUTPUT
    - name: Publish package
      run: |
        npm version $VERSION --no-git-tag-version
        npm publish --tag dev --access public
    - name: Comment on PR with result
      if: success()
      uses: peter-evans/create-or-update-comment@v2
      with:
        token: ${{ secrets.DEPLOYMENT_TOKEN }}
        issue-number: ${{ github.event.pull_request.number }}
        body: |
          ✅ Published **development** version as `@joelmanasdbarrio/vista-spec@${{ steps.version.outputs.new_version }}`.
    - name: Comment on PR with error
      if: failure()
      uses: peter-evans/create-or-update-comment@v2
      with:
        token: ${{ secrets.DEPLOYMENT_TOKEN }}
        issue-number: ${{ github.event.pull_request.number }}
        body: |
          ❌ Error on publication. Review workflow logs for more information.

publish-package-release:
  needs: [generate-types, check-version]
  if: github.event.action != 'closed' && (github.base_ref == 'main' && startsWith(github.head_ref, 'release/'))
  runs-on: ubuntu-latest
  env:
    NODE_AUTH_TOKEN: ${{ secrets.DEPLOYMENT_TOKEN }}
  steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-node@v4
      with:
        node-version: 22
        registry-url: https://npm.pkg.github.com
    - name: Calculate version
      id: version
      run: |
        BASE_VERSION=${{ needs.check-version.outputs.base_version }}
        VERSION=${BASE_VERSION}-rc.$GITHUB_RUN_NUMBER
        echo "VERSION=$VERSION" >> $GITHUB_ENV
        echo "new_version=$VERSION" >> $GITHUB_OUTPUT
    - name: Publish package
      run: |
        npm version $VERSION --no-git-tag-version
        npm publish --tag rc --access public
    - name: Comment on PR with result
      if: success()
      uses: peter-evans/create-or-update-comment@v2
      with:
        token: ${{ secrets.DEPLOYMENT_TOKEN }}
        issue-number: ${{ github.event.pull_request.number }}
        body: |
          ✅ Published **release** candidate version as `@joelmanasdbarrio/vista-spec@${{ steps.version.outputs.new_version }}`.
    - name: Comment on PR with error
      if: failure()
      uses: peter-evans/create-or-update-comment@v2
      with:
        token: ${{ secrets.DEPLOYMENT_TOKEN }}
        issue-number: ${{ github.event.pull_request.number }}
        body: |
          ❌ Error on publication. Review workflow logs for more information.

publish-package-main:
  needs: [generate-types, check-version]
  if: github.ref == 'refs/heads/main'
  runs-on: ubuntu-latest
  env:
    NODE_AUTH_TOKEN: ${{ secrets.DEPLOYMENT_TOKEN }}
  steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-node@v4
      with:
        node-version: 22
        registry-url: https://npm.pkg.github.com
    - run: npm ci
    - name: Calculate version
      id: version
      run: |
        BASE_VERSION=${{ needs.check-version.outputs.base_version }}
        VERSION=${BASE_VERSION}
        echo "VERSION=$VERSION" >> $GITHUB_ENV
        echo "new_version=$VERSION" >> $GITHUB_OUTPUT
    - name: Publish package
      run: |
        npm version $VERSION --no-git-tag-version
        npm publish --tag latest --access public
    - name: Add job summary
      run: |
        echo "✅ Published **main** version as `@joelmanasdbarrio/vista-spec@${{ steps.version.outputs.package_version }}`." >> $GITHUB_STEP_SUMMARY
```

Experimental and Release Candidate versions are autoincremented based on commit number to avoid version overrides.

</details>

### Clean up old packages

Versions published with tags `dev` and `rc` are automatically removed after `CLEANUP_AFTER` days (view online value at [GitHub Actions variables](https://github.com/joelmanasdbarrio/vista-spec/settings/variables/actions)).

<details>
<summary>cleanup</summary>

```yaml
name: Clean up old dev/rc versions

on:
  schedule:
    - cron: '0 6 * * *'
  workflow_dispatch:

permissions:
  contents: read
  packages: write

env:
  PACKAGE_TYPE: npm
  PACKAGE_NAME: vista-spec
  CLEANUP_AFTER: ${{ vars.CLEANUP_AFTER }}
  GH_TOKEN: ${{ secrets.CLEANUP_TOKEN }}

jobs:
  cleanup:
    runs-on: ubuntu-latest
    steps:
      - name: Listar versiones
        run: |
          gh api \
            -H "Accept: application/vnd.github+json" \
            /user/packages/$PACKAGE_TYPE/$PACKAGE_NAME/versions \
            --paginate > versions.json

      - name: Eliminar versiones dev/rc antiguas
        run: |
          now=$(date +%s)
          cutoff=$(( CLEANUP_AFTER * 86400 ))
          jq -c '.[]' versions.json | while read pkg; do
            name=$(jq -r .name <<<"$pkg")
            created=$(jq -r .created_at <<<"$pkg")
            created_ts=$(date -d "$created" +%s)
            age=$(( now - created_ts ))
            if [[ $age -gt $cutoff && "$name" =~ (dev|rc) ]]; then
              id=$(jq -r .id <<<"$pkg")
              days=$(( age / 86400 ))
              echo "Deleting $name (id=$id), $days days old"
              gh api --method DELETE \
                -H "Accept: application/vnd.github+json" \
                /user/packages/$PACKAGE_TYPE/$PACKAGE_NAME/versions/$id
            fi
          done
```

</details>