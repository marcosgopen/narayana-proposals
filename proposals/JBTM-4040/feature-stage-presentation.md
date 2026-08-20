# Feature Stage Management in Narayana
## Learning from WildFly and Quarkus

---

## Section 1: Quarkus Extension Maturity Model

### Levels: Experimental → Preview → Stable

**How It Works:**
- **Metadata-driven**: Status in `META-INF/quarkus-extension.yaml` ([Extension Metadata Guide](https://quarkus.io/guides/extension-metadata))
- **All extensions in one BOM**: No artifact separation, all in `io.quarkus.platform:quarkus-bom` ([Platform Guide](https://quarkus.io/guides/platform))
- **Opt-in**: Per-extension config properties (e.g., `quarkus.<feature>.enabled=true`)
- **Intra-extension features**: Disabled-by-default config, no framework enforcement

**Key Takeaway:** Metadata-level isolation — maturity is informational, all dependencies shared

**Real Examples:**
- **Experimental**: 
  - `quarkus-langchain4j` - LLM integration ([quarkiverse/quarkus-langchain4j](https://github.com/quarkiverse/quarkus-langchain4j))
  - `quarkus-websockets-next` - New WebSocket API ([guide](https://quarkus.io/guides/websockets-next))
- **Preview**: 
  - `quarkus-grpc` was preview until 2.x ([gRPC guide](https://quarkus.io/guides/grpc-getting-started))
  - Various MicroProfile 6.0 extensions during transition phase
- **Stable**: 
  - `quarkus-resteasy-reactive` / `quarkus-rest` ([REST guide](https://quarkus.io/guides/rest))
  - `quarkus-hibernate-orm` ([ORM guide](https://quarkus.io/guides/hibernate-orm))
  - `quarkus-smallrye-openapi` ([OpenAPI guide](https://quarkus.io/guides/openapi-swaggerui))

**Quarkus References:**
- Platform: [github.com/quarkusio/quarkus-platform](https://github.com/quarkusio/quarkus-platform)
- Extension metadata format: [quarkus.io/guides/extension-metadata](https://quarkus.io/guides/extension-metadata)
- Browse extensions with status: [code.quarkus.io](https://code.quarkus.io)

---

## Section 2: WildFly Stability Levels

### Levels: Experimental → Preview → Community → Default

**How It Works:**
- **Artifact-driven**: Separate feature packs per stability level ([Different Flavors of WildFly](http://docs.wildfly.org/40/Different_Flavors_of_WildFly.html))
- **Provisioning control**: `--stability-level` in Galleon ([WFLY-19021](https://docs.wildfly.org/wildfly-proposals/wf-galleon/WFLY-19021-Stability_In_Provisioning.html))
- **Runtime control**: `--stability` CLI flag at server start
- **Intra-module**: Per-attribute `Stability` enum, filtered at management model registration ([Feature Process](https://docs.wildfly.org/wildfly-proposals/FEATURE_PROCESS.html))

**Key Takeaway:** Artifact-level + framework-enforced isolation — features below threshold not present on disk

**Real Examples:**
- **Experimental**: 
  - Jakarta EE 11 preview features during early development
  - New subsystem proposals in `wildfly-proposals` before acceptance
- **Preview**: 
  - MicroProfile Reactive Messaging ([WFLY-13640](https://issues.redhat.com/browse/WFLY-13640))
  - MicroProfile Telemetry ([WFLY-18176](https://issues.redhat.com/browse/WFLY-18176))
  - Features in WildFly Preview distribution ([WildFly Preview docs](http://docs.wildfly.org/40/WildFly_and_WildFly_Preview.html))
- **Community**: 
  - Undertow (web server), Transactions (Narayana integration), Messaging (ActiveMQ Artemis)
  - Standard Jakarta EE subsystems in default WildFly distribution
- **Default**: 
  - Core EE APIs with long-term product support guarantees

**WildFly References:**
- Stability in provisioning: [docs.wildfly.org/wildfly-proposals/wf-galleon/WFLY-19021](https://docs.wildfly.org/wildfly-proposals/wf-galleon/WFLY-19021-Stability_In_Provisioning.html)
- Feature development process: [docs.wildfly.org/wildfly-proposals/FEATURE_PROCESS.html](https://docs.wildfly.org/wildfly-proposals/FEATURE_PROCESS.html)
- WildFly proposals repo: [github.com/wildfly/wildfly-proposals](https://github.com/wildfly/wildfly-proposals)
- WildFly Preview distribution: [docs.wildfly.org/40/WildFly_and_WildFly_Preview.html](http://docs.wildfly.org/40/WildFly_and_WildFly_Preview.html)

---

## Section 3: Narayana's Proposed Approach

### Three Stages: Experimental → Tech Preview → Stable

| Stage | Production Use | Isolation |
|-------|----------------|-----------|
| **Experimental** | ❌ NOT ALLOWED | Config guard + optional dependencies |
| **Tech Preview** | ❌ NOT ALLOWED | Runtime config (disabled by default) |
| **Stable** | ✅ ALLOWED | Standard implementation |

### Hybrid Model

**From WildFly:** Artifact isolation for significant dependencies, hard opt-in (disabled by default)

**From Quarkus:** Metadata-driven, config-property activation, same-module pattern for lightweight features

### Implementation Pattern

**All non-stable features require:**
- Javadoc + manual warnings ([Narayana docs style](https://jbosstm.github.io/public/public//docs/project/index.html#_notes_and_warnings))
- `@Experimental` / `@TechPreview` annotation (lightweight marker, tracked in [JBTM-4041](https://issues.redhat.com/browse/JBTM-4041))
- JIRA/GitHub label (`experimental` or `tech-preview`)
- Config guard (disabled by default)

**Dependency isolation — choose one:**

| Pattern | When | Example |
|---------|------|---------|
| **Separate module** | Significant new dependencies | `narayana-jts-idlj` (separate from core) |
| **Optional dependency** | Classpath-activated integration | `<optional>true</optional>` + runtime check |
| **Same module** | No new dependencies | Config guard only |

**Same-Module Pattern:**
```java
@Experimental("Feature_ABC")
public class FeatureABC {
    public FeatureABC(Configuration config) {
        if (!config.isExperimentalEnabled("Feature_ABC")) {
            throw new IllegalStateException("Feature disabled. "
                + "Set narayana.experimental.feature_abc.enabled=true");
        }
        logger.warn("Experimental feature Feature_ABC enabled.");
    }
}
```

**Integration checks before merge:**
- **WildFly**: Can new `module.xml` dependency be `optional="true"`? If NO → separate module required
- **Quarkus**: Does `quarkus-extension.yaml` reflect correct status?

**Narayana References:**
- Repo: [github.com/jbosstm/narayana](https://github.com/jbosstm/narayana)
- Docs: [jbosstm.github.io](https://jbosstm.github.io/)
- Governance: [github.com/wildfly/wildfly-governance/blob/main/narayana/GOVERNANCE.md](https://github.com/wildfly/wildfly-governance/blob/main/narayana/GOVERNANCE.md)

---

## Section 4: Four Critical Decision Points

### 1. Use Annotations or Not?

**Proposed:** Introduce `@Experimental` / `@TechPreview` annotations as lightweight markers (metadata only)

**Alternative:** No annotations - rely only on Javadoc, config guards, and manual documentation

**Recommendation:** Use annotations. Benefits:
- Source-level visibility for developers and reviewers
- Tooling can discover all non-stable code paths automatically
- Consistent with WildFly's `Stability` annotations and modern Java practices
- Lightweight (metadata only, no runtime overhead)

**If yes, then:** Annotations should be markers, not interceptors. Config property remains the control gate.

**Tracked in:** [JBTM-4041](https://issues.redhat.com/browse/JBTM-4041)

---

### 2. Module Split Threshold

**Question:** When must a feature be in a separate Maven module?

**Proposed decision tree:**
- Significant new dependencies (Infinispan, gRPC, etc.) → **Separate module**
- WildFly `module.xml` dependency cannot be `optional="true"` → **Separate module**
- Small integration, classpath-activated → **Optional dependency**
- No new dependencies → **Same module + config guard**

**Needs clarity:** Is this threshold well-defined enough?

---

### 3. Transition Approval Process

**Proposed:** Team discussion (Zulip/Slack/GitHub) + existing [governance process](https://github.com/wildfly/wildfly-governance/blob/main/narayana/GOVERNANCE.md)

**Alternative:** Formal checklist or milestone criteria

**Question:** Is discussion-based approval sufficient, or add formalization?

---

### 4. BOM Inclusion for Non-Stable Modules

**Question:** Should Experimental/Tech Preview modules be included in `narayana-bom`?

**Proposed:**
- Non-stable modules included in BOM for version alignment
- BOM only manages versions; consumers choose artifacts
- BOM documentation clearly marks non-stable modules

**Alternative:** Separate BOM for non-stable features (`narayana-preview-bom`)

**Recommendation:** Include in main BOM. Quarkus includes all extension statuses in one BOM; WildFly separates via feature packs but their module versions are unified. Separation would complicate version management for consumers using both stable + preview features.

---

## Summary Table

| Decision | Proposed | Alternative | Status |
|----------|----------|-------------|--------|
| **Annotations** | Use lightweight markers | No annotations (Javadoc only) | Recommend annotations |
| **Module split** | 3-path decision tree | Different thresholds | Needs validation |
| **Transitions** | Discussion-based | Formal checklist | Needs agreement |
| **BOM inclusion** | All modules in one BOM | Separate preview BOM | Recommend unified |
