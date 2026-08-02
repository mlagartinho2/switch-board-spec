# Project constitution: Switchboard libraries

Ground rules that apply to both `switchboard-spring-boot2` and `switchboard-spring-boot3`. Run `/speckit.constitution` once per repo with this content — these are not feature-specific, so they shouldn't be re-litigated in every `/speckit.specify` prompt.

## Independence between the two libraries

- These are two fully independent libraries, one per Spring Boot major version, in separate repositories. Neither has a build or runtime dependency on the other.
- No shared module or shared source directory. Common logic is intentionally duplicated in each repo rather than extracted, so the two libraries can diverge in release cadence without coordination overhead.
- Each library has its own semantic versioning line, its own CI pipeline, its own README, and its own release schedule. A release of one must never be blocked by or triggered by the other.

## Framework vs. business logic boundary

- Each library ships only the *framework*: use-case registry, resolver chain, dispatcher, configuration binding, error mapping, and the observability endpoint.
- Business logic (`SwitchboardStrategy` implementations) and the HTTP controller always belong to the consuming application, never to the library.
- Every bean the library registers must be annotated `@ConditionalOnMissingBean`, so a consuming application can override any part of the framework without forking the library.

## Configuration-first design

- Anything that could plausibly need to change per-environment or per-team must be exposed as Spring configuration, not a code change: enabling/disabling a use case, the resolver priority order, the header name used for resolution.
- Config changes take effect on restart/redeploy. Runtime (no-restart) toggling is explicitly out of scope — do not introduce `@RefreshScope` or a feature-flag service dependency for this.

## Error handling

- A request that no resolver can match returns HTTP 501 Not Implemented.
- A request that resolves to a use-case key with no enabled/registered strategy returns HTTP 404 Not Found.
- These are distinct failure modes with distinct exception types and must not be collapsed into a single generic error.

## Keeping the two libraries in parity

- When a change is made to one library's framework code (not a Boot-version-specific fix), the same change must be ported to the other library before that unit of work is considered done, unless there's a documented reason it doesn't apply.
- Every such change is recorded in both repos' changelogs, even in the repo where no version bump happens yet, so a parity gap is never silently invisible.
- Boot-version-specific deviations (e.g., a resolver implementation that differs because one library targets Java 8 and the other Java 17+) must be commented in-code explaining why it's not a straight port.

## Testing

- Every `SwitchboardResolver` and every framework component (registry, dispatcher, composite resolver) has unit tests independent of a Spring context where possible.
- Auto-configuration is tested with `ApplicationContextRunner`, verifying it activates correctly, backs off when disabled via config, and respects `@ConditionalOnMissingBean` overrides.
- Each repo maintains a minimal sample application exercising at least one real strategy end-to-end, as the actual proof the library works for a consumer.

## Publishing

- Both libraries publish to internal Nexus/Artifactory only. No public Maven Central distribution, no OSS licensing requirements apply.
- Artifact and group IDs follow the pattern `com.yourorg:switchboard-spring-boot2` / `com.yourorg:switchboard-spring-boot3` (replace `yourorg` with the actual internal group).
