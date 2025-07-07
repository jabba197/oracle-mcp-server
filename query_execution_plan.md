# Query Execution Implementation Plan

## Overview
Add secure, performant query execution capability to the Oracle MCP server for AI-driven database interactions.

## Security Architecture

### 1. Multi-Layer Query Validation
```python
# query_analyzer.py - New module for SQL validation
import sqlparse
import re
from typing import Set, Tuple

class QueryAnalyzer:
    """Analyzes SQL queries for security and performance implications"""
    
    VOLATILE_FUNCTIONS = {
        'SYSDATE', 'CURRENT_TIMESTAMP', 'SYSTIMESTAMP', 
        'USER', 'UID', 'SYS_GUID', 'USERENV',
        'DBTIMEZONE', 'SESSIONTIMEZONE', 'LOCALTIMESTAMP'
    }
    
    DANGEROUS_PATTERNS = [
        r'DBMS_\w+',  # DBMS packages
        r'UTL_\w+',   # UTL packages
        r'EXECUTE\s+IMMEDIATE',  # Dynamic SQL
        r'PRAGMA\s+AUTONOMOUS_TRANSACTION',  # Autonomous transactions
    ]
    
    def __init__(self, query: str):
        self.query = query
        self.parsed = None
        
    def is_readonly_select(self) -> bool:
        """Verify query is a single SELECT statement"""
        try:
            statements = sqlparse.parse(self.query)
            
            # Only allow single statement
            if len(statements) != 1:
                return False
                
            # Check statement type
            statement = statements[0]
            if statement.get_type() != 'SELECT':
                return False
                
            # Additional checks for embedded DML
            query_upper = self.query.upper()
            for pattern in self.DANGEROUS_PATTERNS:
                if re.search(pattern, query_upper):
                    return False
                    
            return True
        except Exception:
            return False
    
    def contains_volatile_functions(self) -> bool:
        """Check if query contains volatile functions"""
        query_upper = self.query.upper()
        for func in self.VOLATILE_FUNCTIONS:
            if func in query_upper:
                return True
        return False
    
    def extract_table_names(self) -> Set[str]:
        """Extract table names for permission checking"""
        # Implementation would parse FROM clauses
        # For now, return empty set
        return set()
```

### 2. Pre-flight Cost Analysis
```python
# In DatabaseConnector
async def _get_query_cost(self, conn, sql: str, params: dict) -> Tuple[int, int]:
    """Get estimated cost and cardinality from EXPLAIN PLAN"""
    cursor = conn.cursor()
    try:
        # Generate unique statement ID
        stmt_id = f"MCP_{int(time.time() * 1000)}"
        
        # Run EXPLAIN PLAN
        explain_sql = f"EXPLAIN PLAN SET STATEMENT_ID = '{stmt_id}' FOR {sql}"
        await self._execute_cursor(cursor, explain_sql, **params)
        
        # Get cost and cardinality
        cost_query = """
        SELECT cost, cardinality 
        FROM plan_table 
        WHERE statement_id = :stmt_id 
        AND id = 0
        """
        result = await self._execute_cursor(cursor, cost_query, stmt_id=stmt_id)
        
        if result:
            return result[0][0] or 0, result[0][1] or 0
        return 0, 0
        
    finally:
        # Clean up plan table
        await self._execute_cursor(
            cursor, 
            "DELETE FROM plan_table WHERE statement_id = :stmt_id",
            stmt_id=stmt_id
        )
```

## Performance Implementation

### 1. Streaming Results with Memory Control
```python
# In DatabaseConnector
async def execute_select_query(
    self, 
    sql: str, 
    params: Optional[Dict[str, Any]] = None,
    row_limit: int = 1000, 
    timeout_sec: int = 30,
    max_cost: int = 1_000_000,
    max_cardinality: int = 1_000_000
) -> Dict[str, Any]:
    """
    Execute a SELECT query with comprehensive safety controls
    """
    start_time = time.monotonic()
    
    # 1. Security validation
    analyzer = QueryAnalyzer(sql)
    if not analyzer.is_readonly_select():
        raise ValueError("Only SELECT statements are allowed")
    
    # 2. Check if cacheable
    is_cacheable = not analyzer.contains_volatile_functions()
    cache_key = None
    if is_cacheable:
        cache_key = hashlib.sha256(
            f"{sql}:{json.dumps(params, sort_keys=True)}".encode()
        ).hexdigest()
        
        # Check cache
        cached = self._query_cache.get(cache_key)
        if cached:
            return cached
    
    conn = await self.get_connection()
    try:
        # 3. Cost analysis
        cost, cardinality = await self._get_query_cost(conn, sql, params or {})
        
        if cost > max_cost:
            raise ValueError(
                f"Query cost ({cost:,}) exceeds limit ({max_cost:,}). "
                "Please add WHERE clauses or simplify joins."
            )
            
        if cardinality > max_cardinality:
            raise ValueError(
                f"Query estimated to return {cardinality:,} rows, "
                f"exceeds limit ({max_cardinality:,}). Please add filters."
            )
        
        cursor = conn.cursor()
        cursor.arraysize = 200  # Optimal chunk size for network transfer
        
        # 4. Execute with timeout
        if hasattr(cursor, 'callTimeout'):
            cursor.callTimeout = timeout_sec * 1000  # milliseconds
            
        await self._execute_cursor(cursor, sql, **(params or {}))
        
        # 5. Get column metadata
        columns = []
        for desc in cursor.description:
            columns.append({
                "name": desc[0],
                "type": desc[1].__name__ if hasattr(desc[1], '__name__') else str(desc[1]),
                "size": desc[2],
                "precision": desc[4] if len(desc) > 4 else None,
                "scale": desc[5] if len(desc) > 5 else None,
                "nullable": desc[6] if len(desc) > 6 else True
            })
        
        # 6. Stream results
        rows = []
        truncated = False
        
        while len(rows) < row_limit:
            chunk_size = min(row_limit - len(rows), cursor.arraysize)
            chunk = await cursor.fetchmany(numRows=chunk_size)
            
            if not chunk:
                break
                
            for row in chunk:
                if len(rows) >= row_limit:
                    truncated = True
                    break
                rows.append(self._serialize_row(row, cursor.description))
        
        # Check if more rows exist
        if not truncated and len(rows) == row_limit:
            probe = await cursor.fetchone()
            if probe:
                truncated = True
        
        end_time = time.monotonic()
        
        result = {
            "status": "success",
            "truncated": truncated,
            "columns": columns,
            "rows": rows,
            "row_count": len(rows),
            "execution_time_ms": int((end_time - start_time) * 1000),
            "query_cost": cost,
            "estimated_rows": cardinality
        }
        
        # 7. Cache if appropriate
        if is_cacheable and cache_key:
            self._query_cache.set(cache_key, result, ttl=300)  # 5 min TTL
            
        return result
        
    except asyncio.TimeoutError:
        raise ValueError(f"Query exceeded timeout of {timeout_sec} seconds")
    except oracledb.DatabaseError as e:
        error_obj, = e.args
        return {
            "status": "error",
            "error_code": error_obj.code,
            "error_message": self._format_oracle_error(error_obj)
        }
    finally:
        await self._close_connection(conn)
```

### 2. Data Type Serialization
```python
def _serialize_row(self, row: Tuple, description: list) -> list:
    """Convert Oracle data types to JSON-serializable formats"""
    serialized = []
    
    for val, desc in zip(row, description):
        if val is None:
            serialized.append(None)
        elif isinstance(val, oracledb.LOB):
            # Handle CLOBs and BLOBs
            if desc[1] == oracledb.CLOB:
                try:
                    # Read up to 1MB of CLOB data
                    clob_data = val.read(1024 * 1024)
                    if len(clob_data) == 1024 * 1024:
                        serialized.append(clob_data + "... [truncated]")
                    else:
                        serialized.append(clob_data)
                except:
                    serialized.append("[CLOB - error reading]")
            else:
                # BLOB - just indicate presence
                serialized.append(f"[BLOB - {val.size()} bytes]")
        elif isinstance(val, datetime.datetime):
            serialized.append(val.isoformat())
        elif isinstance(val, datetime.date):
            serialized.append(val.isoformat())
        elif isinstance(val, decimal.Decimal):
            # Convert to string to preserve precision
            serialized.append(str(val))
        elif isinstance(val, (bytes, bytearray)):
            # RAW data types
            serialized.append(val.hex())
        else:
            serialized.append(val)
            
    return serialized
```

## Tool Implementation

### main.py Tool Definition
```python
@mcp.tool()
async def execute_query(
    query: str, 
    row_limit: Optional[int] = None,
    timeout_seconds: Optional[int] = None,
    ctx: Context
) -> str:
    """
    Execute a read-only SELECT query and return results as formatted data.
    
    This tool enables data retrieval from the Oracle database with comprehensive 
    safety controls. Only SELECT statements are permitted - any attempt to modify 
    data will be rejected.
    
    Security & Performance Controls:
    - SQL injection prevention through statement type validation
    - Automatic cost analysis prevents resource-intensive queries
    - Result set limiting prevents memory exhaustion  
    - Query timeout protection (default 30 seconds)
    - Streaming results for efficient memory usage
    
    Data Type Handling:
    - DATEs are returned in ISO format (YYYY-MM-DD HH:MM:SS)
    - NUMBERs preserve precision as strings for large values
    - CLOBs are read up to 1MB (larger ones are truncated)
    - BLOBs are indicated but not returned
    - NULLs are preserved as null in JSON
    
    Args:
        query: The SELECT statement to execute. Must be a single SELECT statement.
               Cannot contain DML (INSERT/UPDATE/DELETE) or DDL operations.
        row_limit: Maximum rows to return (default 1000, max 10000). 
                   If query returns more rows, results are truncated.
        timeout_seconds: Query timeout in seconds (default 30, max 300).
    
    Returns:
        JSON formatted result containing:
        - status: 'success' or 'error'
        - truncated: true if row limit was reached
        - columns: Array of column definitions with name, type, nullable
        - rows: Array of arrays containing row data
        - row_count: Number of rows returned
        - execution_time_ms: Query execution time
        - query_cost: Estimated query cost from optimizer
        - error_message: Detailed error if status is 'error'
    
    Example:
        query: "SELECT customer_id, customer_name FROM customers WHERE status = 'ACTIVE'"
        Returns: {
            "status": "success",
            "truncated": false,
            "columns": [
                {"name": "CUSTOMER_ID", "type": "NUMBER", "nullable": false},
                {"name": "CUSTOMER_NAME", "type": "VARCHAR2", "nullable": true}
            ],
            "rows": [
                [1001, "Acme Corp"],
                [1002, "TechStart Inc"]
            ],
            "row_count": 2,
            "execution_time_ms": 45
        }
    """
    db_context: DatabaseContext = ctx.request_context.lifespan_context
    
    # Apply defaults and limits
    row_limit = min(row_limit or 1000, 10000)  # Hard max of 10k
    timeout_seconds = min(timeout_seconds or 30, 300)  # Hard max of 5 min
    
    try:
        result = await db_context.db.execute_select_query(
            sql=query,
            row_limit=row_limit,
            timeout_sec=timeout_seconds,
            max_cost=int(os.getenv('QUERY_MAX_COST', '1000000')),
            max_cardinality=int(os.getenv('QUERY_MAX_CARDINALITY', '1000000'))
        )
        
        # Format for AI consumption
        if result.get("truncated"):
            result["warning"] = (
                f"Results truncated at {row_limit} rows. "
                "Add WHERE clauses or increase row_limit to see more data."
            )
            
        return json.dumps(result, indent=2, ensure_ascii=False)
        
    except ValueError as e:
        return json.dumps({
            "status": "error",
            "error_type": "validation_error", 
            "error_message": str(e),
            "suggestion": "Check your query syntax and ensure it's a SELECT statement"
        }, indent=2)
    except Exception as e:
        return json.dumps({
            "status": "error",
            "error_type": "execution_error",
            "error_message": str(e)
        }, indent=2)
```

## Configuration

### Environment Variables
```bash
# Query execution limits
QUERY_DEFAULT_ROW_LIMIT=1000      # Default rows returned
QUERY_MAX_ROW_LIMIT=10000         # Absolute maximum rows
QUERY_TIMEOUT_SECONDS=30          # Default timeout
QUERY_MAX_TIMEOUT_SECONDS=300     # Absolute max timeout

# Performance thresholds  
QUERY_MAX_COST=1000000           # Max allowed query cost
QUERY_MAX_CARDINALITY=1000000    # Max estimated rows

# Caching
QUERY_CACHE_ENABLED=true         # Enable result caching
QUERY_CACHE_TTL_SECONDS=300      # Cache time-to-live
QUERY_CACHE_MAX_SIZE_MB=100      # Max cache size

# Security
QUERY_ALLOW_SYSTEM_TABLES=false  # Block queries on SYS/SYSTEM objects
QUERY_LOG_EXECUTED=true          # Log all executed queries
```

## Error Handling

### Oracle Error Mapping
```python
def _format_oracle_error(self, error_obj) -> str:
    """Convert Oracle errors to user-friendly messages"""
    
    ERROR_MAPPINGS = {
        904: "Invalid column name. Check column exists in the table.",
        942: "Table or view does not exist. Use search_tables_schema to find correct name.",
        1017: "Invalid username/password",
        1031: "Insufficient privileges. You may not have SELECT permission on this object.",
        1422: "Query returns more than one row where single row expected",
        1476: "Divisor is equal to zero", 
        1722: "Invalid number format in query",
        1830: "Date format picture ends before converting entire string",
        1843: "Invalid month in date",
        12154: "TNS: could not resolve the connect identifier"
    }
    
    base_msg = f"ORA-{error_obj.code:05d}: {error_obj.message}"
    
    if error_obj.code in ERROR_MAPPINGS:
        return f"{base_msg}\nHint: {ERROR_MAPPINGS[error_obj.code]}"
    
    return base_msg
```

## Testing Strategy

### 1. Security Tests
- Verify rejection of INSERT, UPDATE, DELETE, CREATE, DROP
- Test SQL injection attempts
- Verify multi-statement queries are blocked
- Test queries with embedded PL/SQL

### 2. Performance Tests  
- Verify row limiting works correctly
- Test timeout functionality
- Verify cost threshold rejection
- Test memory usage with large results

### 3. Data Type Tests
- Test all Oracle data types
- Verify CLOB handling
- Test NULL handling
- Verify date/timestamp formatting

### 4. Permission Tests
- Test with minimal SELECT permissions
- Verify error messages for permission denied
- Test queries on SYS objects (should fail)

## Implementation Steps

1. Create `db_context/query_analyzer.py` module
2. Add `execute_select_query` method to `DatabaseConnector`
3. Add serialization helpers for Oracle data types
4. Implement caching layer (optional initially)
5. Add `execute_query` tool to `main.py`
6. Add configuration to `.env.example`
7. Create comprehensive test suite
8. Update documentation

## Usage Examples

### Simple Query
```python
result = await execute_query(
    "SELECT * FROM employees WHERE department_id = 10"
)
```

### Complex Query with Limits
```python
result = await execute_query(
    """
    SELECT 
        c.customer_name,
        COUNT(o.order_id) as order_count,
        SUM(o.total_amount) as total_spent
    FROM customers c
    LEFT JOIN orders o ON c.customer_id = o.customer_id
    WHERE o.order_date >= DATE '2024-01-01'
    GROUP BY c.customer_name
    HAVING COUNT(o.order_id) > 5
    ORDER BY total_spent DESC
    """,
    row_limit=50,
    timeout_seconds=60
)
```

### Handling Results
```python
if result["status"] == "success":
    print(f"Found {result['row_count']} rows")
    if result["truncated"]:
        print("Warning: Results were truncated")
    
    # Process rows
    for row in result["rows"]:
        # row is a list matching result["columns"] order
        pass
else:
    print(f"Error: {result['error_message']}")
```

## Security Considerations

1. **No Bind Parameters from User**: The tool deliberately doesn't accept bind parameters to prevent confusion about SQL injection. Users must embed values directly in the query.

2. **Audit Logging**: All executed queries should be logged with timestamp, user, and execution time.

3. **Permission Validation**: The tool relies on database permissions. Ensure the Oracle user only has SELECT grants.

4. **Cost Thresholds**: Set appropriate QUERY_MAX_COST based on your database size and performance characteristics.

5. **Network Security**: Ensure the database connection uses encryption (TCPS/SSL).