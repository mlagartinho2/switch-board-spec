# Project constitution: Switchboard

Ground rules for the `switchboard` repository. Run `/speckit.constitution` once with this content — these are not feature-specific, so they shouldn't be re-litigated in every `/speckit.specify` prompt.

## Single library, two modules, one release

- Switchboard is one repository, released as two modules in lockstep: `switchboard-autoconfigure` (all the actual code) and `switchboard-spring-boot-starter` (dependency-only, no code). Both always share the same version number — there is no scenario where one is released without the other.
- Spring Boot 4.x is the minimum and only supported version. Spring Boot 2.x and 3.x are not supported and will not be — this was a deliberate decision to retire, not a gap to fill later. Do not add compatibility shims, conditional imports, or dual servlet-API code paths for older Boot versions.
- Java 17+ only. Use modern language features (`instanceof` pattern matching, records, sealed interfaces) freely — there is no older Java baseline to protect.

## Framework vs. business logic boundary

- Switchboard ships only the *framework*: use-case registry, resolver chain, dispatcher, configuration binding, error mapping, and the observability endpoint.
- Business logic (`SwitchboardStrategy` implementations) and the HTTP controller always belong to the consuming application, never to this library.
- Every bean this library registers must be annotated `@ConditionalOnMissingBean`, so a consuming application can override any part of the framework without forking it.

## Configuration-first design

- Anything that could plausibly need to change per-environment or per-team must be exposed as Spring configuration, not a code change: enabling/disabling a use case, the resolver priority order, the header name used for resolution.
- Config changes take effect on restart/redeploy. Runtime (no-restart) toggling is explicitly out of scope — do not introduce `@RefreshScope` or a feature-flag service dependency for this.

## Error handling

- A request that no resolver can match returns HTTP 501 Not Implemented.
- A request that resolves to a use-case key with no enabled/registered strategy returns HTTP 404 Not Found.
- These are distinct failure modes with distinct exception types and must not be collapsed into a single generic error.

## Code quality baseline (Effective Java)

- **Validate constructor/method arguments eagerly** (Item 49): every framework class taking required collaborators (`SwitchboardDispatcher`, `SwitchboardResolverChain`, etc.) null-checks them with `Objects.requireNonNull(x, "message")` at construction, not on first use. A missing bean should fail at wiring time with a clear message, never with a later `NullPointerException` at request time.
- **Fail fast and clearly on duplicate registration** (Item 49): if two `SwitchboardStrategy` beans register the same `caseKey()`, `SwitchboardRegistry` must throw at startup naming both conflicting classes and the shared key — never silently let one overwrite the other via an unguarded `Collectors.toMap`.
- **Minimize mutability of registries and resolver lists** (Item 17, 15): once built, `SwitchboardRegistry`'s map and `SwitchboardResolverChain`'s resolver list are never mutated again. Store them as unmodifiable (`Map.copyOf` / `List.copyOf`) rather than trusting the caller not to mutate the backing collection later.
- **Defensively copy mutable collections handed in from outside** (Item 50): any `List`/`Map` passed into a framework constructor (e.g. ordered resolvers, Spring-bound config maps) is copied before being stored, since the caller's reference may be mutated after construction.
- **Scope and justify unchecked casts** (Item 27): the generic cast in `SwitchboardDispatcher.dispatch` is unavoidable (the registry is keyed by runtime strings, not compile-time types) — keep `@SuppressWarnings("unchecked")` on the narrowest possible scope (the local variable, not the method) with a one-line comment explaining why it's safe.
- **Prefer `Optional` for single-value absence, never for collections** (Item 55): `SwitchboardRegistry.find` and resolver results correctly return `Optional<T>` for a single missing value; don't extend this to collection-returning methods, which should return an empty collection instead.

## Testing

- Every `SwitchboardResolver` and every framework component (registry, dispatcher, resolver chain) has unit tests independent of a Spring context where possible.
- Auto-configuration is tested with `ApplicationContextRunner`, verifying it activates correctly, backs off when disabled via config, and respects `@ConditionalOnMissingBean` overrides.
- Maintain a minimal sample application exercising at least one real strategy end-to-end, as the actual proof the library works for a consumer.

## Publishing

- Publishes to internal Nexus/Artifactory only. No public Maven Central distribution, no OSS licensing requirements apply.
- Artifact and group IDs: `com.yourorg:switchboard-autoconfigure` / `com.yourorg:switchboard-spring-boot-starter` (replace `yourorg` with the actual internal group).
- Versioning starts fresh at 2.0.0, marking the clean break from the retired Boot 2/3 line — this is not a continuation of a 1.x series.
