
# ============================================================
# 1. WORKFLOW NAME
# ============================================================
# Defines the name displayed in the GitHub Actions interface.
# This helps developers identify the workflow in the Actions tab.

name: CI/CD Pipeline


# ============================================================
# 2. WORKFLOW TRIGGERS (on)
# ============================================================
# Defines WHEN the workflow should execute.
# A workflow can have one or multiple triggers.

on:

  # Runs when code is pushed to specified branches.
  push:
    branches:
      - main
      - develop

    # Optional: Run only when certain files change.
    paths:
      - "src/**"
      - "tests/**"
      - "package.json"
      - ".github/workflows/**"

  # Runs when a pull request is opened or updated.
  pull_request:
    branches:
      - main
    types:
      - opened
      - synchronize
      - reopened

  # Allows manual execution from the GitHub interface.
  workflow_dispatch:
    inputs:
      environment:
        description: "Deployment environment"
        required: true
        type: choice
        options:
          - staging
          - production

  # Runs automatically on a schedule.
  schedule:
    - cron: "0 2 * * 1"

  # Runs when another workflow completes.
  workflow_run:
    workflows:
      - "Build Application"
    types:
      - completed


# ============================================================
# 3. ENVIRONMENT VARIABLES (env)
# ============================================================
# Defines variables accessible to workflows, jobs, or steps.
# These are configuration values, not a secure secret store.

env:
  NODE_ENV: production
  APP_NAME: my-application
  CI: true


# ============================================================
# 4. PERMISSIONS
# ============================================================
# Controls what the workflow is allowed to do with GitHub resources.
# Follow the principle of least privilege.

permissions:
  contents: read
  pull-requests: write
  checks: write


# ============================================================
# 5. CONCURRENCY
# ============================================================
# Controls simultaneous workflow executions.
# Useful for preventing outdated deployments from running.

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true


# ============================================================
# 6. DEFAULTS
# ============================================================
# Sets default configurations for commands and shells.

defaults:
  run:
    shell: bash


# ============================================================
# 7. JOBS
# ============================================================
# Jobs are independent units of work within a workflow.
# Each job runs on a runner and contains one or more steps.

jobs:

  # ----------------------------------------------------------
  # JOB 1: CODE QUALITY AND TESTING
  # ----------------------------------------------------------

  test:

    # Unique identifier for this job.
    name: Run Tests

    # Specifies the machine or environment that executes the job.
    runs-on: ubuntu-latest

    # Optional: Job-specific environment variables.
    env:
      TEST_ENV: testing

    # Optional: Define a timeout to prevent jobs running forever.
    timeout-minutes: 15

    # Optional: Run this job only when a condition is true.
    if: github.event_name != 'schedule'

    # Optional: Define the execution strategy.
    strategy:
      fail-fast: false

      # Run the job against multiple versions.
      matrix:
        node-version:
          - 20
          - 22

    # Steps execute sequentially within the job.
    steps:

      # Step 1: Retrieve repository code.
      - name: Checkout repository
        uses: actions/checkout@v4

      # Step 2: Install the required runtime.
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: npm

      # Step 3: Install dependencies.
      - name: Install dependencies
        run: npm ci

      # Step 4: Run linting.
      - name: Run ESLint
        run: npm run lint

      # Step 5: Run automated tests.
      - name: Run tests
        run: npm test

      # Step 6: Build the application.
      - name: Build application
        run: npm run build

      # Step 7: Save generated files.
      - name: Upload build artifacts
        uses: actions/upload-artifact@v4
        with:
          name: build-${{ matrix.node-version }}
          path: dist/
          retention-days: 7


  # ----------------------------------------------------------
  # JOB 2: DEPLOYMENT
  # ----------------------------------------------------------

  deploy:

    name: Deploy Application

    # This job waits for the test job to succeed.
    needs: test

    runs-on: ubuntu-latest

    # Defines the GitHub environment for deployment.
    # Configure this environment under repository Settings.
    environment:
      name: staging
      url: ${{ steps.deploy-step.outputs.deployment-url }}

    # Restrict deployment to the main branch.
    if: github.ref == 'refs/heads/main'

    steps:

      # Retrieve repository code.
      - name: Checkout repository
        uses: actions/checkout@v4

      # Download artifacts generated by the test job.
      - name: Download build artifacts
        uses: actions/download-artifact@v4
        with:
          name: build-22
          path: dist/

      # Deploy the application.
      - name: Deploy application
        id: deploy-step
        env:
          DEPLOY_TOKEN: ${{ secrets.DEPLOY_TOKEN }}
        run: |
          echo "Deploying application..."
          echo "deployment-url=https://example.com" >> "$GITHUB_OUTPUT"

      # Run only if all previous steps succeeded.
      - name: Deployment completed
        if: success()
        run: echo "Deployment successful!"

      # Run only if a previous step failed.
      - name: Handle deployment failure
        if: failure()
        run: echo "Deployment failed!"


# ============================================================
# 8. WORKFLOW-LEVEL OUTPUTS AND REUSABILITY
# ============================================================
# Workflow outputs are normally exposed through reusable workflows
# or job outputs. See the advanced examples below.
