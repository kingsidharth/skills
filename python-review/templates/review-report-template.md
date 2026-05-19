# Python Review

## System map

### Entry points
- 

### Data flow
- 

### Persistence
- 

### State machines
- 

### Test setup
- 

### Uncertainties
- 

---

## Highest-risk findings

1. **[Issue class]** — `path/to/file.py:function_name`
   - Evidence:
   - Why it matters:
   - Suggested fix:

---

## Pattern findings

### Separation of concerns
- 

### Duplicate ways of doing things
- 

### State machine reliability
- 

### Serialization/model boundaries
- 

### Error handling/retries/idempotency
- 

### Tests
- 

### Performance
- 

### Security/data safety
- 

### Naming/comments/docs
- 

### Shims/v2/backward compatibility
- 

---

## Tests needed or added

### Functional
- 

### Integration
- 

### Workflow/state-machine
- 

### Regression
- 

---

## Final changes needed

1. 
2. 
3. 

---

## Remaining risks

- 

---

## Do not do

- No `v2`, shims, or backward-compat layers unless explicitly required.
- No useless docs.
- No broad wrappers.
- No fake service layer.
- No broad exception swallowing.
- No destructive tests without isolated test resources.
