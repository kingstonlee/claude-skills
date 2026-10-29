# claude-skills

A small collection of [Claude Code](https://claude.com/claude-code) skills for backend/infra diagnosis and AWS operations. Each skill is a self-contained `SKILL.md` that teaches Claude a repeatable procedure for a specific, recurring problem.

## Skills

| Skill | Use it when |
|-------|-------------|
| [`asyncio-blocking-diagnosis`](skills/asyncio-blocking-diagnosis/SKILL.md) | A Python asyncio/uvicorn service runs concurrent jobs sequentially, or requests stall for the duration of another request — a sync call is holding the event loop. |
| [`aws-resource-decommission`](skills/aws-resource-decommission/SKILL.md) | Verifying an AWS resource (Elastic IP, stopped EC2, EBS volume) is safe to release or terminate, using CloudTrail activity as the signal. |
| [`rds-snapshot-restore`](skills/rds-snapshot-restore/SKILL.md) | Restoring an RDS snapshot to a temporary instance for data investigation or backfill — including the MySQL 9.x client `ERROR 2059` workaround. |

## Installing

Clone into your Claude Code skills directory (or symlink individual skills):

```bash
git clone https://github.com/kingstonlee/claude-skills.git
cp -r claude-skills/skills/* ~/.claude/skills/
```

Claude Code discovers any directory under `~/.claude/skills/` containing a `SKILL.md` with a `name` and `description` in its frontmatter.

## Notes

These skills describe generic procedures. Anywhere a value is environment-specific — regions, subnet groups, security groups, credentials — it is written as a `<placeholder>`. Fill in your own values; never commit real infrastructure identifiers or secrets.

## License

[MIT](LICENSE)
