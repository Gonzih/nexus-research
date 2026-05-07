# Migration Note

The `_research/` folder is intended to hold research files migrated from `Gonzih/money-brain`.

**Status: Pending** — The source repository is private and returned 404 during automated migration. To complete the migration, run the following manually after granting repo access:

```bash
# List available files
curl -H "Authorization: token $GITHUB_TOKEN" \
  "https://api.github.com/repos/Gonzih/money-brain/contents/_research" \
  | python3 -c "import sys,json; [print(f['name']) for f in json.load(sys.stdin) if f['type']=='file']"

# Fetch each file
curl -H "Authorization: token $GITHUB_TOKEN" \
  "https://raw.githubusercontent.com/Gonzih/money-brain/main/_research/FILENAME" \
  > _research/FILENAME
```
