# TAP_SCHEMA Migration to CloudSQL - Architecture & Implementation Breakdown

```{abstract}
Currently the TAP_SCHEMA database is packaged into a Docker image and pre-populated with schema metadata, built from the sdm_schemas YAML schema definitions using Felis. 
This image is deployed as a separate cluster database alongside the main TAP service.
This approach has various disadvantages which are described in this technote.
The proposed solution here is to migrate TAP_SCHEMA to a persistent CloudSQL instance, where Felis loads the schema metadata directly via a Kubernetes Job triggered by Helm hooks during our regular updates via ArgoCD.
```

## Add content here

See the [Documenteer documentation](https://documenteer.lsst.io/technotes/index.html) for tips on how to write and configure your new technote.
