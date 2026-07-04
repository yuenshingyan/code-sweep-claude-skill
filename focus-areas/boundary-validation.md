# Boundary Validation

- **Missing input validation**: API endpoints that accept user input without checking length, range, format, or type constraints
- **Inconsistent validation**: One endpoint validates a field (e.g., email format) but another endpoint accepting the same field doesn't
- **String length before DB insert**: Strings accepted without length limits that will hit DB column limits and error at write time
- **Enum/type deserialization**: Accepting serialized values that the business logic can't handle or doesn't expect
- **Negative/zero values**: Numeric inputs (pagination, quantities, IDs) not checked for negative or zero when those are nonsensical
- **Path traversal**: File paths or identifiers built from user input without sanitization
