# Elide Architecture Refactoring: Protocol-Agnostic Core

## Overview

This document describes the architectural changes needed to separate protocol-specific concerns (JSON-API, GraphQL, etc.) from the core Elide data access framework.

## Current Problem

Elide's `RequestScope` and `PersistentResource` classes are tightly coupled to JSON-API concerns:

### RequestScope (elide-core)
- **JSON-API specific**: `sparseFields`, `expressionsByType`, `globalFilterExpression`
- **JSON-API specific methods**: `parseSparseFields()`, `getFilterExpressionByType()`, `getLoadFilterExpression()`, `getExpressionForRelation()`
- **Generic methods**: Lifecycle hooks, CRUD event publishing, entity caching

### PersistentResource (elide-core)
- **JSON-API specific**: `toResource()`, `getRelationships()`, `getAttributes()`
- **Generic methods**: `updateAttribute()`, `updateRelation()`, `deleteResource()`

### Result
- GraphQL has to work around JSON-API semantics
- Can't add new protocols (REST, gRPC, etc.) cleanly
- ~50% of core is protocol-specific; 50% is reusable

## Solution Architecture

### New Class Hierarchy

```
RequestScope (abstract, in elide-core)
  ├── JsonApiRequestScope (in elide-jsonapi)
  ├── GraphQLRequestScope (in elide-graphql)
  └── [Future] RestRequestScope, GrpcRequestScope, etc.

PersistentResource (concrete, in elide-core)
  ├── JsonApiPersistentResource (in elide-jsonapi)
  └── [Future] GraphQLPersistentResource
```

### Core Responsibilities (Protocol-Agnostic)

**RequestScope** handles:
- Transaction management
- User context & permissions
- Entity caching (new, dirty, deleted resources)
- Lifecycle hook publishing (CREATE, UPDATE, DELETE events)
- Request metadata storage
- Audit logging
- Object UUID/ID tracking

**PersistentResource** handles:
- Entity mutation (updateAttribute, updateRelation, addRelation, removeRelation, deleteResource)
- Relationship loading (toOne, toMany)
- Permission checking
- Lifecycle hook invocation
- Inverse relationship management

### Protocol-Specific Responsibilities

**JsonApiRequestScope** adds:
- Sparse fields parsing (`fields[User]=name,email`)
- RSQL filter expression storage & retrieval
- JSON-API specific request semantics

**JsonApiPersistentResource** adds:
- Serialization to JSON-API `Resource` objects
- JSON-API relationship and attribute extraction
- JSON-API metadata/links handling

**GraphQLRequestScope** (future) adds:
- GraphQL selection set handling
- GraphQL variable storage
- GraphQL query semantics

## File Structure (New)

```
elide-core/
  src/main/java/com/yahoo/elide/core/
    ├── RequestScope.java (abstract)
    ├── PersistentResource.java (concrete, no JSON-API)
    └── ... (other generic components)

elide-jsonapi/
  src/main/java/com/yahoo/elide/jsonapi/
    ├── JsonApiRequestScope.java (new, extends RequestScope)
    ├── JsonApiPersistentResource.java (new, extends PersistentResource)
    ├── filter/
    │   └── RsqlFilterParser.java (moved from core)
    └── ... (existing JSON-API components)

elide-graphql/
  src/main/java/com/yahoo/elide/graphql/
    ├── GraphQLRequestScope.java (new, extends RequestScope)
    └── ... (existing GraphQL components)
```

## Benefits

1. **Cleaner Core**: Elide-core becomes a pure data access abstraction layer
2. **Easier to Extend**: Adding new protocols requires only a RequestScope subclass
3. **Better Testing**: Each protocol can be tested independently
4. **GraphQL Parity**: GraphQL becomes a first-class citizen, not a bolt-on
5. **Future Protocols**: REST, gRPC, WebSocket, etc. can be added easily
6. **Backward Compatible**: Public APIs remain stable during transition

## Migration Path

1. **Introduce abstract RequestScope** (no breaking changes)
2. **Create JsonApiRequestScope** (alongside existing code)
3. **Gradually migrate JSON-API callers** to use new class
4. **Deprecate old RequestScope constructor** after migration
5. **Remove old code** in next major version

## Metrics

- **Lines moved out of core**: ~400-500 lines
- **New files**: 2-3 (JsonApiRequestScope, JsonApiPersistentResource, RsqlFilterParser)
- **Modified files**: ~50-70 (updating callers)
- **Estimated effort**: 60 hours
- **Risk**: Low (can be done incrementally with zero breaking changes)

## Success Criteria

✅ All JSON-API tests pass with new `JsonApiRequestScope`
✅ All GraphQL tests pass without modification
✅ No changes to public API contracts
✅ New protocols (e.g., REST) can be added without modifying core
✅ Code coverage remains >80%
