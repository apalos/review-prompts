# OP-TEE Technical Deep-dive Patterns

## Core instructions

- Trace full execution flow, gather additional context from the call chain to make sure you fully understand
- IMPORTANT: never make assumptions based on return types, checks, assert(), comments, or error
  handling patterns - explicitly verify the code is correct by tracing concrete execution paths
- IMPORTANT: never skip any steps just because you found a bug in a previous step.
- IMPORTANT: OP-TEE documentation and comments are sometimes incomplete, outdated, or
  misleading. When relying on documentation or comments to understand behavior:
  - Always read the ACTUAL IMPLEMENTATION, not just the comment
  - Check for #ifdef/#else branches - the same comment may be copy-pasted to multiple
    implementations with different semantics
  - If a function comment says "returns X" but the code shows conditional behavior
    based on config options, verify which config applies to your analysis
- Never report errors without checking to see if the error is impossible in the
  call path you found.
    - Some call paths might always check IS_ENABLED(feature) before
      dereferencing a variable
    - The actual implementations of "feature" might not have those checks,
      because they are never called unless the feature is on
    - It's not enough for the API contract to be unclear, you must prove the
    bug can happen in practice.
    - Do not recommend defensive programming unless it fixes a proven bug.

### Error Handling

**Notes**:
- If code checks for a condition via assert() assume that condition will never happen, unless you can provide concrete evidence of that condition existing via code snippets and call traces

### Bounds & Validation

**Important**: Never suggest defensive bounds checks unless you can prove the source is untrusted.

### Context Rules
- **typeof() safety**: Can be used with container_of() before init
- **likely()/unlikely()**: don't report on changes to compiler hinting unless
  they introduce larger logic bugs
- READ_ONCE() is not required when the data structure being read is protected by a lock we're currently holding

### Resource Management Knowledge
- Every resource must have balanced lifecycle: alloc→init→use→cleanup→free
- All pointers have the same size: `char *foo` takes as much room as `int *foo`
  - but for code clarity, if we're allocating an array of pointers, and using
    `sizeof(type *)` to calculate the size, we should use the correct type
- If you find a type mismatch (using `*foo` instead of `foo` etc), trace the type
  fully and check against the expected type to make sure you're flagging it correctly
- global variables and static variables are zero filled automatically
- When fields move into a sub-struct, search for all static instances of the parent struct and verify their initializers are updated — especially locks, which require explicit initialization macros (e.g., `MUTEX_INITIALIZER`) rather than zero-fill
- when freeing/destroying resources referenced by structure fields, ensure pointer fields are set to NULL to prevent use-after-free on reuse if the structure is not also immediately freed
  - ex: unregister_foo() { foo->dead = 1; free(foo->ptr); add to list}
       register_foo() { pull from list ; skip allocation of foo->ptr; foo->ptr->use_after_free;}
  - safe: free(foo->ptr); ... ; free(foo);
    - nobody will find foo->ptr is non-NULL because foo is gone
  - Assume free(); malloc(); APIs handle this properly unless you find proof initialization is skipped

### for loops
- for(init; condition; advance) { body } -- checks 'condition' BEFORE executing 'body'
- for(init; condition; advance) { body } -- 'advance' only runs AFTER 'body'

### Additional resource checks
- Resource switching detection:
  - Check every path where a function returns a different resource than it was meant to modify
  - ensure the proper locks are held or released as needed.
- Caller expectation tracing: What does the caller expect to happen to the resources it passed into functions?

- strscpy() auto-detects array sizes when compiler can find the type
- `char *s = ""`; `strlen(s)` returns zero, but `s[0]` is safe to access
- memcpy(dst + (offset & mask), src, size) usually has alignment validation elsewhere
- Global arrays with MAX_FOO: check if it is possible to create more than MAX_FOO elements

### NULL Pointer Dereference

Review the examples carefully before analyzing NULL pointer dereferences.
Misunderstanding what constitutes a dereference causes false positives.

**Dereference Types**:
```
- val = *foo dereferences foo
  - if (foo) val = *foo is safe
- val = foo[var] dereferences foo but only reads var without dereference
  - if (foo) val = foo[var] is safe
- val = foo->ptr dereferences foo but only reads ptr without dereference
  - if (foo) val = foo->ptr is safe
- val = *foo->ptr dereferences foo and ptr.
  - if (foo && foo->ptr) val = *foo->ptr is safe
- val = (*foo)->ptr dereferences foo, then dereferences what foo points to, then
         reads ptr without dereference
  - if (foo && *foo) val = (*foo)->ptr is safe
- val = foo->ptr->something dereferences foo and ptr but only reads something
  - if (foo && foo->ptr) val = foo->ptr->something is safe
```

**Key Points**:
1. **Reading a pointer field is not the same as dereferencing it**
   - `ptr = foo->bar` dereferences `foo`, reads `bar`, but does NOT dereference `bar`
   - The dereference happens when you later USE `ptr`

2. **Check where the pointer is actually used**
   - If `ptr = foo->bar` and `bar` can be NULL, the problem occurs when `ptr` is dereferenced
   - Example: `ptr->field` or `*ptr` or `ptr[0]` or passing `ptr` to a function that dereferences it

3. **NULL checks protect the pointer being checked**
   - `if (foo)` protects dereferencing `foo`
   - `if (foo && foo->bar)` protects dereferencing both `foo` and `foo->bar`
