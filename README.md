# Data Gateway

A deployment of a GraphQL router, acting as the Data Gateway for Diamond Light Source.

## Docs

Docs can be found [on github pages](https://diamondlightsource.github.io/graph-federation/)

## Structure

- `supergraph-schema.yaml` & `schema/`: A description of how subgraph schemas and how they are composed to produce the supergraph schema.
- `apps.yaml` & `charts/apps/`: An [ArgoCD](https://argoproj.github.io/cd/) [App-of-Apps](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/#app-of-apps-pattern) used to deploy the other charts in various configurations
- `charts/graph`: A Helm chart used to deploy the [Apollo Router](https://www.apollographql.com/docs/router/)
- `charts/monitoring`: A Helm chart used to deploy [Prometheus](https://prometheus.io/) and [Jaeger](https://www.jaegertracing.io/) for observability
- `charts/supergraph`: A Helm chart used to deploy the supergraph schema
- `action.yaml`: A [GitHub action](https://github.com/features/actions) used to create subgraph schema update pull requests
- `mkdocs.yaml` & `docs/`: User facing documentation, built with [mkdocs](https://www.mkdocs.org/)

## Action

### Update Supergraph Schema

This workflow may be used to create or update a Subgraph Schema by adding the schema to the `schema/` directory and an entry in the `supergraph-config.yaml` of this repository.
The action can be used to simply check that the schema will federate by setting `publish` to `false`.

#### Usage

##### Inputs

```yaml
- uses: diamondlightsource/graph-federation@main
  with:
    # A unique name given to the subgraph.
    # Required.
    name:

    # The public-facing URL of the subgraph.
    # Required.
    routing-url:

    # The name of an artifact from this workflow run containing the subgraph schema.
    # Required.
    subgraph-schema-artifact:

    # The name of the subgraph schema file within the artifact.
    # Required.
    subgraph-schema-filename:

    # The name of the artifact to be created containing the supergraph schema.
    # Optional. Default is 'supergraph'
    supergraph-schema-artifact:

    # The name of the supergraph schema file within the created artifact.
    # Optional. Default is 'supergraph.graphql'
    supergraph-schema-filename:

    # The ID of the GitHub App used to create the commit / pull request
    # Required when publish is true.
    github-app-id:
   
    # The private key of the GitHub App used to create the commit / pull request
    # Required when publish is true.
    github-app-private-key:

    # A boolean value which determines whether a pull request should be created
    # Optional. Default is ${{ github.event_name == 'push' && startsWith(github.ref, 'refs/tags') }}
    publish:
```

##### Outputs

| Name                           | Description                                              | Example                                                                   |
| ------------------------------ | -------------------------------------------------------- | ------------------------------------------------------------------------- |
| supergraph-schema-artifact-id  | The id of the artifact containing the supergraph schema  | 1234                                                                      |
| supergraph-schema-artifact-url | The url of the artifact containing the supergraph schema | <https://github.com/example-org/example-repo/actions/runs/1/artifacts/1234> |

##### Example

```yaml
steps:
  - name: Create Test Subgraph schema
    run: >
      echo "
        schema {
          query: Query
        }
        type Query {
          _empty: String
        }
      " > test-schema.graphql

  - name: Upload Test Subgraph schema
    uses: actions/upload-artifact@v4.4.3
    with:
      name: test-schema
      path: test-schema.graphql
 
  - name: Update Supergraph
    uses: diamondlightsource/graph-federation@main
    with:
      name: test
      routing-url: https://example.com/graphql
      subgraph-schema-artifact: test-schema
      subgraph-schema-filename: test-schema.graphql
      github-app-id: 1010045
      github-app-private-key: ${{ secrets.GRAPH_FEDERATOR }}
      publish: false
```
