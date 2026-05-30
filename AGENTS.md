# Agent Guidelines

## Deployment

After pushing to main, always wait for the GitHub Actions deploy workflow to succeed before reporting completion. Check with `gh run list` and if it fails, inspect logs with `gh run view <id> --log-failed`, fix the issue, and push again until the deploy succeeds.
