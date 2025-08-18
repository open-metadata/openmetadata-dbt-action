# OpenMetadata DBT Ingestion with GitHub Actions

This repository demonstrates how to automate DBT metadata ingestion into OpenMetadata using GitHub Actions. It provides a complete example of running DBT workflows externally and ingesting the generated artifacts into OpenMetadata.

## Overview

This solution enables you to:
- Automatically ingest DBT metadata into OpenMetadata on every push
- Run ingestion workflows without direct access to the OpenMetadata server
- Store DBT artifacts locally and process them through GitHub Actions
- Maintain version control of your DBT metadata configuration

## Architecture

```
DBT Project → Generate Artifacts → GitHub Actions → OpenMetadata Ingestion → OpenMetadata Server
```

## Prerequisites

- An OpenMetadata instance (version 1.0+)
- DBT artifacts (manifest.json, catalog.json, run_results.json)
- GitHub repository with Actions enabled
- OpenMetadata JWT token for authentication

> **⚠️ Version Compatibility:** The `openmetadata-ingestion` package version MUST match your OpenMetadata server version. For example, if your OpenMetadata server is version 1.5.11, you must install `openmetadata-ingestion[dbt]==1.5.11`

## Repository Structure

```
.
├── .github/
│   └── workflows/
│       └── openmetadata-ingestion.yml    # GitHub Action workflow
├── artifacts/                            # Example location (can be anywhere in repo)
│   ├── catalog.json                      # DBT catalog artifact
│   ├── manifest.json                     # DBT manifest artifact
│   └── run_results.json                  # DBT run results artifact
└── openmetadata_dbt_config.yaml          # OpenMetadata ingestion configuration
```

> **Note:** DBT artifacts can be stored anywhere within your repository. The `artifacts/` folder is just an example. You can organize them in any directory structure that suits your project (e.g., `dbt/target/`, `build/dbt/`, etc.)

## Setup Instructions

### 1. Generate DBT Artifacts

First, generate your DBT artifacts by running these commands in your DBT project:

```bash
# Generate manifest.json
dbt compile

# Generate catalog.json
dbt docs generate

# Generate run_results.json
dbt run
```

Place these files in your repository. They can be stored in any directory structure you prefer (e.g., `artifacts/`, `dbt/target/`, `build/`, etc.).

### 2. Configure GitHub Secrets

You have two options for setting up secrets:

#### Option A: Environment Secrets (Recommended)
1. Go to Settings → Environments → New environment
2. Create an environment named `test` (or modify the workflow to use your environment name)
3. Add these secrets to the environment:

| Secret Name | Description | Example | Required |
|------------|-------------|---------|----------|
| `OPENMETADATA_HOST` | Your OpenMetadata API endpoint | `http://your-server:8585/api` | ✅ |
| `OPENMETADATA_JWT_TOKEN` | JWT token for authentication | `eyJhbGciOiJIUzI1NiIsInR5cCI6...` | ✅ |
| `OPENMETADATA_VERSION` | OpenMetadata server version | `1.9.1.0` | ✅ |

#### Option B: Repository Secrets
Add the secrets directly to Settings → Secrets and variables → Actions → Repository secrets (then remove the `environment: test` line from the workflow)

> **Important:** 
> - The `OPENMETADATA_VERSION` must exactly match your OpenMetadata server version
> - Check your server version in OpenMetadata UI → Settings → About
> - Common versions: `1.5.11`, `1.6.2`, `1.7.3`, `1.8.0`, `1.9.1.0`

### 3. Configuration File

The `openmetadata_dbt_config.yaml` file contains the ingestion configuration. **You need to update the following fields:**

#### Required Updates:

1. **`serviceName`**: Must match an existing database service name in your OpenMetadata instance
2. **File paths**: Update to match your actual artifact locations in the repository

```yaml
source:
  type: dbt
  serviceName: dbt_git_test  # ⚠️ UPDATE: Must match existing service in OpenMetadata
  sourceConfig:
    config:
      type: DBT
  dbtConfigSource:
    dbtConfigType: local
    # ⚠️ UPDATE: Change these paths to match your artifact locations
    dbtCatalogFilePath: /home/runner/work/openmetadata-dbt-action/openmetadata-dbt-action/artifacts/catalog.json
    dbtManifestFilePath: /home/runner/work/openmetadata-dbt-action/openmetadata-dbt-action/artifacts/manifest.json
    dbtRunResultsFilePath: /home/runner/work/openmetadata-dbt-action/openmetadata-dbt-action/artifacts/run_results.json

sink:
  type: metadata-rest
  config: {}

workflowConfig:
  loggerLevel: INFO
  openMetadataServerConfig:
    hostPort: $OPENMETADATA_HOST
    authProvider: openmetadata
    securityConfig:
      jwtToken: $OPENMETADATA_JWT_TOKEN
```

> **Important:** 
> - The `serviceName` must exactly match a DBT service that already exists in your OpenMetadata instance
> - If the service doesn't exist, you'll need to create it first in OpenMetadata
> - File paths are relative to the repository root

### 4. GitHub Action Workflow

The workflow (`.github/workflows/openmetadata-ingestion.yml`) automatically:
- Installs the OpenMetadata ingestion package
- Validates DBT artifacts exist
- Runs the metadata ingestion
- Uploads logs as artifacts for debugging

## Usage

### Automatic Ingestion

The ingestion runs automatically when:
- You push changes to `main` or `master` branches
- DBT artifacts are updated
- The configuration file is modified
- A pull request is opened

### Manual Ingestion

You can manually trigger the workflow:
1. Go to Actions tab in your GitHub repository
2. Select "OpenMetadata DBT Ingestion"
3. Click "Run workflow"

### Local Testing

To test the ingestion locally:

```bash
# Install the package (version must match your OpenMetadata server)
# Replace X.X.X with your OpenMetadata version (e.g., 1.5.11)
pip install "openmetadata-ingestion[dbt]==X.X.X"

# Set environment variables
export OPENMETADATA_HOST="http://localhost:8585/api"
export OPENMETADATA_JWT_TOKEN="your-jwt-token"

# Run ingestion
metadata ingest -c openmetadata_dbt_config.yaml
```

> **Note:** Always ensure the `openmetadata-ingestion` package version matches your OpenMetadata server version to avoid compatibility issues.

## Monitoring

- Check the Actions tab in GitHub for workflow runs
- Download ingestion logs from workflow artifacts
- View imported metadata in your OpenMetadata instance

## Troubleshooting

### Common Issues

1. **Version Mismatch Error**
   - Ensure `openmetadata-ingestion` package version matches your OpenMetadata server version exactly
   - Check server version: OpenMetadata UI → Settings → About
   - Update `OPENMETADATA_VERSION` secret in GitHub

2. **Authentication Failed**
   - Verify JWT token is valid and not expired
   - Check token has necessary permissions

3. **Service Not Found**
   - Ensure the `serviceName` in your YAML matches an existing DBT service in OpenMetadata
   - Create the service in OpenMetadata first if it doesn't exist

4. **Artifacts Not Found**
   - Verify DBT artifacts exist in your repository
   - Ensure file paths in configuration match actual locations
   - Remember: paths are relative to repository root

5. **Connection Timeout**
   - Verify OpenMetadata server is accessible
   - Check network/firewall settings

### Debug Mode

Enable debug logging by modifying the configuration:

```yaml
workflowConfig:
  loggerLevel: DEBUG
```

## Advanced Configuration

### Service Name Configuration

The `serviceName` must match an existing DBT service in OpenMetadata:

```yaml
source:
  serviceName: your_existing_dbt_service  # Must exist in OpenMetadata
```

To create a new DBT service in OpenMetadata:
1. Navigate to Services → Add Service → DBT
2. Configure the service with your DBT project details
3. Use the exact service name in your YAML configuration

### Additional Metadata

You can enhance the ingestion with additional metadata:

```yaml
source:
  sourceConfig:
    config:
      type: DBT
      dbtUpdateDescriptions: true  # Update descriptions from DBT
      includeTags: true            # Include DBT tags
      dbtClassificationName: PII   # Add classification tags
```

## Security Considerations

- Never commit JWT tokens or sensitive credentials
- Use GitHub Secrets for all sensitive information
- Regularly rotate JWT tokens
- Review GitHub Actions permissions

## Contributing

To contribute to this demonstration:

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Submit a pull request

## Resources

- [OpenMetadata Documentation](https://docs.open-metadata.org/)
- [DBT Connector Documentation](https://docs.open-metadata.org/latest/connectors/ingestion/workflows/dbt)
- [Running DBT Workflow Externally](https://docs.open-metadata.org/latest/connectors/ingestion/workflows/dbt/run-dbt-workflow-externally)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)

## License

This demonstration repository is provided as-is for educational purposes.

## Support

For issues related to:
- This demo: Open an issue in this repository
- OpenMetadata: Visit [OpenMetadata Slack](https://slack.open-metadata.org/)
- DBT: Visit [DBT Community](https://community.getdbt.com/)