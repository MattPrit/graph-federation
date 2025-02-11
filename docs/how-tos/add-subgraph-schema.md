# Add Subgraph Schema

## Preface

This guide will explain how to introduce or update the schema of a subgraph via a Pull Request to the [graph-federation](https://github.com/DiamondLightSource/graph-federation/) repository.
Code snippets will be given in the form of GitHub actions.

## Generate Subgraph Schema

To begin, we must generate the subgraph schema of our application.
How this is performed will differ depending on our chosen language and GraphQL library.
For instance;

- [`strawberry` (Python)](https://strawberry.rocks/) users can use`strawberry export-schema my_package:schema` as described in the [schema export docs](https://strawberry.rocks/docs/guides/schema-export).
- [`async-graphql` (Rust)](https://crates.io/crates/async-graphql) users should create a command which builds and exports the schema as described on the [SDL export page of the book](https://async-graphql.github.io/async-graphql/en/sdl_export.html).

Our CI job should checkout the source, setup the environment and generate the schema, as shown below.
It is useful to upload the schema as an artifact to allow download both in future workflow jobs and for review.

!!! example "GitHub Actions Schema Generation"

    ```yaml
    jobs:
      generate_schema:
        runs-on: ubuntu-latest
        steps:
          - name: Checkout source
            uses: actions/checkout@v4.1.7

          - name: Setup Python
            uses: actions/setup-python@v5.2.0

          - name: Install (Dev) Dependencies
            run: pip install .[dev]
            
          - name: Generate Schema
            run: strawberry export-schema my_app:schema > schema.graphql

          - name: Upload Schema Artifact
            uses: actions/upload-artifact@v4.4.0
            with:
              name: graphql-schema
              path: schema.graphql
    ```

!!! example "GitLab CI Schema Generation"

    ```yaml
    generate_schema:
      image: python:3
      stage: my-stage
      tags: ["my-tag"]
      script:
        - pip install .[dev]
        - strawberry export-schema my_app:schema > schema.graphql
      artifacts:
        paths:
          - schema.graphql
        expose_as: "GraphQL schema"
    ```

## Check Schema Validity

### Locally

During local development, to check our schema is valid we will use the [Apollo Rover CLI](https://www.apollographql.com/docs/rover/).
We can retrieve the exisitng schema and configuration by cloning the [`graph-federation` repository](https://github.com/DiamondLightSource/graph-federation/) - in which we will find a `supergraph-config.yaml` file describing the federation and a number of schemas in the `schema/` directory.

Once cloned, we can add our subgraph definition to the `schema/` directory and a descriptor to the `supergraph-config.yaml`.
This should reside under the `subgraphs` key, be given a key which describes our service and should contain:

- A `routing_url` which points to the production instance of our service
- A `schema.file` which points to our subgraph schema in the `schema/` directory

We can now use the `Apollo Rover CLI` to compose the supergraph, as:

```bash
rover supergraph compose --config supergraph-config.yaml --elv2-license accept
```

### GitHub CI

As part of our continuous integration process on GitHub, we can use the [`diamondlightsource/graph-federation`](https://github.com/DiamondLightSource/graph-federation) action in GitHub actions;
Setting `publish` to false will simply compose the new schema, without creating a pull request.

!!! example "GitHub Actions Validity Checking"

    ```yaml
    jobs:
      federate:
        runs-on: ubuntu-latest
        needs: generate_schema
        uses: diamondlightsource/graph-federation@main
        with:
          name: example
          routing-url: https://example.com/graphql
          subgraph-schema-artifact: ${{ jobs.generate_schema.artifact }}
          subgraph-schema-artifact: ${{ jobs.generate_schema.filename }}
          publish: false
    ```

### GitLab CI

As part of our continuous integration process on GitLab, we can use the [`graph-federation`]([https://github.com/DiamondLightSource/graph-federation](https://gitlab.diamond.ac.uk/lims/gitlab-ci-components#graph-federation-component) component in GitLab CI pipelines;
Setting `publish` to false will simply compose the new schema, without creating a pull request.

!!! example "GitLab CI Validity Checking"

    ```yaml
    include:
      - component: https://gilab.diamond.ac.uk/lims/gitlab-ci-components/graph-federation-component@0.1.0
        inputs:
          tags: ["my-tag"]
          stage: my-stage
          needs: ["generate_schema"]
          github_repo: graph-federation
          github_repo_owner: DiamondLightSource
          subgraph_name: example
          subgraph_routing_url: https://example.com/graphql
          subgraph_schema_file: schema.graphql
          subgraph_version: 0.0.0
          publish: false
    ```

## Pull Request Generation

To update the deployed supergraph schema, we should create a Pull Request to the [`graph-federation` repository](https://github.com/DiamondLightSource/graph-federation/) with the desired subgraph schema and relevant configuration changes.

<!-- markdownlint-disable-next-line MD024 -->
### GitHub CI

A [GitHub App](https://docs.github.com/en/apps/creating-github-apps/about-creating-github-apps/about-creating-github-apps) can be created in order to act as a bot account capable of pushing to branches and creating Pull Requests from the GitHub actions of another repository. The [`diamondlightsource/graph-federation`](https://github.com/DiamondLightSource/graph-federation) action can be used to perform the necessary schema composition, branch update, and pull request generation with `publish` set to true.

!!! tip

    GitHub Apps can be requested in the [#github-requests slack channel](https://diamondlightsource.slack.com/archives/C06A18ZPP44).

!!! example "Full GitHub Actions Workflow"

    ```yaml
    jobs:
      update:
        runs-on: ubuntu-latest
        needs: generate_schema
        uses: diamondlightsource/graph-federation@main
        with:
          name: example
          routing-url: https://example.com/graphql
          subgraph-schema-artifact: ${{ jobs.generate_schema.artifact }}
          subgraph-schema-artifact: ${{ jobs.generate_schema.filename }}
          github-app-id: 1010045
          github-app-private-key: ${{ secrets.GRAPH_FEDERATOR }}
          publish: ${{ github.event_name == 'push' && startsWith(github.ref, 'refs/tags') }}
    ```

<!-- markdownlint-disable-next-line MD024 -->
### GitLab CI

A [GitHub App](https://docs.github.com/en/apps/creating-github-apps/about-creating-github-apps/about-creating-github-apps) can be created in order to act as a bot account capable of pushing to branches and creating Pull Requests from the CI of a GitLab repository. The the [`graph-federation`]([https://github.com/DiamondLightSource/graph-federation](https://gitlab.diamond.ac.uk/lims/gitlab-ci-components#graph-federation-component) component can be used to perform the necessary schema composition, branch update, and pull request generation with `publish` set to true.

!!! tip

    GitHub Apps can be requested in the [#github-requests slack channel](https://diamondlightsource.slack.com/archives/C06A18ZPP44).

!!! example "Full GitLab CI Workflow"

    ```yaml
    include:
      - component: https://gilab.diamond.ac.uk/lims/gitlab-ci-components/graph-federation-component@0.1.0
        inputs:
          tags: ["my-tag"]
          stage: my-stage
          needs: ["generate_schema"]
          github_app_id: $MY_GITHUB_APP_ID
          github_installation_id: $MY_GITHUB_INSTALLATION_ID
          github_private_key_variable: MY_GITHUB_PRIVATE_KEY_VARIABLE_NAME
          github_repo: graph-federation
          github_repo_owner: DiamondLightSource
          subgraph_name: example
          subgraph_routing_url: https://example.com/graphql
          subgraph_schema_file: schema.graphql
          subgraph_version: 0.0.0
          publish: true
    ```
