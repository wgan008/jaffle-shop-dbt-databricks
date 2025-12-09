# Deploying Jaffle Shop dbt Project with Databricks Asset Bundles

This guide explains how to deploy the Jaffle Shop dbt project to Databricks using Databricks Asset Bundles (DAB) with Git integration.

## Prerequisites

1. **Databricks CLI** installed and configured:
   ```bash
   # Install Databricks CLI
   pip install databricks-cli

   # Or via Homebrew (macOS)
   brew tap databricks/tap
   brew install databricks
   ```

2. **Authentication** set up for your Databricks workspace:
   ```bash
   # Use OAuth (recommended)
   databricks auth login --host https://e2-demo-field-eng.cloud.databricks.com
   ```

3. **Git repository** accessible from Databricks:
   - Repository: https://github.com/wgan008/jaffle-shop-dbt-databricks.git
   - Ensure your Databricks workspace can access GitHub
   - For private repos, configure Git credentials in Databricks

4. **Verify CLI setup**:
   ```bash
   databricks workspace list /
   ```

## How It Works

The bundle uses **Git integration** to clone your repository directly into job runs:
- No file uploads needed
- Code is pulled from GitHub on each job run
- Uses the `main` branch by default (configurable)
- Ensures you're always running the latest committed code

## Bundle Configuration

Key configurations in `databricks.yml`:

- **Git source**: Clones from `https://github.com/wgan008/jaffle-shop-dbt-databricks.git`
- **Branch**: `main` (configurable via `git_branch` variable)
- **Catalog**: `vera_demo_suite` (configurable)
- **SQL Warehouse**: Uses warehouse ID `862f1d757f0424f7`

## Deployment Steps

### 1. Validate the Bundle

Before deploying, validate the bundle configuration:

```bash
# Validate dev target
databricks bundle validate -t dev --profile e2-demo-field-eng

# Validate prod target
databricks bundle validate -t prod --profile e2-demo-field-eng
```

This checks for syntax errors and configuration issues.

### 2. Deploy to Development

Deploy to the development environment:

```bash
databricks bundle deploy --target dev --profile e2-demo-field-eng
```

This will:
- Create the Jaffle Shop dbt job in your workspace
- Configure Git integration to clone from the repository
- Set up job tasks for seed, run, and test
- Configure the job to use your SQL Warehouse

### 3. Deploy to Production

Deploy to the production environment:

```bash
databricks bundle deploy --target prod --profile e2-demo-field-eng
```

### 4. Run the Job

After deployment, run the job manually:

```bash
databricks bundle run jaffle_shop_dbt_job --target dev
```

Or trigger it from the Databricks workspace UI:
1. Go to **Workflows** in the Databricks workspace
2. Find "Jaffle Shop dbt - dev" (or prod)
3. Click **Run now**

The job will:
1. Clone the repository from GitHub
2. Run `dbt seed` to load data
3. Run `dbt run` to build models
4. Run `dbt test` to validate results

### 5. Monitor the Job

View job runs in the Databricks UI:
1. Navigate to **Workflows**
2. Click on your job
3. View run history and logs

Or use the CLI:
```bash
# List recent runs
databricks jobs list-runs --job-id <job-id>
```

## Configuration Variables

You can override default variables during deployment:

```bash
# Custom catalog and schema
databricks bundle deploy --target dev \
  --var="catalog=my_catalog" \
  --var="schema=my_schema"

# Use a different branch
databricks bundle deploy --target dev \
  --var="git_branch=feature-branch"

# Custom warehouse
databricks bundle deploy --target dev \
  --var="warehouse_id=your_warehouse_id"
```

## Environment-Specific Settings

### Development (`dev` target)
- Schema: `jaffle_shop_dev_<your_username>`
- Workspace path: `~/.bundle/jaffle-shop-dbt/dev`
- Mode: development
- Schedule: Paused (run manually)

### Production (`prod` target)
- Schema: `jaffle_shop_prod`
- Workspace path: `/Workspace/.bundles/jaffle-shop-dbt/prod`
- Mode: production
- Schedule: Daily at 2 AM UTC

## Job Configuration

The deployed job includes three tasks that run sequentially:

1. **dbt_seed** - Loads CSV seed data from the repository
2. **dbt_run** - Runs all dbt models (depends on seed)
3. **dbt_test** - Runs all dbt tests (depends on run)

Each task:
- Uses SQL Warehouse for execution (no cluster required for dbt tasks)
- Runs against the configured catalog and schema
- Has access to the cloned Git repository

## Updating the Deployment

To update the job configuration:

```bash
# Make changes to databricks.yml
# Then redeploy
databricks bundle deploy --target dev
```

**Note**: Code changes in your Git repository will be pulled automatically on each job run. You don't need to redeploy for code changes, just commit to GitHub!

## Updating dbt Code

To update your dbt models:

1. Make changes locally
2. Commit and push to GitHub:
   ```bash
   git add .
   git commit -m "Update dbt models"
   git push origin main
   ```
3. Run the job - it will use the latest code from GitHub

## Destroying the Deployment

To remove the deployed resources:

```bash
databricks bundle destroy --target dev
```

This will delete the job from Databricks. Your Git repository remains unchanged.

## Troubleshooting

### Authentication Issues
If you encounter authentication errors:
```bash
# Re-authenticate
databricks auth login --host https://e2-demo-field-eng.cloud.databricks.com

# Verify authentication
databricks workspace list /
```

### Git Access Issues
If the job fails to clone the repository:
- Verify the repository is public, or configure Git credentials in Databricks
- Check the repository URL is correct
- Ensure your workspace has internet access

### Job Failures
Check job logs in the Databricks UI:
1. Go to **Workflows**
2. Click on the job name
3. View the latest run
4. Expand each task to see logs

Common issues:
- **dbt connection errors**: Check catalog, schema, and warehouse configuration
- **Git clone errors**: Verify repository access
- **SQL errors**: Review dbt model SQL and test locally first

### Validation Errors
If bundle validation fails:
```bash
# Get detailed validation output
databricks bundle validate --verbose
```

## Workflow Best Practices

1. **Test Locally First**:
   ```bash
   dbt debug
   dbt seed
   dbt run
   dbt test
   ```

2. **Use Feature Branches**: Test changes in a feature branch before merging to main
   ```bash
   databricks bundle deploy --target dev --var="git_branch=feature-xyz"
   ```

3. **Deploy Dev Before Prod**: Always test in development first

4. **Monitor Job Runs**: Set up email notifications (configured in `databricks.yml`)

5. **Use Service Principals for Prod**: Configure `run_as` in the prod target

6. **Version Control Everything**: Commit `databricks.yml` changes to Git

## Additional Configuration

### Change the Schedule

Edit `databricks.yml` to modify the cron expression:

```yaml
schedule:
  quartz_cron_expression: "0 0 2 * * ?"  # Daily at 2 AM UTC
  timezone_id: "UTC"
  pause_status: UNPAUSED  # Set to PAUSED to disable
```

### Add More Tasks

Add additional dbt tasks in `databricks.yml`:

```yaml
- task_key: dbt_docs_generate
  description: "Generate dbt documentation"
  depends_on:
    - task_key: dbt_test
  job_cluster_key: dbt_cluster
  dbt_task:
    project_directory: ""
    commands:
      - "dbt docs generate"
    warehouse_id: ${var.warehouse_id}
```

### Configure Notifications

Update email notifications in `databricks.yml`:

```yaml
email_notifications:
  on_failure:
    - your-email@example.com
  on_success:
    - your-email@example.com
```

## Additional Resources

- [Databricks Asset Bundles Documentation](https://docs.databricks.com/dev-tools/bundles/index.html)
- [Databricks Git Integration](https://docs.databricks.com/repos/index.html)
- [dbt Databricks Documentation](https://docs.getdbt.com/reference/warehouse-setups/databricks-setup)
- [Databricks CLI Reference](https://docs.databricks.com/dev-tools/cli/index.html)
