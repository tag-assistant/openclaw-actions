# HEARTBEAT.md — GitHub Actions Agent

## On Each Heartbeat
1. Report chain number and remaining time
2. Check repository issues and PRs (if github skill available)
3. Run any pending tasks from the task queue
4. Log activity to memory/

## Tasks
- Monitor this repository for issues
- Keep workspace memory updated
- Report status via commit messages or issue comments
