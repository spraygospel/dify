# Execute Requirements Command

Execute implementation of completed requirements specifications.

## Usage:
```bash
/execute-requirements [requirement-name] [options]
```

## Options:
- `--mode`: `implementation` (default), `planning`, `partial`
- `--sections`: Comma-separated list of TRs to implement (e.g., TR1,TR2,TR3)
- `--dry-run`: Show what would be done without making changes
- `--no-backup`: Skip creating backup files
- `--commit`: Create git commits after each section

## Instructions:

### 1. Initial Validation
1. Check if requirement exists in requirements/ directory
2. Verify requirement status is "complete"
3. Look for latest version: 06-requirements-spec.md, 06-requirements-spec-v2.md, etc.
4. Load the latest requirements specification
5. Parse technical requirements (TR1, TR2, etc.)

### 2. Pre-Implementation Checklist
1. Verify all dependencies are available:
   - Required files exist
   - Database connection available
   - Required services running
2. Check for conflicting work:
   - No uncommitted changes in target files
   - No other active implementations
3. Create implementation directory structure
4. Initialize implementation-log.md

### 3. Implementation Modes

#### Implementation Mode (Default)
```bash
/execute-requirements llm-provider-config
```

**Process:**
1. Generate implementation plan from requirements
2. Create TodoWrite items for each TR
3. Execute each technical requirement in order:
   - Create/modify files as specified
   - Run tests after each section
   - Update progress in real-time
4. Handle dependencies between TRs
5. Complete with summary report

#### Planning Mode
```bash
/execute-requirements llm-provider-config --mode planning
```

**Process:**
1. Analyze all technical requirements
2. Generate detailed task breakdown
3. Identify dependencies
4. Estimate complexity/time
5. Output implementation-plan.md without executing

#### Partial Mode
```bash
/execute-requirements llm-provider-config --mode partial --sections TR1,TR2
```

**Process:**
1. Implement only specified sections
2. Verify section dependencies
3. Maintain system compatibility
4. Update logs for partial completion

### 4. Implementation Workflow

**For each Technical Requirement (TR):**

1. **Pre-Implementation:**
   - Read current state of target files
   - Create backups if needed
   - Validate preconditions

2. **Implementation:**
   - Create new files with proper headers
   - Modify existing files preserving structure
   - Follow patterns from contextFiles in metadata
   - Apply security best practices

3. **Post-Implementation:**
   - Run applicable tests/linters
   - Verify changes work correctly
   - Update implementation-log.md
   - Commit if --commit flag set

### 5. Progress Tracking

**TodoWrite Integration:**
- Create todo for each TR
- Mark as in_progress when starting
- Mark as completed when done
- Track blockers or issues

**Implementation Log Format:**
```markdown
# Implementation Log: [requirement-name]

## Status: IN_PROGRESS
## Started: [timestamp]
## Mode: [implementation|planning|partial]

## Progress:
### TR1: Database Schema
- Status: COMPLETE
- Started: [timestamp]
- Completed: [timestamp]
- Files Created:
  - migrations/20240110103000_create_llm_configurations.js
- Tests: PASSED
- Notes: Migration runs successfully

### TR2: GraphQL Schema Extension
- Status: IN_PROGRESS
- Started: [timestamp]
- Files Modified:
  - src/apollo/server/schema.ts (backed up)
- Current Step: Adding resolver types

## Issues:
- None

## Rollback Plan:
- Restore from backups/ directory
- Run migration rollback if needed
```

### 6. File Operations

**File Creation Pattern:**
```typescript
// For new files
1. Check if file already exists
2. If exists, create backup: backups/[timestamp]-[filename]
3. Create file with proper headers/imports
4. Add implementation following requirements
5. Format with appropriate formatter

// For existing files
1. Read current content
2. Create backup
3. Find insertion points
4. Add new code maintaining style
5. Preserve existing functionality
```

### 7. Testing Strategy

**After Each Section:**
1. **For Database (TR1):**
   - Run migration up
   - Verify table creation
   - Test migration down

2. **For GraphQL (TR2):**
   - Check schema compilation
   - Verify resolver types

3. **For Backend (TR3-5):**
   - Run unit tests
   - Check TypeScript compilation
   - Verify imports

4. **For UI (TR6):**
   - Check component compilation
   - Verify props/state types
   - Test basic rendering

5. **For Integration (TR7):**
   - Test config loading
   - Verify fallback behavior

### 8. Error Handling

**On Error:**
1. Log error details in implementation-log.md
2. Mark current TR as FAILED
3. Stop execution
4. Display error context
5. Suggest fixes:
   - Missing dependencies
   - File permissions
   - Syntax errors
   - Failed tests

**Recovery Options:**
- Resume from failed TR
- Skip failed TR (if non-critical)
- Rollback and retry
- Manual intervention

### 9. Completion

**Success Completion:**
1. Mark all TRs as COMPLETE
2. Generate implementation-summary.md:
   ```markdown
   # Implementation Summary
   
   ## Requirement: [name]
   ## Status: COMPLETE
   ## Duration: [time]
   
   ## Files Created (X):
   - [list of new files]
   
   ## Files Modified (Y):
   - [list of modified files]
   
   ## Next Steps:
   1. Run full test suite
   2. Test in development environment
   3. Create PR if needed
   4. Deploy to staging
   ```

3. Clean up temporary files
4. Suggest testing commands

### 10. Special Considerations

**For LLM Provider Config:**
- Ensure encryption setup before API key storage
- Test database connections
- Verify GraphQL schema changes
- Check UI component integration

**For Database Migrations:**
- Always test rollback
- Check for data loss scenarios
- Verify indexes created

**For UI Components:**
- Follow existing component patterns
- Maintain consistent styling
- Add proper TypeScript types

## Command Examples:

```bash
# Full implementation
/execute-requirements llm-provider-config

# Just planning
/execute-requirements llm-provider-config --mode planning

# Only database and GraphQL
/execute-requirements llm-provider-config --mode partial --sections TR1,TR2

# Dry run to see what would happen
/execute-requirements llm-provider-config --dry-run

# Implementation with git commits
/execute-requirements llm-provider-config --commit
```

## Important Rules:

1. **Always backup before modifying**
2. **Test incrementally**
3. **Follow existing patterns**
4. **Document decisions**
5. **Handle errors gracefully**
6. **Maintain backwards compatibility**
7. **Security first for sensitive data**

## File Structure:

```
requirements/[requirement-name]/
├── 06-requirements-spec-v2.md     # Source requirements
├── implementation-plan.md         # Generated plan
├── implementation-log.md          # Progress tracking
├── implementation-summary.md      # Final summary
└── backups/                       # Backup files
    ├── 20240110-103000-schema.ts.bak
    └── 20240110-103100-settings.ts.bak
```