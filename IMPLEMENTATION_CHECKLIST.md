# Implementation Checklist: Protocol Separation

## Phase 1: Prepare Core RequestScope (Week 1, Days 1-2)

### Create Abstract Base
- [ ] Identify all protocol-agnostic code in RequestScope (lines 45-414)
- [ ] Create abstract RequestScope with generic fields only
- [ ] Move generic lifecycle methods to abstract class:
  - [ ] `saveOrCreateObjects()`
  - [ ] `publishLifecycleEvent()` (all overloads)
  - [ ] `runQueuedPreSecurityTriggers()`
  - [ ] `runQueuedPreFlushTriggers()`
  - [ ] `runQueuedPreCommitTriggers()`
  - [ ] `runQueuedPostCommitTriggers()`
  - [ ] UUID/caching methods
  - [ ] Metadata methods
- [ ] Make constructor protected (not public)
- [ ] Add copy constructor (protected)
- [ ] Deprecate old `RequestScope.builder()` → redirect to protocol-specific builders

### Unit Tests for New Core
- [ ] Create `AbstractRequestScopeTest` to verify core functionality
- [ ] Test lifecycle hook publishing independently
- [ ] Test entity caching (new, dirty, deleted resources)
- [ ] Test permission executor initialization

---

## Phase 2: Create JsonApiRequestScope (Week 1, Days 3-4)

### New Class
- [ ] Create `elide-jsonapi/JsonApiRequestScope.java`
- [ ] Extend abstract `RequestScope`
- [ ] Add JSON-API specific fields:
  - [ ] `sparseFields` (Map<String, Set<String>>)
  - [ ] `expressionsByType` (Map<String, FilterExpression>)
  - [ ] `globalFilterExpression` (FilterExpression)
  - [ ] `entityProjectionResolver`, `entityProjection`
- [ ] Implement JSON-API specific methods:
  - [ ] `parseSparseFields(Map<String, List<String>>)`
  - [ ] `getFilterExpressionByType(String type)`
  - [ ] `getFilterExpressionByType(Type<?> entityClass)`
  - [ ] `getLoadFilterExpression(Type<?> loadClass)`
  - [ ] `getExpressionForRelation(Type<?> parentType, String relationName)`

### Builder Class
- [ ] Create `JsonApiRequestScopeBuilder` inner class
- [ ] Match interface of old `RequestScope.RequestScopeBuilder`
- [ ] Add factory method: `JsonApiRequestScope.jsonApiBuilder()`

### Backward Compatibility Layer
- [ ] Create adapter: old RequestScope constructor → JsonApiRequestScope
- [ ] Add deprecation warning to old constructor
- [ ] Ensure existing tests pass without modification

### Unit Tests
- [ ] `JsonApiRequestScopeTest` → test sparse fields parsing
- [ ] Test RSQL filter expression storage/retrieval
- [ ] Test builder pattern
- [ ] Test that it inherits core lifecycle behavior correctly

---

## Phase 3: Create JsonApiPersistentResource (Week 2, Days 1-2)

### Extract Serialization Logic
- [ ] Create `elide-jsonapi/JsonApiPersistentResource.java`
- [ ] Move from `PersistentResource.java`:
  - [ ] `toResource()` (all overloads) — lines 1544-1567
  - [ ] `getRelationships()` (both versions) — lines 1617-1643
  - [ ] `getRelationshipsWithRelationshipFunction()` — lines 1652-1682
  - [ ] `getAttributes()` — lines 1689-1698
  - [ ] `filterFields()` — lines 1919-1944 (but keep core logic)
  - [ ] Related helper methods

### Keep in Core PersistentResource
- [ ] Verify these stay in `elide-core/PersistentResource.java`:
  - [ ] `updateAttribute()` — protocol-agnostic
  - [ ] `updateRelation()` (both versions) — protocol-agnostic
  - [ ] `removeRelation()` — protocol-agnostic
  - [ ] `addRelation()` — protocol-agnostic
  - [ ] `clearRelation()` — protocol-agnostic
  - [ ] `deleteResource()` — protocol-agnostic
  - [ ] `getAttribute()` — protocol-agnostic
  - [ ] All relationship loading methods
  - [ ] Inverse relationship handling

### Update References
- [ ] Fix imports in `JsonApiPersistentResource` for JSON-API models:
  - [ ] `com.yahoo.elide.jsonapi.models.Resource`
  - [ ] `com.yahoo.elide.jsonapi.models.Relationship`
  - [ ] `com.yahoo.elide.jsonapi.models.Meta`
  - [ ] `com.yahoo.elide.jsonapi.JsonApiSettings`

### Unit Tests
- [ ] `JsonApiPersistentResourceTest` → test serialization
- [ ] Test that core methods still exist in base class
- [ ] Test relationship and attribute extraction
- [ ] Ensure sparse fields filtering works correctly

---

## Phase 4: Migrate RSQL Parsing (Week 2, Day 3)

### New Parser Class
- [ ] Create `elide-jsonapi/filter/RsqlFilterParser.java`
- [ ] Move from RequestScope:
  - [ ] Existing RSQL parsing logic
  - [ ] `getFilterParams()` helper method
- [ ] Add factory method to parse and store in JsonApiRequestScope

### Update RequestScope
- [ ] Remove JSON-API filter parsing logic from core

### Unit Tests
- [ ] `RsqlFilterParserTest` → test RSQL → FilterExpression conversion
- [ ] Test filter param extraction

---

## Phase 5: Update All Callers (Week 2-3, Days 4-7)

### JSON-API Endpoint
- [ ] Find all `new RequestScope(...)` calls in `elide-jsonapi/`
- [ ] Replace with `new JsonApiRequestScope(...)` or builder
- [ ] Files to update (estimated ~10-15):
  - [ ] `JsonApiEndpoint.java`
  - [ ] Request processors
  - [ ] Test fixtures

### Test Fixtures
- [ ] Update all test helpers that create RequestScope
- [ ] Ensure tests create `JsonApiRequestScope` for JSON-API tests
- [ ] Create generic `AbstractRequestScopeTest` for core functionality

### GraphQL (optional, Phase 1)
- [ ] Create `GraphQLRequestScope.java` (empty for now)
- [ ] Ensure GraphQL tests still pass

### Search and Replace Pattern
```
// Old pattern
RequestScope scope = new RequestScope(
    route, transaction, user, requestId, elideSettings, entityProjection
);

// New pattern
JsonApiRequestScope scope = new JsonApiRequestScope(
    route, transaction, user, requestId, elideSettings, entityProjection
);
```

### Manual Verification
- [ ] Grep for "new RequestScope" in elide-jsonapi → 0 results (except deprecated adapter)
- [ ] Grep for "RequestScope.builder" in elide-jsonapi → 0 results
- [ ] All JSON-API tests still passing

---

## Phase 6: Deprecation & Cleanup (Week 3)

### Deprecate Old API
- [ ] Add `@Deprecated` to:
  - [ ] `RequestScope(Route, DataStoreTransaction, User, UUID, ElideSettings, Function)` constructor
  - [ ] `RequestScope.builder()` factory method
  - [ ] `RequestScope.parseSparseFields()` static method
  - [ ] `RequestScope.getFilterExpressionByType()` methods
  - [ ] `RequestScope.getLoadFilterExpression()`
  - [ ] `RequestScope.getExpressionForRelation()`
- [ ] Add Javadoc: "Use JsonApiRequestScope instead"

### Backward Compatibility Adapter (optional)
- [ ] If needed, keep old RequestScope implementation that delegates to JsonApiRequestScope
- [ ] Mark with `@Deprecated`
- [ ] Remove in next major version

### Update Documentation
- [ ] Update README/wiki with new usage pattern
- [ ] Document how to create a new protocol (GraphQLRequestScope example)
- [ ] Migration guide for users still using old API

---

## Phase 7: Comprehensive Testing (Week 4)

### Unit Tests
- [ ] Core RequestScope lifecycle tests: ✅ passing
- [ ] JsonApiRequestScope specific tests: ✅ passing
- [ ] JsonApiPersistentResource serialization tests: ✅ passing
- [ ] RsqlFilterParser tests: ✅ passing

### Integration Tests
- [ ] Full JSON-API endpoint tests: ✅ passing
- [ ] GraphQL endpoint tests: ✅ passing (unchanged)
- [ ] Async/export tests: ✅ passing (use core RequestScope)
- [ ] Aggregation tests: ✅ passing (use core RequestScope)

### Backward Compatibility Tests
- [ ] Old `RequestScope(...)` constructor still works (deprecated): ✅ passing
- [ ] Old `RequestScope.builder()` still works (deprecated): ✅ passing
- [ ] Old `RequestScope` code paths redirect correctly: ✅ passing

### Performance Tests
- [ ] No regression in request latency
- [ ] No regression in memory usage
- [ ] No regression in throughput

### Code Coverage
- [ ] elide-core: ≥80%
- [ ] elide-jsonapi: ≥80%
- [ ] Combined: ≥85%

---

## Phase 8: Documentation (Week 4)

### Code Documentation
- [ ] Javadoc for abstract RequestScope
- [ ] Javadoc for JsonApiRequestScope
- [ ] Architecture decision record (ADR)

### Developer Guide
- [ ] "How to create a new protocol" guide
- [ ] Example: `CustomProtocolRequestScope` stub
- [ ] Copy constructor pattern explanation

### Migration Guide
- [ ] For users upgrading from old version
- [ ] Step-by-step migration path
- [ ] Deprecation timeline

---

## Rollback Plan

If issues arise:
- [ ] Keep old RequestScope code in a branch
- [ ] Restore if needed (estimated 4 hours)
- [ ] Fallback: revert last 2 phases, keep abstract base

---

## Sign-Off

- [ ] Code review approved
- [ ] All tests passing
- [ ] Performance validated
- [ ] Documentation complete
- [ ] Merged to main

---

## Timeline Summary

| Week | Phase | Tasks |
|------|-------|-------|
| **1** | 1-2 | Abstract core + JsonApiRequestScope |
| **2** | 3-4 | JsonApiPersistentResource + RSQL parser |
| **3** | 5-6 | Migrate callers + deprecate old API |
| **4** | 7-8 | Testing + documentation |

**Total Effort**: ~60 hours (1.5 developer weeks)
**Risk Level**: Low (can be undone; incremental changes)
**Go-Live**: Week 4, after comprehensive testing
