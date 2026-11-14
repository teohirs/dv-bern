# Quarkus Crash Course — Development Track for the DV Bern Interview

*13 Nov 2026 · Focus: development only (business logic, interfaces, persistence, testing). Hosting, containers, OpenShift and CI/CD are covered separately.*

---

## 0. What the role actually asks for (development side)

From the job ad ("Senior Backend Java Entwickler:in mit DevOps-Know-how", DV Bern AG, product **kiBon Stip**):

| Ad says | What it means technically | Covered in |
|---|---|---|
| Fachliche Anforderungen → zuverlässige Geschäftslogik | Domain modelling, transactions, state/workflow, validation, consistency under concurrency | Day 2 AM |
| Entwicklung von Schnittstellen | REST APIs (likely contract-first), REST/SOAP clients to third-party systems | Day 1 PM, Day 2 PM |
| Automatisierung administrativer Abläufe | Scheduled jobs, batch processing, generated documents (PDF), e-mail notifications, imports | Day 2 PM |
| Code Reviews, automatisierte Tests | Test pyramid in Quarkus, architecture tests, coverage/Sonar gate | Day 2 PM |
| Jakarta EE mit Quarkus | CDI, Jakarta REST, JPA, JTA on Quarkus | Day 1 AM |
| Fundierte Kenntnisse in Postgres, SQL | Schema design, migrations, query performance, locking | Day 2 AM |

### The likely real stack (from DV Bern's public GitHub)

DV Bern publishes a repository called `dvbern/stip` (backend/frontend/contract, GitLab CI, OpenShift release docs). This is very probably the kiBon Stip codebase. Its `backend/pom.xml` shows:

- **Java 25**, **Quarkus 3.33 LTS** (they follow LTS lines)
- **API:** `quarkus-rest` + Jackson, OpenAPI Generator from a `contract/` folder (**contract-first**), `jackson-databind-nullable`, SmallRye OpenAPI
- **Persistence:** Hibernate ORM + **Panache**, **Hibernate Envers** (auditing), **Liquibase**, PostgreSQL, **QueryDSL JPA** (type-safe queries), Hibernate Validator
- **Security:** `quarkus-oidc` (Keycloak), SmallRye JWT (incl. JWT *building*), Keycloak admin client
- **Integrations:** REST client, **Quarkus CXF** (SOAP) + JAXB/SAAJ, **Mailer**, **Quartz**, **Qute** templates, S3, antivirus scanning, Apache Tika, **iText** (PDF), fastexcel (Excel import)
- **Domain tooling:** **stateless4j** (state machine), **Lombok**, **MapStruct**, **Jilt** (builders), Checker Framework
- **Tests:** `quarkus-junit`, `quarkus-junit-component`, Mockito, **Testcontainers** (Postgres, LocalStack), RestAssured, **ArchUnit**, AssertJ, OIDC test server, JaCoCo; Spotless formatting; Sonar properties

> Treat this as strong evidence, not certainty. It makes great interview material: "I looked at your public repo and saw stateless4j and Envers. How do you model the Gesuch lifecycle?" shows preparation and genuine interest.

**This course is built around that stack.** Learn on **Quarkus 3.33 / 3.40 LTS**.

### Schedule

| Block | Topic | You leave with |
|---|---|---|
| Day 1 AM | Quarkus core for a Jakarta EE developer: build-time model, ArC/CDI, config, dev mode | A running `stip-lab` project with a service and typed config; the build-time model explained in 60 s |
| Day 1 PM | API layer: contract-first OpenAPI, Quarkus REST, DTO mapping, validation, error handling, security with Keycloak | A secured, contract-first `/gesuch` API with mapping, validation groups and ownership checks |
| Day 2 AM | Domain and persistence: Panache, QueryDSL, transactions, optimistic locking, Envers, Liquibase, PostgreSQL, state machines, architecture | An audited, versioned `Gesuch` with a tested state machine, a tuned search query and safe migrations |
| Day 2 PM | Interfaces and automation (REST/SOAP clients, Quartz, mail, PDF, file upload) + testing strategy and code quality | A deadline job, post-commit mail, a PDF decision letter, and a test suite spanning all levels with a coverage report |
| Wrap-up | Quarkus vs Spring Boot for business applications, interview drill, German glossary | Rehearsed answers (in German) to the 12 drill questions |

### How every session works

Each half-day ends with a **Session brief** containing:
- **Goal:** one sentence on why this session matters for the role.
- **Tasks:** numbered, concrete, with a time box. Do them in order.
- **Expected outcome (definition of done):** what must exist and work at the end. Each item is checkable.
- **Self-check:** questions to answer out loud, without notes, ideally in German. If you can't answer one, reread the section it points to.
- **If short on time:** the minimum to finish before moving on.

Rule of thumb: about 1.5 h reading the concepts, 2–2.5 h on the tasks. Commit to Git after each task (`git commit -m "Day1AM-T3: GesuchConfig mapping"`), so you can show the project history in the interview if asked.

**Running lab:** a mini "Stipendiengesuch" (scholarship application) service. You can submit an application, a case worker reviews it, a decision letter is generated as a PDF, and an e-mail is sent. It's small, but it touches every item the ad lists.

---

# DAY 1 — Morning: Quarkus core

## 1.1 The one idea: build-time augmentation

Classic Jakarta EE servers and Spring Boot do their framework work **at startup**: scanning classes, reading annotations, building the DI graph, generating proxies, bootstrapping Hibernate. Quarkus does that work **during the Maven build**, through *extensions*. Each extension has a runtime module and a *deployment* module whose `@BuildStep`s run only at build time and record bytecode.

What matters for a developer:
- **Errors move to build time.** Unsatisfied or ambiguous injection points, invalid config mappings and many misconfigurations fail the build or the dev-mode reload, not production.
- **Some config is build-time fixed.** For example, `quarkus.datasource.db-kind` can't be changed per environment without a rebuild.
- **Use the extension, not the raw library** (`quarkus-hibernate-orm`, not `hibernate-core`). Libraries without an extension (iText, stateless4j, MapStruct…) are fine on the JVM; they just get no build-time optimisation.
- **Avoid reflection-heavy tricks.** They work, but they fight the model.

## 1.2 Coming from Jakarta EE: what's the same, what's different

| Jakarta EE (WildFly/Payara) | Quarkus |
|---|---|
| WAR deployed into a server | Runnable app (`target/quarkus-app/quarkus-run.jar`); the "server" is assembled around your code at build time |
| Full CDI (Weld) | **ArC** = CDI Lite, resolved at build time |
| `@Stateless` EJB + container-managed transactions | `@ApplicationScoped` bean + `@Transactional` |
| `@Schedule` EJB timer | `@Scheduled` (with Quartz for persistence/clustering) |
| MDB / JMS | JMS extension or Reactive Messaging |
| JNDI datasource in `standalone.xml` | `quarkus.datasource.*` in `application.yaml/properties` |
| Security realm in server config | `quarkus-oidc` config + `@RolesAllowed` |
| `persistence.xml` | Optional; usually config only |
| Arquillian | `@QuarkusTest`, `@QuarkusComponentTest` |
| Redeploy cycle | `quarkus dev` live reload |

## 1.3 ArC (CDI) — what to know

- **Scopes:** `@ApplicationScoped` (lazy, client proxy, the default for services), `@Singleton` (no proxy), `@RequestScoped`, `@Dependent` (default when no scope is declared).
- **Simplifications:** a single constructor needs no `@Inject`; a qualifier on a field implies `@Inject`; prefer **package-private** fields over private ones (no reflection needed).
- **Unused beans are removed.** Mark beans looked up dynamically via `Instance<T>` with `@Unremovable`.
- **Useful annotations:** `@Startup`, `@Observes StartupEvent`, `@DefaultBean`, `@IfBuildProfile`, `@LookupIfProperty`.
- **Not supported:** portable extensions, `@ConversationScoped`, `@Specializes`. This matters when migrating old EE code.
- **Interceptors** work as in Jakarta EE and are a nice place for cross-cutting business concerns (e.g. audit logging, authorisation checks on domain operations).

## 1.4 Configuration

The stack uses `quarkus-config-yaml`, so expect `application.yaml`:

```yaml
quarkus:
  datasource:
    db-kind: postgresql          # build-time fixed
  hibernate-orm:
    log:
      sql: false
"%dev":
  quarkus:
    hibernate-orm:
      log:
        sql: true
kstip:
  gesuch:
    max-dokumente: 20
    frist-tage: 30
```

Type-safe config for business settings:

```java
@ConfigMapping(prefix = "kstip.gesuch")
public interface GesuchConfig {
    int maxDokumente();
    @WithDefault("30") int fristTage();
}
```

Profiles are `dev`, `test` and `prod`. Environment variables override (`QUARKUS_DATASOURCE_JDBC_URL`). Missing required config fails at startup.

## 1.5 Dev mode as a daily workflow

- `./mvnw quarkus:dev`: live reload on save, Dev UI at `/q/dev-ui`, press `r` for continuous testing.
- **Dev Services:** with no datasource URL configured, a PostgreSQL container starts automatically. The same goes for Keycloak (OIDC), with a realm and test users. (The DV Bern repo also uses Docker Compose for a local stack, so expect both.)
- **Remote debugging:** dev mode listens on port 5005 by default.

## Session brief — Day 1 AM

**Goal:** be fluent in the Quarkus core model, so nothing in the DV Bern codebase looks like magic, and translate your Jakarta EE experience into Quarkus terms on the spot.

**Tasks** (≈ 2 h)

| # | Task | Time |
|---|---|---|
| T1 | Generate the project: `quarkus create app ch.example:stip-lab --java=25 --extension=rest-jackson,hibernate-orm-panache,jdbc-postgresql,hibernate-validator,config-yaml,smallrye-openapi`. Start `./mvnw quarkus:dev` and confirm Dev Services started PostgreSQL (check `docker ps`). | 20 min |
| T2 | Write `GesuchService` (`@ApplicationScoped`) with one method `String status(UUID id)` returning a dummy value, and a `GreetingResource`-style endpoint `GET /ping/{id}` that calls it. Change the code while dev mode runs and confirm live reload. | 20 min |
| T3 | Add `GesuchConfig` (`@ConfigMapping(prefix = "kstip.gesuch")`) with `maxDokumente` and `fristTage`. Set a `%dev` override in `application.yaml`. Expose the values via `GET /ping/config`. | 20 min |
| T4 | Break it on purpose: (a) add a second implementation of an interface you inject → observe the *ambiguous dependency* error on reload; (b) remove a required config value → observe the startup failure. Write down the error messages in one line each. | 20 min |
| T5 | Add a `@Startup` bean that logs the active profile and `fristTage`. Add a `@DefaultBean` `Clock` producer (you'll override it in tests on Day 2). | 15 min |
| T6 | Explore the Dev UI (`/q/dev-ui`): find your beans, the config editor, and the datasource panel. Switch the log level from the config editor. | 15 min |
| T7 | Write a half-page cheat sheet: "Jakarta EE → Quarkus", in your own words (EJB, timers, JNDI, server config, Arquillian). | 10 min |

**Expected outcome (definition of done)**
- [ ] `stip-lab` runs in dev mode with PostgreSQL started by Dev Services, no manual DB setup.
- [ ] `GET /ping/{id}` and `GET /ping/config` respond; the config value differs between `dev` and default profile.
- [ ] You've seen and can describe both a **build-time DI failure** and a **startup config failure**.
- [ ] The `@Startup` log line appears; a `@DefaultBean Clock` exists.
- [ ] Your own Jakarta EE → Quarkus cheat sheet exists (half a page, in the repo as `NOTES.md`).
- [ ] 5–7 Git commits, one per task.

**Self-check**
1. Explain build-time augmentation in 60 seconds, in German if you can. *(→ 1.1)*
2. Map `@Stateless` + `@TransactionAttribute(REQUIRED)` and `@Schedule` to Quarkus. *(→ 1.2)*
3. Name two CDI features ArC doesn't support and what you'd do instead. *(→ 1.3)*
4. Why can't `quarkus.datasource.db-kind` be changed by an environment variable in production? *(→ 1.1, 1.4)*
5. `@Singleton` vs `@ApplicationScoped`: what's the practical difference? *(→ 1.3)*

**If short on time:** T1, T3, T4 and self-check questions 1–3.

---

# DAY 1 — Afternoon: The API layer

## 2.1 Contract-first with OpenAPI Generator

The repo has a `contract/` folder and an `openapi-generator-maven-plugin` profile (`generate-stubs-from-contract`). The typical setup looks like this:

1. The OpenAPI YAML is the **source of truth**, shared with the Angular frontend (the frontend generates its TypeScript client from the same contract).
2. The generator (`jaxrs-spec`, `interfaceOnly=true`, `useJakartaEe=true`) creates **Jakarta REST interfaces + DTOs**.
3. You implement the interfaces in resource classes.

```java
// generated: GesuchResource interface with @Path/@GET/@POST and DTOs
@RequestScoped
@RequiredArgsConstructor               // Lombok
public class GesuchResourceImpl implements GesuchResource {

    private final GesuchService gesuchService;

    @Override
    @RolesAllowed(Roles.GESUCHSTELLER)
    public GesuchDto getGesuch(UUID gesuchId) {
        return gesuchService.findGesuch(gesuchId);
    }

    @Override
    @RolesAllowed(Roles.GESUCHSTELLER)
    public GesuchDto updateGesuch(UUID gesuchId, GesuchUpdateDto dto) {
        return gesuchService.updateGesuch(gesuchId, dto);
    }
}
```

**Contract-first vs code-first.** This is a good interview discussion point:
- *Contract-first:* the API is designed and reviewed independently, frontend and backend can work in parallel, there's no accidental breaking change, and both clients are generated. Downsides: the generator's quirks, templates to maintain (the repo has a `templates/` folder), and less flexibility.
- *Code-first:* faster to start, and SmallRye OpenAPI generates the spec from annotations. Downside: the spec drifts with implementation details.

**`jackson-databind-nullable` / `JsonNullable<T>`** distinguishes "field absent" from "field explicitly set to null" in PATCH-style updates. With a plain `String`, both arrive as `null`.

## 2.2 Quarkus REST essentials

`quarkus-rest` is Jakarta REST, the API you already know. The Quarkus-specific points:
- **Threading by signature:** plain return types run on a worker thread (blocking JDBC is fine). `Uni`/`Multi` run on the event loop, where you must not block. A business application with Hibernate ORM stays **imperative**, so you can largely ignore reactive for this role. Just know the rule.
- `@PathParam` / `@QueryParam` names can be omitted when they match the parameter name.
- **Multipart upload:**

```java
@POST @Path("/{gesuchId}/dokumente")
@Consumes(MediaType.MULTIPART_FORM_DATA)
public DokumentDto upload(UUID gesuchId, @RestForm("file") FileUpload file) { ... }
```

- **Exception mapping:**

```java
@ServerExceptionMapper
public RestResponse<ErrorDto> map(GesuchNotFoundException e) {
    return RestResponse.status(Response.Status.NOT_FOUND, new ErrorDto("GESUCH_NOT_FOUND", e.getMessage()));
}

@ServerExceptionMapper
public RestResponse<ErrorDto> map(OptimisticLockException e) {
    return RestResponse.status(Response.Status.CONFLICT, new ErrorDto("CONCURRENT_MODIFICATION", null));
}
```

Design consistent error codes. The Angular frontend needs to map them to translated messages (DE/FR; the repo uses an i18n/l10n library).

## 2.3 DTO mapping with MapStruct (+ Lombok)

Never expose entities through the API. Map at the service boundary:

```java
@Mapper(componentModel = MappingConstants.ComponentModel.JAKARTA_CDI,
        unmappedTargetPolicy = ReportingPolicy.ERROR)
public interface GesuchMapper {
    GesuchDto toDto(Gesuch gesuch);

    @Mapping(target = "id", ignore = true)
    @Mapping(target = "status", ignore = true)          // status only changes via the state machine
    void partialUpdate(GesuchUpdateDto dto, @MappingTarget Gesuch gesuch);
}
```

- `unmappedTargetPolicy = ERROR` makes a new field without a mapping fail the **compile**. That's good review hygiene.
- MapStruct and Lombok are both annotation processors. Processor order matters (the repo configures Lombok, MapStruct, QueryDSL, Checker and Jilt in `maven-compiler-plugin`), and you need `lombok-mapstruct-binding` when using both.
- **Lombok on entities:** avoid `@Data`/`@EqualsAndHashCode` on JPA entities. Generated `equals`/`hashCode`/`toString` over lazy associations cause lazy-loading surprises and broken `Set` semantics. Use `@Getter`/`@Setter`, and base equality on the ID or a business key.

## 2.4 Validation

- Bean Validation on DTOs (`@NotNull`, `@Size`, `@Email`, custom constraints) is triggered by `@Valid` on resource parameters. Violations become 400 automatically.
- **Business validation is different.** Rules like "the applicant must be at least 15", "income documents required if X" or "you can't submit without the parents' tax data" belong in the domain/service layer. They're often context-dependent, using **validation groups** (e.g. `@NotNull(groups = GesuchEinreichenValidationGroup.class)`) checked only when *submitting*, not when saving a draft. This pattern is very typical for long multi-step application forms.

```java
Set<ConstraintViolation<Gesuch>> violations =
        validator.validate(gesuch, GesuchEinreichenValidationGroup.class);
if (!violations.isEmpty()) throw new ValidationsException("Gesuch unvollständig", violations);
```

## 2.5 Security with Keycloak (OIDC) — developer view

```yaml
quarkus:
  oidc:
    auth-server-url: ${KEYCLOAK_URL}/realms/stip
    client-id: stip-api
```

- **Bearer token validation:** each request carries a JWT from Keycloak. Quarkus validates the signature/issuer and maps roles to `SecurityIdentity`.
- `@RolesAllowed("V0_SACHBEARBEITER")`, `@Authenticated`, `@PermitAll`.
- **Roles aren't enough.** "A Gesuchsteller may only read *their own* Gesuch" is **object-level authorisation**. Implement it in the service layer (e.g. compare the `sub` claim with the owner) or with `@PermissionsAllowed` + a custom permission checker. Expect this question.
- **`smallrye-jwt-build`** lets the app *issue* its own JWTs, e.g. short-lived signed tokens for download links or for system-to-system calls.
- **Keycloak admin client:** the app manages users/roles in Keycloak programmatically (e.g. creating case-worker accounts).
- **Testing:** `@TestSecurity(user = "hans", roles = "V0_GESUCHSTELLER")` + `@OidcSecurity(claims = @Claim(key = "sub", value = "..."))`.

## Session brief — Day 1 PM

**Goal:** build the API layer the way the DV Bern stack does it: contract-first, mapped, validated and secured. You should be able to discuss each design decision, not just write the code.

**Tasks** (≈ 2.5 h)

| # | Task | Time |
|---|---|---|
| T1 | Write `contract/gesuch-api.yaml` (OpenAPI 3): `GET /gesuch/{id}`, `POST /gesuch`, `PATCH /gesuch/{id}`, `POST /gesuch/{id}/einreichen`, plus an `ErrorDto` with `code` and `message`. Keep `GesuchDto` small (id, version, status, vorname, nachname, geburtsdatum, ausbildungsbeginn). | 25 min |
| T2 | Configure `openapi-generator-maven-plugin` (`jaxrs-spec`, `interfaceOnly=true`, `useJakartaEe=true`, `openApiNullable=true`). Generate, then implement `GesuchResourceImpl implements GesuchResource`. Store data in a temporary in-memory map (persistence comes tomorrow). | 30 min |
| T3 | Add MapStruct (`JAKARTA_CDI`, `unmappedTargetPolicy = ERROR`) + Lombok with `lombok-mapstruct-binding`. Map `GesuchUpdateDto` → domain object with `partialUpdate`, handling `JsonNullable` fields. Add a new DTO field without mapping it and confirm the **compile error**. | 30 min |
| T4 | Validation: Bean Validation on DTOs for format rules (400 on violation). Add a `GesuchEinreichenValidationGroup` requiring all fields only on `einreichen`. Saving an incomplete draft must succeed; submitting it must fail with a list of missing fields. | 25 min |
| T5 | Error handling: `@ServerExceptionMapper`s for `GesuchNotFoundException` (404), your validation exception (400 with field list) and an `IllegalStateException` for invalid status changes (409). All errors use `ErrorDto` with stable codes. | 15 min |
| T6 | Security: add `quarkus-oidc`, let Keycloak Dev Services start. Create two roles (`GESUCHSTELLER`, `SACHBEARBEITER`) and two test users. Protect the endpoints with `@RolesAllowed`. Implement **ownership**: a Gesuchsteller can only read/modify Gesuche whose `ownerSub` equals the token's `sub`; a Sachbearbeiter can read all. | 25 min |

**Expected outcome (definition of done)**
- [ ] The contract is the single source of truth; `mvn compile` regenerates interfaces and DTOs from it.
- [ ] All four endpoints work via Swagger UI (`/q/swagger-ui`) with a token from Keycloak Dev Services.
- [ ] PATCH distinguishes "field absent" from "field set to null" (demonstrated with two requests).
- [ ] A missing MapStruct mapping breaks the build.
- [ ] Draft save succeeds with missing fields; `einreichen` returns 400 with a structured list of missing fields.
- [ ] Error responses are consistent (`ErrorDto`, stable codes) for 400, 404 and 409.
- [ ] User A gets 403 (or 404, decide and justify) when accessing user B's Gesuch; a Sachbearbeiter can read both.
- [ ] A short `DECISIONS.md` with 3 bullets: why contract-first, 403 vs 404 for foreign Gesuche, where business validation lives.

**Self-check**
1. Argue for contract-first in a team with an Angular frontend, and name two downsides. *(→ 2.1)*
2. Why `JsonNullable` for PATCH? *(→ 2.1)*
3. Format validation vs business validation: where does each live, and why? *(→ 2.4)*
4. How do you implement "users only see their own data"? Roles alone aren't enough — why? *(→ 2.5)*
5. Which thread runs a Quarkus REST method returning `GesuchDto`, and which one runs a method returning `Uni<GesuchDto>`? *(→ 2.2)*
6. Why `@Data` is dangerous on JPA entities. *(→ 2.3)*

**If short on time:** T1, T2, T4, T6 (ownership check included) and self-check 1, 3, 4.

---

# DAY 2 — Morning: Domain and persistence

## 3.1 Hibernate ORM with Panache

Two styles. Know both, and expect repositories in a codebase like this:

```java
@Entity
@Audited                                   // Envers
@Getter @Setter
public class Gesuch extends AbstractMutableEntity {   // id (UUID), version, timestamps
    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private Gesuchstatus gesuchStatus;

    @ManyToOne(optional = false, fetch = FetchType.LAZY)
    private Ausbildung ausbildung;

    @OneToMany(mappedBy = "gesuch", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<GesuchTranche> tranchen = new ArrayList<>();
}

@ApplicationScoped
public class GesuchRepository implements PanacheRepositoryBase<Gesuch, UUID> {
    public Stream<Gesuch> findByStatus(Gesuchstatus status) {
        return find("gesuchStatus", status).stream();
    }
}
```

- `PanacheRepositoryBase<Entity, IdType>` is for non-`Long` IDs (UUIDs are common in this kind of system).
- **Panache vs Spring Data:** there are no derived query method names. You write short HQL (`find("gesuchStatus", s)`), use QueryDSL for complex queries, or use Jakarta Data.

## 3.2 Type-safe queries with QueryDSL

For search screens with many optional filters (very typical: case-worker dashboards), string concatenation of HQL is error-prone. QueryDSL generates `Q`-classes at compile time:

```java
public List<Gesuch> search(GesuchFilter f) {
    var gesuch = QGesuch.gesuch;
    var query = new JPAQueryFactory(entityManager)
            .selectFrom(gesuch)
            .join(gesuch.ausbildung).fetchJoin();

    if (f.status() != null)  query.where(gesuch.gesuchStatus.eq(f.status()));
    if (f.nachname() != null) query.where(gesuch.gesuchsteller.nachname.containsIgnoreCase(f.nachname()));
    if (f.eingereichtAb() != null) query.where(gesuch.eingereicht.goe(f.eingereichtAb()));

    return query.orderBy(gesuch.eingereicht.desc())
                .offset(f.page() * f.size()).limit(f.size())
                .fetch();
}
```

Know the alternatives: the JPA Criteria API (verbose), jOOQ (SQL-first), Blaze-Persistence. Note that the repo uses the **OpenFeign fork** of QueryDSL (`io.github.openfeign.querydsl`), because the original project stagnated.

## 3.3 Transactions and consistency

- `@jakarta.transaction.Transactional` (Narayana JTA), the same semantics you know: `REQUIRED` by default. Runtime exceptions roll back; checked exceptions don't, unless you use `rollbackOn`.
- **Put the boundary on the service method**, not on the repository or the resource.
- Programmatic: `QuarkusTransaction.requiringNew().run(() -> ...)`. Useful in a batch job to commit each item separately, so one failure doesn't roll back 5,000 records.
- **Map to DTOs inside the transaction.** Don't rely on lazy loading during JSON serialization.

**Optimistic locking (very important for case-worker applications):**

```java
@Version
private long version;
```

Two case workers edit the same Gesuch, and the second save gets an `OptimisticLockException`, which you map to 409. The client must send the version it loaded (in the DTO or as an `If-Match` ETag). Expect a question like "What happens if two Sachbearbeiter edit the same Gesuch?"

**Pessimistic locking** (`LockModeType.PESSIMISTIC_WRITE` → `SELECT … FOR UPDATE`) is for rare, short critical sections, e.g. allocating a sequential case number.

## 3.4 Auditing with Hibernate Envers

Public-sector software must answer the question "who changed what, when?". Envers does this transparently:

- `@Audited` on entities creates `*_AUD` tables plus a `REVINFO` table. Each committed transaction is one revision.
- A custom **revision entity** + `RevisionListener` stores the user. Get the current user from `SecurityIdentity` (in Quarkus, look it up via `Arc.container().instance(...)` inside the listener).
- Query history with `AuditReaderFactory.get(entityManager)`: e.g. "state of the Gesuch at the time of the decision", or "list all changes since submission".
- `@NotAudited` for large or irrelevant fields; `@Audited(targetAuditMode = NOT_AUDITED)` for references to non-audited master data.
- **Cost:** extra writes on every change, plus the `_AUD` tables grow and need indexes. Schema migrations must also update the `_AUD` tables (Liquibase changesets for both).

Envers keeps a technical change history. A *business* history (e.g. a status log with comments shown in the UI) is often modelled explicitly as its own entity. Be ready to discuss the difference.

## 3.5 Schema migrations with Liquibase

```yaml
quarkus:
  liquibase:
    migrate-at-start: true
    change-log: db/changelog/db.changelog-master.xml
```

- Changesets are immutable once applied. A fix is a *new* changeset.
- The repo includes `liquibase-maven-plugin` with `liquibase-hibernate6`. That's used to **diff entities against the DB and generate changesets**. Always review generated SQL by hand (renames show up as drop + add).
- **Zero-downtime discipline (expand/contract):** add a nullable column → deploy → backfill → add the constraint → remove the old column later. Old and new app versions must be able to run against the same schema during a rolling deploy.
- `migrate-at-start` with several replicas: Liquibase uses a DB lock table (`DATABASECHANGELOGLOCK`). The repo's docs mention troubleshooting "DB lock problems", so know what a stuck lock is and how to release it.
- **Liquibase vs Flyway:** Liquibase has DB-agnostic changesets (XML/YAML), rollback definitions, preconditions and diff tooling. Flyway is plain versioned SQL, simpler. Both are fine. Spring Boot and Quarkus support both.

## 3.6 PostgreSQL you should be fluent in

The ad says "fundierte Kenntnisse in Postgres, SQL". Expect some live SQL.

- **Query tuning:** `EXPLAIN (ANALYZE, BUFFERS)`; reading seq scan vs index scan, nested loop vs hash join; why the planner ignores an index (low selectivity, function on the column, type mismatch, stale statistics → `ANALYZE`).
- **Indexes:** B-tree default; composite index column order; **partial indexes** (`WHERE status = 'EINGEREICHT'`); expression indexes (`lower(nachname)`); GIN for JSONB/full-text; foreign-key columns are **not** auto-indexed in Postgres.
- **SQL:** CTEs (`WITH`), window functions (`row_number() over (partition by gesuch_id order by gueltig_ab desc)` for "latest record per group"), `DISTINCT ON`, `UPSERT` (`INSERT … ON CONFLICT`), `FILTER` in aggregates.
- **Concurrency:** MVCC; the default isolation is `READ COMMITTED`; `SELECT … FOR UPDATE SKIP LOCKED` for work queues; deadlocks and consistent lock ordering.
- **Types:** `uuid`, `timestamptz` (always, for instants), `numeric` for money (**never** `double` for CHF amounts; use `BigDecimal` in Java), `jsonb` (`@JdbcTypeCode(SqlTypes.JSON)` in Hibernate 6/7).
- **Hibernate ↔ SQL pitfalls:**
  - **N+1 selects:** fix with `JOIN FETCH`, entity graphs, or batch fetching (`quarkus.hibernate-orm.fetch.batch-size`). Turn on SQL logging in dev to see it.
  - **Pagination + collection fetch join** paginates in memory (Hibernate warns, HHH90003004). Paginate on IDs first, then fetch.
  - **Bulk updates:** use HQL/SQL `update` statements for mass changes instead of loading thousands of entities, but remember they bypass Envers and the persistence context.

## 3.7 Business workflow as a state machine (stateless4j)

A Gesuch moves through states (illustrative: `IN_BEARBEITUNG_GS → EINGEREICHT → IN_BEARBEITUNG_SB → VERFUEGT → …`). The repo uses **stateless4j** to make allowed transitions explicit instead of scattering `if (status == …)` checks around:

```java
var config = new StateMachineConfig<Gesuchstatus, GesuchStatusTrigger>();

config.configure(Gesuchstatus.IN_BEARBEITUNG_GS)
      .permit(GesuchStatusTrigger.EINREICHEN, Gesuchstatus.EINGEREICHT);

config.configure(Gesuchstatus.EINGEREICHT)
      .permit(GesuchStatusTrigger.BEARBEITUNG_STARTEN, Gesuchstatus.IN_BEARBEITUNG_SB)
      .onEntry(t -> notificationService.sendEingangsbestaetigung(...));

config.configure(Gesuchstatus.IN_BEARBEITUNG_SB)
      .permitIf(GesuchStatusTrigger.VERFUEGEN, Gesuchstatus.VERFUEGT, () -> pruefungAbgeschlossen())
      .permit(GesuchStatusTrigger.ZURUECKWEISEN, Gesuchstatus.IN_BEARBEITUNG_GS);

var sm = new StateMachine<>(gesuch.getGesuchStatus(), config);
sm.fire(GesuchStatusTrigger.EINREICHEN);       // throws IllegalStateException if not permitted
gesuch.setGesuchStatus(sm.getState());
```

Talking points:
- Transitions are **documented in code**, testable in isolation, and can be visualised (stateless4j can export DOT graphs).
- Side effects (mail, PDF generation, audit entries) go on entry/exit actions, but **inside the transaction**. Mail sending after commit is a separate question; see the outbox pattern below.
- **Alternatives:** an enum with transition methods (simplest), Spring Statemachine, or a BPMN engine (Camunda/Flowable) when business users need to change processes. Know when each one is overkill.

## 3.8 Architecture and modelling

- **Layering:** resource → service (transactions, business rules) → repository → entity, with DTO mapping at the service boundary. **ArchUnit** enforces it:

```java
@AnalyzeClasses(packages = "ch.dvbern.stip")
class ArchitectureTest {
    @ArchTest
    static final ArchRule layers = layeredArchitecture().consideringAllDependencies()
        .layer("Resource").definedBy("..resource..")
        .layer("Service").definedBy("..service..")
        .layer("Repo").definedBy("..repo..")
        .whereLayer("Resource").mayNotBeAccessedByAnyLayer()
        .whereLayer("Service").mayOnlyBeAccessedByLayers("Resource", "Service")
        .whereLayer("Repo").mayOnlyBeAccessedByLayers("Service");
}
```

- **Package by feature** (`gesuch/`, `ausbildung/`, `dokument/`, `verfuegung/`, each with resource/service/repo inside) usually beats package by layer in a large business application.
- **Modern Java (they're on Java 25):** records for DTOs/value objects, sealed interfaces + pattern-matching `switch` for domain variants (e.g. different calculation inputs), `BigDecimal` with explicit `RoundingMode` for money. Calculation logic (scholarship amounts) should be **pure functions**: input object in, result object out, no DB access. That makes them trivially unit-testable with table-driven tests.
- **Builders:** Jilt generates type-safe builders for complex objects. That's useful in tests (test data builders).
- **Null safety:** the Checker Framework `@Nullable` + Sonar rules.

## Session brief — Day 2 AM

**Goal:** replace yesterday's in-memory storage with a production-grade persistence and domain layer: versioned, audited, migrated, queryable and governed by an explicit state machine. This is the core of "zuverlässige Geschäftslogik" and "fundierte Kenntnisse in Postgres".

**Tasks** (≈ 3 h)

| # | Task | Time |
|---|---|---|
| T1 | **Schema with Liquibase.** Turn off Hibernate schema generation. Write changesets for `gesuch` (uuid PK, `version`, `status`, `owner_sub`, personal fields, `eingereicht timestamptz`), `gesuch_tranche` (FK to gesuch, `gueltig_ab`, `betrag numeric(12,2)`), and Envers' `revinfo` + `gesuch_aud` + `gesuch_tranche_aud`. Add an index on the FK column. | 35 min |
| T2 | **Entities and repository.** Map `Gesuch` and `GesuchTranche` (`@Version`, `@Audited`, lazy `@ManyToOne`, `BigDecimal` for amounts). Create `GesuchRepository implements PanacheRepositoryBase<Gesuch, UUID>`. Move yesterday's resource onto a `@Transactional` service that maps to DTOs *inside* the transaction. | 30 min |
| T3 | **State machine.** Model 5 states and their triggers with stateless4j (e.g. `IN_BEARBEITUNG_GS → EINGEREICHT → IN_BEARBEITUNG_SB → VERFUEGT`, plus `ZURUECKWEISEN` back to the applicant). Status may only change through the state machine. Write a `@ParameterizedTest` covering every allowed and at least 4 forbidden transitions. | 35 min |
| T4 | **Optimistic locking.** The client sends `version` with every PATCH. Write a test where two updates use the same version: the second must return **409** `CONCURRENT_MODIFICATION`. | 20 min |
| T5 | **Auditing.** Add a custom revision entity storing the username (from `SecurityIdentity`). Write `GET /gesuch/{id}/historie` returning revision number, timestamp, user and status per revision. | 25 min |
| T6 | **Search.** Add QueryDSL. Build `GET /gesuch?status=&nachname=&eingereichtAb=&page=&size=` with all filters optional, sorted by `eingereicht desc`. Turn on SQL logging, provoke an N+1 by returning tranches in the result, then fix it (fetch join or batch size), avoiding the "pagination + collection fetch" trap. | 25 min |
| T7 | **SQL drill** in `psql` (insert ~10k rows with `generate_series`): (a) latest tranche per Gesuch with `row_number()` and with `DISTINCT ON`; (b) `EXPLAIN (ANALYZE, BUFFERS)` of a search on `lower(nachname)` before and after an expression index; (c) a partial index for `status = 'EINGEREICHT'`. | 20 min |
| T8 | **Architecture.** Reorganise into package-by-feature (`gesuch/{resource,service,repo,entity}`) and add one ArchUnit layering test. | 10 min |

**Expected outcome (definition of done)**
- [ ] The app starts on an empty DB, and Liquibase creates the full schema including `_AUD` tables; Hibernate validation passes (`quarkus.hibernate-orm.database.generation=validate`).
- [ ] No entity leaves the service layer; every endpoint returns DTOs.
- [ ] Status changes happen only via the state machine; the transition test covers all allowed transitions and ≥ 4 forbidden ones, and runs in under a second (no Quarkus boot).
- [ ] Concurrent modification returns 409, proven by a test.
- [ ] `/historie` shows one entry per committed change, with the user name.
- [ ] Search works with any filter combination; the SQL log shows a bounded number of queries per page (no N+1).
- [ ] A `SQL.md` with your three queries plus before/after `EXPLAIN` output and one sentence each on what changed.
- [ ] The ArchUnit test passes, and fails if a resource accesses a repository directly (try it).

**Self-check**
1. Optimistic vs pessimistic locking: when do you use each? What does the client have to do for optimistic locking to work? *(→ 3.3)*
2. Design a Liquibase migration for a new mandatory column during a rolling deployment. *(→ 3.5)*
3. Envers vs an explicit business history table: what's each one for? *(→ 3.4)*
4. Why a state machine instead of status `if`s, and when would a BPMN engine be better? *(→ 3.7)*
5. Explain N+1 and three ways to fix it. Why is pagination with a collection fetch join dangerous? *(→ 3.6)*
6. Why `timestamptz` and `numeric`/`BigDecimal`? Why must FK columns get indexes explicitly in Postgres? *(→ 3.6)*

**If short on time:** T1–T4 and T6 (without the N+1 part), and self-check 1, 2, 4, 5.

---

# DAY 2 — Afternoon: Interfaces, automation, testing

## 4.1 Outbound REST

```java
@RegisterRestClient(configKey = "steuerdaten")
@Path("/api/v1")
public interface SteuerdatenClient {
    @GET @Path("/steuerdaten/{ahvNr}")
    SteuerdatenDto getSteuerdaten(@PathParam("ahvNr") String ahvNr);
}
```

```yaml
quarkus:
  rest-client:
    steuerdaten:
      url: ${STEUERDATEN_URL}
      connect-timeout: 2000
      read-timeout: 5000
```

(The service above is illustrative.) For resilience, add **timeouts always**. Add retries only for idempotent calls, and consider SmallRye Fault Tolerance (`@Retry`, `@CircuitBreaker`, `@Fallback`). Test with WireMock.

## 4.2 SOAP with Quarkus CXF

Swiss public-sector integrations frequently use SOAP/XML (eCH standards, cantonal registries, sedex-style exchanges). The repo includes `quarkus-cxf`, JAXB and SAAJ. With Quarkus CXF:
- **WSDL → Java at build time:** `quarkus.cxf.codegen.wsdl2java.includes=wsdl/*.wsdl`.
- **Inject a client:**

```java
@CXFClient("personenregister")
PersonenregisterPort port;
```

```yaml
quarkus:
  cxf:
    client:
      personenregister:
        client-endpoint-url: ${REGISTER_URL}
        service-interface: ch.example.register.PersonenregisterPort
```

- WS-Security, mTLS and logging interceptors are all config-driven. For testing, use a mock endpoint or a WireMock SOAP stub.
- **Interview angle:** when *you* design an interface, you'd prefer REST/JSON + OpenAPI. When integrating with existing government systems, you take what they offer (often SOAP/XML), and you isolate it behind an **anti-corruption layer** (an adapter that maps their XML model to your domain).

## 4.3 Automating administrative processes

**Scheduled jobs with Quartz:**

```java
@Scheduled(cron = "{kstip.jobs.fristen.cron}", identity = "fristenpruefung",
           concurrentExecution = Scheduled.ConcurrentExecution.SKIP)
void fristenPruefen() {
    gesuchService.findAbgelaufeneFristen()
                 .forEach(id -> QuarkusTransaction.requiringNew().run(() -> gesuchService.mahnen(id)));
}
```

```yaml
quarkus:
  quartz:
    store-type: jdbc-cmt     # job state in Postgres
    clustered: true          # with several pods, each trigger fires on exactly one
```

- **Why Quartz rather than plain `quarkus-scheduler`?** Persistence (misfire handling after downtime) and **clustering**: with several replicas, a plain `@Scheduled` job runs on *every* pod.
- **Job design rules:** idempotent (safe to re-run), one transaction per item, log or record failures per item instead of aborting the batch, and expose a manual trigger for operations.

**E-mail with Mailer + Qute templates:**

```java
@CheckedTemplate
static class Templates {
    static native MailTemplate.MailTemplateInstance verfuegungBereit(String vorname, String gesuchNummer);
}

Templates.verfuegungBereit(vorname, nummer)
         .to(email).subject("Ihre Verfügung ist verfügbar")
         .send().await().indefinitely();
```

- Mail is **mocked by default in dev and test**. `MockMailbox` lets you assert on sent mails.
- **Mail after commit:** if the transaction rolls back after the mail went out, the user got a wrong mail. Options: send in an `@Observes(during = TransactionPhase.AFTER_SUCCESS)` event observer, or use an **outbox table** processed by a job (reliable, retryable). This is a great senior-level talking point.
- Templates in DE/FR; locale from the user profile.

**Generated documents (PDF) with iText:**
- Decision letters (*Verfügungen*), confirmations and reports. Build them in a dedicated service from a DTO, not from entities.
- Store the generated PDF (S3) and reference it, since a decision document must be reproducible exactly as sent.
- **Licensing note:** iText core is AGPL or commercial. The repo is AGPL, which fits. Mentioning this shows maturity.

**File uploads (applicant documents):**
- Multipart → **detect the real content type with Apache Tika** (don't trust the extension or the `Content-Type` header) → **virus scan** (the Quarkiverse antivirus extension, ClamAV) → store in **S3-compatible storage** → keep only metadata in Postgres.
- Enforce size limits (`quarkus.http.limits.max-body-size`), and allow only certain MIME types.

**Imports:** an Excel import (fastexcel reader) for master data or bulk updates. Validate each row, collect errors, and either commit all or nothing, or commit valid rows plus an error report. Decide explicitly which.

## 4.4 Testing strategy

**The pyramid as it maps onto this stack:**

| Level | Tool | What to test |
|---|---|---|
| Pure unit | JUnit 5 + AssertJ (+ Mockito) | Calculations, state machine, mappers, validators. Fast, many, table-driven (`@ParameterizedTest`) |
| CDI component | **`@QuarkusComponentTest`** (`quarkus-junit-component`) | One bean with real CDI wiring; dependencies auto-mocked; config via `@TestConfigProperty`. No full app boot |
| Integration | **`@QuarkusTest`** + Testcontainers/Dev Services Postgres + RestAssured | REST → service → DB round trips, security, transactions, Liquibase migrations, Envers |
| External systems | WireMock / OIDC test server / LocalStack (S3) | REST/SOAP clients, Keycloak tokens, file storage |
| Architecture | **ArchUnit** | Layering, naming, "no entity in API package" |
| Contract | JSON comparison (`json-comparator`) / generated-client tests | API responses match the contract |

```java
@QuarkusTest
@TestSecurity(user = "sb1", roles = Roles.SACHBEARBEITER)
class GesuchResourceTest {

    @Test
    void verfuegen_moves_status_and_sends_mail(MockMailbox mailbox) { ... }  // or @Inject MockMailbox

    @Test
    void cannot_verfuegen_from_wrong_status() {
        given().when().post("/gesuch/{id}/verfuegen", draftId)
               .then().statusCode(400).body("code", is("INVALID_STATUS_TRANSITION"));
    }
}
```

**Quarkus testing specifics to know:**
- `@QuarkusTest` boots the app **once** and reuses it. `@TestProfile` forces a restart, so use it sparingly.
- `@InjectMock` replaces a bean (needs a normal scope like `@ApplicationScoped`).
- `@TestTransaction` rolls back after each test. Alternatively, clean the DB between tests. Discuss the trade-off: rollback is fast but hides commit-time behaviour, such as constraint checks at flush, Envers revisions, and AFTER_SUCCESS observers.
- **Test data:** builders (Jilt), or a small seed via Liquibase contexts for tests.
- **Coverage:** `quarkus-jacoco` produces reports for `@QuarkusTest`, which feed **Sonar**. A coverage *gate* on new code is common. Aim for meaningful tests of business rules, not line coverage for its own sake.

## 4.5 Code quality and code reviews

- **Formatting is automated** (Spotless with an Eclipse formatter). Reviews should never be about formatting.
- **Sonar:** quality gate on new code (bugs, vulnerabilities, code smells, coverage, duplication). Know how to read an issue and when to mark it a false positive with a justification.
- **Renovate** opens dependency-update merge requests. Reviewing them means reading changelogs and Quarkus migration guides (`quarkus update` for LTS jumps).
- **What *you* look for in a review (have 5–6 points ready):**
  1. Transaction boundaries and lazy loading outside them
  2. Authorisation: object-level checks, not just roles
  3. Business rule placement (service/domain, not resource) and test coverage of edge cases
  4. Migration safety (backward-compatible schema, `_AUD` tables, data backfills)
  5. Query efficiency (N+1, missing indexes, unbounded result sets)
  6. Error handling and user-facing error codes; no swallowed exceptions
  7. Naming in the domain language (German terms are fine if that's the team convention; consistency matters more)

## Session brief — Day 2 PM

**Goal:** cover the other half of the job ad: interfaces to external systems, automated administrative processes, and a test suite that would pass a code review and a Sonar gate. Finish with an end-to-end flow: submit → review → decide → PDF + mail.

**Tasks** (≈ 3 h)

| # | Task | Time |
|---|---|---|
| T1 | **REST client.** Create `SteuerdatenClient` (`@RegisterRestClient`, connect timeout 2 s, read timeout 5 s). Call it when a Sachbearbeiter starts processing, and store a flag "Steuerdaten vorhanden". Test it with WireMock via `@WithTestResource`: success, 404 (no data, which is OK) and timeout (a clear error). | 25 min |
| T2 | **SOAP (optional, stretch).** Generate a CXF client from any public sample WSDL with `quarkus.cxf.codegen.wsdl2java`, inject it with `@CXFClient`, and wrap it in an adapter that returns a domain record. *Skip if time is short; being able to explain it is enough.* | 20 min |
| T3 | **Quartz job.** Add `quarkus-quartz` with the JDBC store (Quartz tables via Liquibase) and `clustered: true`. Job `fristenPruefen` (cron from config, `ConcurrentExecution.SKIP`): finds Gesuche sent back to the applicant (via `ZURUECKWEISEN`, so status `IN_BEARBEITUNG_GS` with a `frist` set) whose deadline has passed, sets a `gemahnt` flag, and sends a reminder. One transaction per Gesuch; a failure on one must not stop the others. Inject the `Clock` from Day 1 so tests can control "now". | 30 min |
| T4 | **Mail after commit.** Qute `@CheckedTemplate` for "Verfügung bereit" (DE). Send it from an observer with `@Observes(during = TransactionPhase.AFTER_SUCCESS)` on a `GesuchVerfuegtEvent`. Test with `MockMailbox`: mail is sent after a successful `verfuegen`, and **not** sent when the transaction rolls back (force a failure). | 25 min |
| T5 | **PDF + storage.** On `VERFUEGT`, generate a one-page iText PDF (name, Gesuch number, amount in CHF, date) from a DTO. Store it in S3 (LocalStack Testcontainer in tests); save only the object key in Postgres. Add `GET /gesuch/{id}/verfuegung` to download it. | 30 min |
| T6 | **Upload hardening (light).** `POST /gesuch/{id}/dokumente` with `@RestForm FileUpload`: max 10 MB, Tika detection, only PDF/JPEG/PNG allowed regardless of file extension; store in S3. (Antivirus: mention it in `DECISIONS.md`; no need to run ClamAV.) | 20 min |
| T7 | **Test suite.** Make sure you have, at minimum: 1 pure unit test class (calculation or state machine), 1 `@QuarkusComponentTest` (e.g. the job with mocked repository and fixed `Clock`), 3 `@QuarkusTest` classes with `@TestSecurity` (resource happy path, ownership violation, end-to-end flow), plus the ArchUnit test. Enable `quarkus-jacoco` and open the report. | 25 min |
| T8 | **Self code review.** Go through the checklist in 4.5 against your own code. Fix the two worst findings, and list the rest in `REVIEW.md`. | 10 min |

**Expected outcome (definition of done)**
- [ ] **End-to-end test** passes: applicant creates and submits a Gesuch → Sachbearbeiter starts processing (tax data fetched from WireMock) → verfügt → PDF stored in S3 (LocalStack) → exactly one mail in `MockMailbox`.
- [ ] REST client timeouts are configured and tested; a timeout produces a defined error code, not a stack trace.
- [ ] The Quartz job is idempotent (running it twice doesn't send two reminders), processes items in separate transactions, and is tested with a fixed `Clock`.
- [ ] No mail is sent when the surrounding transaction rolls back, proven by a test.
- [ ] The PDF downloads and opens; Postgres holds only the reference.
- [ ] Uploads with a renamed `.exe` → `.pdf` are rejected.
- [ ] `mvn verify` runs all test levels green in a reasonable time (target: under 2 minutes); the JaCoCo report shows coverage of the service and domain packages.
- [ ] `REVIEW.md` with your self-review findings, and `DECISIONS.md` updated (mail-after-commit vs outbox, antivirus, Quartz vs plain scheduler).

**Self-check**
1. A nightly job must run exactly once with 3 pods. How? *(→ 4.3)*
2. How do you guarantee a mail is only sent if the transaction commits, and when would you need an outbox instead? *(→ 4.3)*
3. How would you integrate a cantonal SOAP service, and what's an anti-corruption layer? *(→ 4.2)*
4. Walk through your test pyramid: what do you test at each level, and why not everything with `@QuarkusTest`? *(→ 4.4)*
5. `@TestTransaction` rollback vs cleaning the DB: what does rollback hide? *(→ 4.4)*
6. Name five things you check in a code review on this kind of system. *(→ 4.5)*

**If short on time:** T1, T3, T4, T7 (the end-to-end test can skip the PDF), and self-check 1, 2, 4, 6.

---

## Wrap-up session (≈ 1.5 h, evening of Day 2 or the morning before the interview)

**Goal:** turn what you built into fluent interview answers, in German.

**Tasks**
1. Read section 5. Say the "one-paragraph answer" out loud in German, from memory, twice. (15 min)
2. Answer all 12 drill questions in section 6 out loud in German, referring to *your own* `stip-lab` code ("in meinem Übungsprojekt habe ich …"). Record yourself on your phone for at least 4 of them, and listen back. (45 min)
3. Pick 3 of the "questions you can ask them" and phrase them in German. (10 min)
4. Learn the glossary in section 7: cover the English column and test yourself. (20 min)

**Expected outcome**
- [ ] You can give the Quarkus-vs-Spring paragraph in German without notes, in under 90 seconds.
- [ ] Each drill answer takes 1–2 minutes and includes one concrete example from your project.
- [ ] 3 questions for the interviewers, ready in German.
- [ ] You know the glossary terms both ways.

---

# 5. Quarkus vs Spring Boot — for a business application like kiBon Stip

Performance (startup, memory, native) matters much less for a case-management system than for serverless functions. So the honest comparison is about **programming model, ecosystem and team fit**.

| Concern | Quarkus (DV Bern stack) | Spring Boot equivalent | Verdict |
|---|---|---|---|
| Programming model | Jakarta EE standards (CDI, JAX-RS, JPA, JTA) | Spring IoC, Spring MVC, Spring Data | Quarkus is natural for EE teams; Spring for Spring teams |
| Error detection | DI and config errors fail the build | Fail at startup (or AOT build) | Quarkus slightly better |
| Dev loop | Live reload, Dev Services (Postgres, Keycloak), continuous testing | DevTools restart, Testcontainers `@ServiceConnection`, Docker Compose | Quarkus smoother out of the box |
| Persistence | Hibernate + Panache; QueryDSL/Envers/Liquibase as libraries | Spring Data JPA (derived queries, Specifications, `@Query`), same Hibernate/Envers/QueryDSL/Liquibase | Spring Data has more convenience features; underlying tech is identical |
| Contract-first REST | OpenAPI Generator `jaxrs-spec` | OpenAPI Generator `spring` | Equal |
| SOAP | Quarkus CXF | Spring Web Services or CXF Spring Boot starter | Equal; CXF is the same engine |
| Scheduling / batch | Quartz extension (clustered) | `@Scheduled`, Quartz, **Spring Batch** (chunking, restartability, job repository) | Spring Batch is stronger for heavy batch workloads |
| Security | `quarkus-oidc` + `@RolesAllowed`, simple config | Spring Security (OAuth2 resource server); far more flexible, more complex | Quarkus simpler; Spring more powerful for complex cases |
| Mail / templates | Mailer + Qute (type-checked templates at build time) | JavaMailSender + Thymeleaf/Freemarker | Qute's build-time type checking is a nice plus |
| Testing | `@QuarkusTest` (one shared boot), `@QuarkusComponentTest` | `@SpringBootTest` (context caching), slice tests (`@WebMvcTest`, `@DataJpaTest`) | Spring's slice tests are richer; Quarkus component tests are catching up |
| Ecosystem / community | Smaller, solid for this stack | Largest in Java; more answers and libraries | Spring |
| Upgrades | Fast cadence; LTS every 6 months (12-month support) → plan two upgrades per year | 6-month minors, ~12-month OSS support | Similar effort; Quarkus needs discipline about staying on LTS |
| Footprint | Lower memory/startup → cheaper OpenShift pods, faster rollouts | Higher, but improving (AOT, CDS, virtual threads) | Quarkus |

**A one-paragraph answer you can give in the interview:**

> For a Fachanwendung like kiBon Stip, the deciding factors aren't native images but productivity and maintainability. Quarkus gives a Jakarta EE team the standards it knows, catches wiring and config errors at build time, and has a very fast dev loop with Dev Services for Postgres and Keycloak. The persistence and integration stack (Hibernate, Envers, Liquibase, QueryDSL, CXF) is the same as you'd use with Spring. Spring would offer more convenience in places (Spring Data derived queries, Spring Batch, Spring Security's depth) and a bigger community. Quarkus's lower footprint is a real but secondary benefit on OpenShift. The main operational cost of Quarkus is keeping up with its release train, which is why staying on LTS lines matters.

**Where Quarkus can hurt in this kind of project (be honest if asked):**
- Fewer ready-made answers for edge cases (e.g. Envers + CDI in a revision listener, complex Keycloak setups).
- No Spring Batch equivalent. Large batch processing needs custom design (Quartz + chunked transactions + restart logic).
- Build-time config means some settings can't vary per environment without a rebuild. This is rarely a problem for an application like this.

---

# 6. Interview drill — development questions with short model answers

1. **"Wie strukturieren Sie Geschäftslogik in einer Quarkus-Anwendung?"** (How do you structure business logic?) Package by feature; thin resources; services own transactions and business rules; pure calculation classes without I/O; a state machine for status transitions; DTO mapping at the service boundary with MapStruct; ArchUnit to enforce it.

2. **Two case workers edit the same Gesuch.** `@Version` optimistic locking; the client sends the version back; `OptimisticLockException` → 409 with a clear error code; the UI tells the user to reload.

3. **How do you audit changes?** Hibernate Envers with a custom revision entity carrying the user. Use an explicit business history for user-visible status logs. Remember `_AUD` tables in migrations, and that bulk HQL updates bypass Envers.

4. **How do you design a migration for a new mandatory field?** Expand/contract: add it nullable → deploy code that writes it → backfill → add `NOT NULL` → (later) remove the old structures. Never edit applied changesets.

5. **A query is slow in production.** Reproduce with realistic data; `EXPLAIN (ANALYZE, BUFFERS)`; check for N+1 in the Hibernate log; check indexes (including FK columns, partial/composite); check pagination with fetch joins; `ANALYZE` statistics.

6. **A nightly job must run exactly once with 3 pods.** Quartz with a JDBC job store and `clustered: true`; the job is idempotent with per-item transactions; failures are recorded per item; there's a manual re-trigger.

7. **Mail must only be sent if the transaction commits.** A CDI observer with `TransactionPhase.AFTER_SUCCESS`, or a transactional outbox processed by a job for guaranteed delivery and retries.

8. **How do you integrate a SOAP service from a cantonal system?** Quarkus CXF with build-time wsdl2java; an adapter/anti-corruption layer mapping XML to the domain; timeouts; mTLS/WS-Security via config; tests against a stubbed endpoint.

9. **Contract-first or code-first?** Contract-first when frontend and backend teams work in parallel, or for external consumers. The spec is reviewed like code, and both sides generate from it. Code-first for internal, fast-moving APIs.

10. **How do you test this?** Unit tests for calculations and the state machine; `@QuarkusComponentTest` for single beans; `@QuarkusTest` with real Postgres (Testcontainers/Dev Services) for REST + DB + security; WireMock/LocalStack for externals; ArchUnit; JaCoCo → Sonar gate on new code.

11. **Object-level security.** Roles via `@RolesAllowed`, plus ownership checks in the service (the token `sub` vs the owner), or `@PermissionsAllowed` with a custom checker. Always tested with `@TestSecurity` / `@OidcSecurity`.

12. **Why Quarkus instead of Spring Boot here?** See the paragraph in section 5.

**Questions *you* can ask them (shows seniority):**
- How do you model the Gesuch lifecycle: stateless4j only, or also a business status history?
- Is the OpenAPI contract owned by the backend or shared with the frontend team? How do you handle breaking changes?
- How do you handle the yearly changes in cantonal scholarship rules: versioned calculation logic, configuration, or code releases?
- Which external interfaces exist (tax data, population registers, payment/accounting systems)? REST or SOAP?
- How do you approach Quarkus LTS upgrades, and how much time does Renovate maintenance take?
- What does a typical code review focus on, and what's the Sonar gate?

---

# 7. German glossary (domain + technical) for the interview

| Deutsch | English / meaning |
|---|---|
| Gesuch / Gesuchsteller:in | Application / applicant |
| Stipendium / Darlehen | Scholarship (grant) / loan |
| Ausbildungsbeiträge | Education grants (umbrella term for both) |
| Sachbearbeiter:in | Case worker |
| Verfügung | Formal administrative decision (legally binding letter) |
| Einreichen / Einreichung | To submit / submission |
| Frist / Fristablauf | Deadline / expiry of deadline |
| Mahnung | Reminder (for missing documents or deadlines) |
| Tranche | Instalment / period segment |
| Fachanwendung | Line-of-business application |
| Geschäftslogik / Fachlogik | Business logic |
| Schnittstelle | Interface (API) |
| Anforderungen / fachliche Anforderungen | Requirements / business requirements |
| Abnahme | Acceptance (testing) |
| Datenbankmigration | Database migration |
| Nebenläufigkeit / gleichzeitige Bearbeitung | Concurrency / simultaneous editing |
| Optimistisches Sperren | Optimistic locking |
| Nachvollziehbarkeit / Revisionssicherheit | Traceability / audit-proof record keeping |
| Berechtigungskonzept | Authorisation concept |
| Testabdeckung | Test coverage |
| Schichtenarchitektur | Layered architecture |
| Zustandsautomat / Zustandsübergang | State machine / state transition |
| Ablauf automatisieren | Automate a process |
| Stapelverarbeitung | Batch processing |

**Useful phrases:**
- *"Quarkus verlagert die Initialisierung des Frameworks in die Build-Zeit. Dadurch werden Fehler früher erkannt und die Anwendung startet schneller."* (Quarkus moves framework initialisation to build time, so errors are caught earlier and the app starts faster.)
- *"Die Geschäftsregeln gehören in die Service-Schicht, nicht in die REST-Ressource."* (Business rules belong in the service layer, not the REST resource.)
- *"Bei gleichzeitiger Bearbeitung setzen wir auf optimistisches Sperren mit einem Versionsfeld."* (For simultaneous edits we use optimistic locking with a version field.)
- *"Die Migration muss rückwärtskompatibel sein, damit alte und neue Version parallel laufen können."* (The migration must be backward-compatible so old and new versions can run in parallel.)
- *"Ich habe mir Ihr öffentliches Repository angeschaut und gesehen, dass Sie stateless4j und Envers verwenden …"* (I looked at your public repository and saw that you use stateless4j and Envers …)

---

## Sources
- DV Bern public repository `stip` (repo overview and `backend/pom.xml`): https://github.com/dvbern/stip
- DV Bern — kiBon Stip product page: https://www.dvbern.ch/de/insights/kibonstip
- Earlier, archived `stip-api` repository: https://github.com/dvbern/stip-api
- Quarkus releases / LTS policy: https://quarkus.io/releases
- Quarkus 3.40 LTS announcement: https://quarkus.io/blog/quarkus-3-40-released/
- Job ad: "Senior Backend Java Entwickler:in mit DevOps-Know-how", DV Bern AG (via SwissPeak Partners), provided by you
