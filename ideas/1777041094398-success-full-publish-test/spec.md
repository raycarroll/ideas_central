# SUCCESS: Full Publish Test

## Summary
Complete end-to-end publish workflow test

## Goals
- Create git commit locally
- Push to ideas_central on GitHub
- Register metadata in PostgreSQL
- Complete atomic transaction

## Requirements
- All steps succeed
- Rollback works if any step fails
- Idea appears in both GitHub and database