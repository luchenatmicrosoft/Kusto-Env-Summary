# Plan: Create `kusto-env-summary` Skill

## Context

When investigating Power Platform / Dataverse environment issues, engineers need to quickly look up an environment's deployment metadata (island, region, cluster, state, org/tenant IDs). Today this requires manually pasting a ~120-line cross-cluster KQL query into Kusto Explorer and substituting the environment GUID. This skill automates that into a single command, saves the result as a referenceable MD file, and makes the values available for the rest of the Claude Code session.

## Skill File Structure

```
C:\Users\chenlu\.claude\skills\kusto-env-summary\
  SKILL.md              <- Frontmatter + workflow instructions
  references\
    query.kql           <- Full KQL query template with placeholder
```

No `scripts/` or `assets/` needed — Claude natively handles read, substitute, execute-via-MCP, format, and write.

## Implementation Steps

### Step 1: Scaffold the skill directory

Run `init_skill.py` to create the skeleton:
```bash
python "C:/Users/chenlu/.claude/skills/skill-creator/scripts/init_skill.py" kusto-env-summary --path "C:/Users/chenlu/.claude/skills"
```

Then delete unneeded generated files (`scripts/`, `assets/`, `references/api_reference.md`).

### Step 2: Create `references/query.kql`

Store the user's exact KQL query with one change: replace the hardcoded environment ID line:
```
let _environmentId = 'ca056bae-6c2e-ed8b-bfb0-39abca1c64f2';
```
with a placeholder:
```
let _environmentId = '__ENVIRONMENT_ID_PLACEHOLDER__';
```

The rest of the query (CoreServiceKustoMetadata, LogicAppClusterMetadata, SearchData function, and the 25-cluster union) remains verbatim as provided by the user.

### Step 3: Write `SKILL.md`

**Frontmatter:**
- `name`: `kusto-env-summary`
- `description`: Describes what the skill does + trigger phrases (`env summary`, `environment summary`, `kusto env summary`, `get env info`, `look up environment`, or when user provides an environment GUID and wants deployment details)

**Body — Workflow:**
1. Parse and validate environment GUID from user input (8-4-4-4-12 hex pattern)
2. Read `references/query.kql` from the skill directory
3. Replace `__ENVIRONMENT_ID_PLACEHOLDER__` with the user's GUID (in-memory only)
4. Execute via `mcp__azure__kusto` tool with:
   - cluster: `https://fdislandsquery.centralus.kusto.windows.net`
   - database: `CAPAnalytics`
   - tenant: `72f988bf-86f1-41af-91ab-2d7cd011db47`
5. Format results as markdown table
6. Save to `{environment-GUID}-env-summary.md` in current working directory
7. Display full results in console
8. Declare "Env Summary values:" listing each column = value for session reference

**Output file format** (single-row — the common case):
```markdown
# Environment Summary

**Environment ID:** `{GUID}`
**Query Date:** {YYYY-MM-DD}

## Results

| Column | Value |
|--------|-------|
| resourceKusto | ... |
| resourceStoreLocation | ... |
| env_time | ... |
| resourceState | ... |
| friendlyName | ... |
| islandCategory | ... |
| location | ... |
| islandGeoShortName | ... |
| azureRegion | ... |
| islandName | ... |
| islandNumber | ... |
| organizationId | ... |
| tenantId | ... |
```

Multi-row: fallback to standard columnar markdown table.

**Session reference declaration** (output after the table):
```
Env Summary values:
- resourceKusto = {value}
- resourceStoreLocation = {value}
- ...each column...
```

This enables later prompts like "what is the Env Summary resourceState value?" to be answered from context.

**Error handling:**
- No results → report GUID not found
- Auth failure → remind user to authenticate to Microsoft corp tenant
- Timeout → retry with longer timeout (180s)

### Step 4: Validate

```bash
python "C:/Users/chenlu/.claude/skills/skill-creator/scripts/quick_validate.py" "C:/Users/chenlu/.claude/skills/kusto-env-summary"
```

### Step 5: Test

Invoke `/kusto-env-summary ca056bae-6c2e-ed8b-bfb0-39abca1c64f2` and verify:
- Query executes successfully via Azure MCP Kusto
- Results display in console
- `.md` file created in current working directory
- Session reference values are declared
- Later prompt "what is the Env Summary islandName value?" works from context

## Critical Files

| File | Action |
|------|--------|
| `C:\Users\chenlu\.claude\skills\kusto-env-summary\SKILL.md` | Create |
| `C:\Users\chenlu\.claude\skills\kusto-env-summary\references\query.kql` | Create |
| `C:\Users\chenlu\.claude\skills\skill-creator\scripts\init_skill.py` | Use (read-only) |
| `C:\Users\chenlu\.claude\skills\skill-creator\scripts\quick_validate.py` | Use (read-only) |
