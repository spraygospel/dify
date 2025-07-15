# Requirements Refine Command

Refine and iterate on existing requirements specifications through interactive conversation.

## Usage:
```bash
/requirements-refine [requirement-name] [mode]
```

## Instructions:

### 1. Initial Setup
1. Read requirements/.current-requirement to check for active requirement
2. If active requirement exists, warn user and ask if they want to end it first
3. Parse command arguments:
   - requirement-name: folder name in requirements/
   - mode: --interactive (default), --targeted, --feedback, --expand

### 2. Requirement Validation
1. Check if requirement exists in requirements/ directory
2. If not found, list available requirements and exit
3. Load existing requirement files and metadata
4. Determine current version number for new refinement

### 3. Mode Selection and Execution

#### Interactive Mode (Default)
```bash
/requirements-refine user-auth
/requirements-refine user-auth --interactive
```

**Process:**
1. Display current requirement overview
2. Show all available sections to refine:
   - Functional Requirements
   - Technical Requirements
   - API Endpoints
   - Database Schema
   - Security Considerations
   - Acceptance Criteria
   - Implementation Notes
   - Add new section
3. Let user choose section or ask open questions
4. Engage in conversational refinement (NOT yes/no questions)
5. Allow user to move between sections freely
6. Save when user indicates they're done

#### Targeted Mode
```bash
/requirements-refine user-auth --targeted security
/requirements-refine user-auth --targeted api-design
```

**Process:**
1. Extract target area from command or ask user
2. Focus conversation specifically on that area
3. Show current content of target section
4. Ask: "What specific aspects of [target] need refinement?"
5. Engage in deep conversation about that area only
6. Save targeted improvements

#### Feedback Mode
```bash
/requirements-refine user-auth --feedback "JWT implementation had issues with refresh tokens"
```

**Process:**
1. Extract feedback from command or ask user for implementation feedback
2. Analyze feedback against current requirements
3. Identify which sections need updates based on feedback
4. Ask clarifying questions about the feedback
5. Propose specific changes to address the feedback
6. Update relevant sections with improvements

#### Expand Mode
```bash
/requirements-refine user-auth --expand api-design
```

**Process:**
1. Extract section to expand or ask user
2. Show current content of target section
3. Ask: "What additional details should be added to [section]?"
4. Focus on adding more depth, examples, edge cases
5. Suggest areas that typically need more detail
6. Expand the section with comprehensive additions

### 4. File Management and Versioning

**Version Detection:**
- Check for existing versions: 06-requirements-spec.md, 06-requirements-spec-v2.md, etc.
- Determine next version number (v2, v3, etc.)

**File Creation:**
- Create new version: `06-requirements-spec-v[X].md`
- Update metadata.json with new version info
- Create or update refinement-log.md with changes

**Refinement Log Format:**
```markdown
# Refinement Log: [Requirement Name]

## Version 2 - [Date]
**Mode:** Interactive
**Changes:**
- Enhanced security section with token refresh details
- Added API response format specifications
- Expanded error handling scenarios

## Version 3 - [Date]
**Mode:** Feedback
**Feedback:** "JWT implementation had issues with refresh tokens"
**Changes:**
- Updated JWT token management approach
- Added refresh token rotation strategy
- Enhanced session management requirements
```

### 5. Conversation Flow

**Start Template:**
```
🔄 Refining Requirements: [requirement-name]

Current Version: v[X]
Mode: [selected-mode]

📋 Current Specification Overview:
[Brief summary of current spec]

📝 What would you like to refine?
[Mode-specific guidance]

Let's start the conversation - what specific aspects need improvement?
```

**During Refinement:**
- Maintain conversational tone
- Ask follow-up questions naturally
- Suggest improvements based on best practices
- Reference existing code patterns when relevant
- Allow user to change direction freely

**End Template:**
```
✅ Refinement Complete

📄 Created: 06-requirements-spec-v[X].md
📝 Updated: refinement-log.md
🔄 Changes: [summary of key changes]

Your refined requirements are ready for implementation!
```

### 6. Error Handling

**Common Scenarios:**
- Requirement not found: List available requirements
- No specifications to refine: Suggest running requirements-start first
- Invalid mode: Show available modes
- File permission issues: Clear error message
- Incomplete refinement: Offer to save progress

### 7. Integration Points

**Before Starting:**
- Check for active requirements gathering
- Validate requirement exists and has completed specs

**After Completion:**
- Update requirements/index.md with new version
- Clear any temporary files
- Maintain compatibility with existing commands

## Important Notes:

1. **Conversational Style:** This is NOT a yes/no question system. Engage in natural conversation.

2. **Flexibility:** Users can change topics, ask questions, or refine multiple sections in one session.

3. **Preservation:** Always create new versions, never overwrite existing specifications.

4. **Context Awareness:** Reference existing codebase patterns and constraints during refinement.

5. **Quality Focus:** Suggest improvements based on software engineering best practices.

6. **User Control:** Let user drive the conversation and decide when they're satisfied.