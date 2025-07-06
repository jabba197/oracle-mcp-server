# Plan: Adding View Support for Read-Only Analysts

## Overview
This plan outlines the implementation strategy for adding Oracle view support to the MCP server while maintaining compatibility with read-only analyst permissions.

## Current Situation
- System only queries `ALL_TABLES` (requires broad permissions)
- Views are completely invisible to schema discovery
- Uses `UTL_MATCH` for similarity (requires EXECUTE privilege)
- Assumes users have access to `ALL_*` dictionary views

## Constraints
- Users have minimal Oracle permissions (SELECT only on specific objects)
- May not have access to `ALL_TABLES`, `ALL_VIEWS`, `ALL_OBJECTS`
- Cannot create database objects
- May not have EXECUTE privilege on `UTL_MATCH`
- Must maintain backward compatibility

## Implementation Strategy

### Phase 1: Permission-Aware Discovery (Priority: HIGH)

#### 1.1 Create Tiered Metadata Discovery
**File**: `db_context/database.py`

```python
async def get_all_table_names(self) -> Set[str]:
    """
    Try multiple approaches based on available permissions:
    1. ALL_TABLES + ALL_VIEWS (if accessible)
    2. USER_TABLES + USER_VIEWS (fallback)
    3. Return empty set with warning (final fallback)
    """
```

**Implementation Steps**:
1. Wrap existing query in try/except for ORA-01031 (insufficient privileges)
2. Add fallback to USER_TABLES/USER_VIEWS
3. Log permission issues clearly
4. Return combined results from whatever views are accessible

#### 1.2 Add Graceful Permission Handling
**File**: `db_context/database.py`

- Add `_try_query()` helper method that catches permission errors
- Return None or empty results instead of crashing
- Log which dictionary views failed

### Phase 2: View Support Implementation (Priority: HIGH)

#### 2.1 Modify Core Queries
**Location**: `db_context/database.py`

1. **get_all_table_names()**
   ```sql
   -- Try this first:
   SELECT object_name FROM all_objects 
   WHERE owner = :owner AND object_type IN ('TABLE', 'VIEW')
   
   -- If fails, try:
   SELECT table_name FROM user_tables
   UNION ALL
   SELECT view_name FROM user_views
   ```

2. **load_table_details()**
   - Change existence check to use ALL_OBJECTS or USER_OBJECTS
   - Skip relationship queries for views (they have no FK constraints)
   - Add object_type detection

3. **search_in_database()**
   - Include views in search results
   - Remove UTL_MATCH dependency (see Phase 3)

#### 2.2 Update Data Model
**File**: `db_context/models.py`

Add minimal changes to TableInfo:
```python
class TableInfo:
    # ... existing fields ...
    object_type: str = "TABLE"  # New field with default
    
    def format_schema(self):
        # Add VIEW/TABLE indicator to output
        header = f"{self.object_type}: {self.table_name}"
```

### Phase 3: Remove Permission Dependencies (Priority: HIGH)

#### 3.1 Replace UTL_MATCH with Client-Side Logic
**File**: `db_context/database.py`

1. Remove `UTL_MATCH.EDIT_DISTANCE_SIMILARITY` from SQL
2. Fetch all names and filter in Python:
   ```python
   # Use difflib for similarity matching
   from difflib import SequenceMatcher
   
   def calculate_similarity(s1: str, s2: str) -> float:
       return SequenceMatcher(None, s1.upper(), s2.upper()).ratio()
   ```

3. Apply filtering after fetching results

### Phase 4: Configuration Support (Priority: MEDIUM)

#### 4.1 Add Permission Mode Configuration
**File**: `.env` or environment variables

```bash
# Permission mode: "full" (ALL_* views) or "limited" (USER_* views)
ORACLE_PERMISSION_MODE=limited

# Known schemas (for limited mode)
ORACLE_KNOWN_SCHEMAS=HR,SALES,DW

# Disable similarity search if needed
ORACLE_ENABLE_FUZZY_SEARCH=false
```

#### 4.2 Schema List Configuration
**File**: `config/schemas.json` (optional)

For environments where discovery is impossible:
```json
{
  "schemas": {
    "HR": ["EMPLOYEES", "DEPARTMENTS", "VW_EMPLOYEE_HIERARCHY"],
    "SALES": ["ORDERS", "CUSTOMERS", "VW_SALES_PERFORMANCE"]
  }
}
```

### Phase 5: Error Handling & User Feedback (Priority: HIGH)

#### 5.1 Improve Error Messages
**All files**

Replace generic errors with specific guidance:
- "Cannot access ALL_TABLES. Using USER_TABLES instead."
- "View support requires ALL_VIEWS access. Currently showing only tables."
- "Fuzzy search disabled due to missing permissions."

#### 5.2 Add Permission Check Tool
**File**: `main.py`

New MCP tool to help users understand their permissions:
```python
@mcp.tool()
async def check_permissions(ctx: Context) -> str:
    """Check which Oracle dictionary views are accessible"""
```

### Phase 6: Testing Strategy (Priority: HIGH)

#### 6.1 Create Test Users
```sql
-- Minimal permission user
CREATE USER analyst_readonly IDENTIFIED BY password;
GRANT CREATE SESSION TO analyst_readonly;
GRANT SELECT ON hr.employees TO analyst_readonly;
GRANT SELECT ON hr.vw_employee_hierarchy TO analyst_readonly;
-- Note: NO grants on ALL_TABLES, ALL_VIEWS, etc.
```

#### 6.2 Test Scenarios
1. Full permissions (existing tests should pass)
2. Limited permissions (USER_* views only)
3. No dictionary access (manual schema list)
4. Mixed access (some ALL_* views work, others don't)

### Phase 7: Documentation Updates (Priority: MEDIUM)

1. Update README.md with permission requirements
2. Add troubleshooting section for permission issues
3. Document configuration options
4. Update CLAUDE.md with new patterns

## Implementation Order

1. **Week 1**: Phase 3.1 (Remove UTL_MATCH) + Phase 1 (Permission handling)
2. **Week 1**: Phase 2.1 (Core query changes)
3. **Week 2**: Phase 5 (Error handling) + Phase 6 (Testing)
4. **Week 2**: Phase 4 (Configuration) + Phase 7 (Documentation)

## Success Criteria

1. Views are discoverable when permissions allow
2. System works with only USER_* view access
3. Clear error messages guide users on permission issues
4. No crashes due to permission errors
5. Fuzzy search works without database functions
6. Existing functionality remains intact

## Rollback Plan

If issues arise:
1. Environment variable to disable view support: `ORACLE_INCLUDE_VIEWS=false`
2. Revert to table-only queries
3. All changes are backward compatible

## Future Enhancements

1. View definition parsing to infer pseudo-relationships
2. Automatic permission level detection on startup
3. Caching of permission checks to avoid repeated failures
4. Support for materialized views
5. Integration with Oracle privilege analysis tools

## Code Examples

### Example: Permission-Aware Query
```python
async def _execute_with_fallback(self, all_query: str, user_query: str, 
                                 params: dict) -> List[tuple]:
    """Execute query with ALL_* views, fallback to USER_* if needed"""
    conn = await self.get_connection()
    cursor = conn.cursor()
    
    # Try ALL_* views first
    try:
        return await self._execute_cursor(cursor, all_query, **params)
    except oracledb.DatabaseError as e:
        if e.args[0].code == 1031:  # ORA-01031: insufficient privileges
            print(f"Permission denied for ALL_* views, trying USER_* views", 
                  file=sys.stderr)
            # Try USER_* views
            try:
                return await self._execute_cursor(cursor, user_query, **params)
            except oracledb.DatabaseError as e2:
                if e2.args[0].code == 1031:
                    print(f"No access to dictionary views", file=sys.stderr)
                    return []
                raise
        raise
```

### Example: Client-Side Similarity
```python
def search_with_similarity(items: List[str], search_term: str, 
                          threshold: float = 0.65) -> List[str]:
    """Client-side fuzzy matching to replace UTL_MATCH"""
    results = []
    search_upper = search_term.upper()
    
    for item in items:
        # Direct substring match first
        if search_upper in item.upper():
            results.append((item, 1.0))
        else:
            # Calculate similarity
            similarity = SequenceMatcher(None, item.upper(), 
                                       search_upper).ratio()
            if similarity >= threshold:
                results.append((item, similarity))
    
    # Sort by similarity descending
    results.sort(key=lambda x: x[1], reverse=True)
    return [item for item, _ in results]
```

## Notes

- This plan prioritizes maintaining functionality over feature completeness
- The tiered approach ensures something works even with minimal permissions
- Client-side processing replaces database functions where needed
- Configuration options provide escape hatches for difficult environments