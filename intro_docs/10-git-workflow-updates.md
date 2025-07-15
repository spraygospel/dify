# Git Workflow untuk Tetap Update dengan Official Dify

Panduan untuk maintain customization sambil tetap bisa update dari official Dify repository.

## Setup Initial Repository

### 1. Fork Strategy (Recommended)

```bash
# 1. Fork di GitHub UI dulu, lalu clone fork Anda
git clone https://github.com/YOUR_USERNAME/dify.git
cd dify

# 2. Add official dify sebagai upstream
git remote add upstream https://github.com/langgenius/dify.git
git remote -v  # Verify remotes

# 3. Create branch untuk customization
git checkout -b my-customizations
```

### 2. Local-Only Strategy

```bash
# Clone official
git clone https://github.com/langgenius/dify.git
cd dify

# Create branch untuk custom work
git checkout -b my-customizations

# Setup upstream
git remote rename origin upstream
```

## Workflow untuk Update

### 1. Sync dengan Official Dify

```bash
# 1. Fetch latest dari official
git fetch upstream

# 2. Checkout ke main/master branch
git checkout main

# 3. Merge updates dari official
git merge upstream/main

# 4. Push ke fork Anda (jika pakai fork strategy)
git push origin main
```

### 2. Update Custom Branch

```bash
# 1. Checkout ke custom branch
git checkout my-customizations

# 2. Rebase atau merge dari main
# Option A: Rebase (cleaner history)
git rebase main

# Option B: Merge (safer, easier conflict resolution)
git merge main

# 3. Resolve conflicts jika ada
# Edit conflicted files
git add .
git rebase --continue  # atau git commit jika merge
```

## Best Practices untuk Customization

### 1. Isolated Custom Code

```bash
# Structure yang direkomendasikan
dify/
├── api/
│   └── core/
│       └── tools/
│           └── builtin_tool/
│               └── providers/
│                   └── custom/        # ✅ Your custom tools here
│                       ├── my_tool1/
│                       └── my_tool2/
├── custom/                           # ✅ Additional custom code
│   ├── extensions/
│   ├── scripts/
│   └── configs/
└── intro_docs/                       # ✅ Your documentation
```

### 2. Git Ignore untuk Custom Files

```bash
# Tambahkan ke .gitignore
# Custom development files
custom/configs/local.env
custom/data/
*.custom.yaml
.my-settings/

# Tapi JANGAN ignore custom tools
!api/core/tools/builtin_tool/providers/custom/
```

### 3. Commit Strategy

```bash
# Separate commits untuk different concerns
git add api/core/tools/builtin_tool/providers/custom/
git commit -m "feat(tools): add custom weather tool"

git add intro_docs/
git commit -m "docs: add Indonesian documentation"

# Tag stable versions
git tag -a v1.0-custom -m "Stable custom version with weather tool"
```

## Handling Conflicts

### 1. Common Conflict Areas

```bash
# Areas yang sering conflict:
- docker-compose.yaml     # Jika modify services
- .env.example           # Jika add new env vars  
- api/requirements.txt   # Jika add dependencies
- web/package.json      # Jika add frontend deps
```

### 2. Conflict Resolution Strategy

```bash
# 1. Lihat files yang conflict
git status

# 2. Untuk file config, biasanya keep both
# Edit file, cari markers:
# <<<<<<< HEAD (your version)
# =======  
# >>>>>>> upstream/main (official version)

# 3. Test after resolution
docker compose down
docker compose up -d

# 4. Complete merge/rebase
git add resolved_file.txt
git rebase --continue
```

## Automated Update Script

```bash
#!/bin/bash
# scripts/update-from-upstream.sh

echo "🔄 Updating from official Dify..."

# Stash local changes
git stash save "Auto-stash before update"

# Checkout main and update
git checkout main
git fetch upstream
git merge upstream/main

# Update custom branch
git checkout my-customizations
git rebase main

# Check for conflicts
if [ $? -ne 0 ]; then
    echo "❌ Conflicts detected! Please resolve manually."
    echo "After resolving, run: git rebase --continue"
    exit 1
fi

# Restore stashed changes
git stash pop

echo "✅ Update complete!"

# Run tests
echo "🧪 Running tests..."
cd api && uv run pytest tests/unit_tests/tools/

# Restart services
echo "🔄 Restarting services..."
docker compose restart api worker

echo "🎉 All done! Your Dify is up to date."
```

## CI/CD Integration

```yaml
# .github/workflows/sync-upstream.yml
name: Sync with Upstream

on:
  schedule:
    - cron: '0 0 * * 0'  # Weekly
  workflow_dispatch:

jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 0
          
      - name: Sync upstream
        run: |
          git config user.name github-actions
          git config user.email github-actions@github.com
          
          git remote add upstream https://github.com/langgenius/dify.git
          git fetch upstream
          
          git checkout main
          git merge upstream/main
          
          git push origin main
          
      - name: Create PR for custom branch
        uses: peter-evans/create-pull-request@v5
        with:
          branch: update-from-upstream
          title: 'Update from official Dify'
          body: |
            Automated update from official Dify repository.
            Please review changes and test before merging.
```

## Rollback Strategy

```bash
# Jika update break something
# 1. Check previous tags
git tag -l

# 2. Rollback to previous stable
git checkout v1.0-custom

# 3. Create hotfix branch
git checkout -b hotfix/fix-update-issue

# 4. Fix and test
# ... make fixes ...

# 5. Merge back
git checkout my-customizations
git merge hotfix/fix-update-issue
```

## Tips untuk Smooth Updates

### 1. Pre-Update Checklist
- ✅ Backup database
- ✅ Commit all changes  
- ✅ Tag current version
- ✅ Run tests
- ✅ Document custom changes

### 2. Post-Update Checklist
- ✅ Check migration files
- ✅ Run database migrations
- ✅ Test all custom tools
- ✅ Verify API endpoints
- ✅ Check frontend builds

### 3. Keep Custom Changes Minimal
```python
# ❌ AVOID: Modifying core files
# api/core/tools/tool_manager.py

# ✅ BETTER: Extend via inheritance
# api/core/tools/builtin_tool/providers/custom/my_tool.py
from core.tools.builtin_tool.builtin_tool import BuiltinTool

class MyCustomTool(BuiltinTool):
    # Your implementation
    pass
```

## Emergency Contacts

Jika ada major breaking changes:
- GitHub Issues: https://github.com/langgenius/dify/issues
- Discord: https://discord.gg/FngNHpbcY7
- Check CHANGELOG.md untuk breaking changes

---

> 💡 **Key Point**: Dengan workflow ini, Anda bisa maintain customization sambil tetap mendapat security updates, bug fixes, dan features baru dari official Dify. Always test di staging environment dulu sebelum update production!