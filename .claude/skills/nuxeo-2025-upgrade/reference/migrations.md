# Nuxeo 2023 → 2025 migrations reference

Exhaustive, concrete before→after changes. Apply only the ones the project actually triggers
(grep first). "Verify" notes say how to confirm a class/artifact location against the real 2025
artifacts when in doubt (decompile a cached jar with `javap`, or grep the Nuxeo 2025 source).

---

## Versions & build

### Parent vs platform version (critical)
`nuxeo-parent` releases as a **2-part** version (e.g. `2025.16`). The platform artifacts
(`nuxeo-runtime`, `nuxeo-core-*`, …) release as a **3-part** version (e.g. `2025.16.13`). The
parent's own `<parent>` is `org.nuxeo:nuxeo-ecm:<3-part>` and the BOM versions every artifact via
`${nuxeo.platform.version}`. So:

```xml
<!-- root pom -->
<parent>
  <groupId>org.nuxeo</groupId>
  <artifactId>nuxeo-parent</artifactId>
  <version>2025.16</version>          <!-- was 2023.x : 2-part build parent -->
</parent>
...
<properties>
  <nuxeo.platform.version>2025.16.13</nuxeo.platform.version>      <!-- 3-part! not 2025.16 -->
  <nuxeo.distribution.version>2025.16.13</nuxeo.distribution.version>
  <nuxeo.target.version>2025.16.13</nuxeo.target.version>
  <maven.compiler.release>21</maven.compiler.release>             <!-- was 17 -->
</properties>
```
To find the 3-part version: read the cached `nuxeo-parent-<2-part>.pom`'s `<parent><version>`.
**Setting these to the 2-part `2025.16` makes every nuxeo artifact 404.**

### Java 21
`maven.compiler.release` 17 → 21 in the root pom and in **every module** that overrides it
(`<source>17</source><target>17</target>` or `<release>17</release>` inside `maven-compiler-plugin`).
Build with `JAVA_HOME=$(/usr/libexec/java_home -v21)`.

### Marketplace package (`package.xml`)
```xml
<platform>lts-2023.*</platform>   →   <platform>lts-2025.*</platform>
```

### CI (`.github/workflows/*.yml`, Jenkinsfile)
JDK `17` → `21`; Maven `3.8.x` → `3.9.9`.

### Studio dependency
Maven versions can't contain `/`. For a Studio branch (e.g. `feature/2025`), use the sanitized
GAV from Studio → Branch Management → "Maven GAV" (often `2025-SNAPSHOT`). If resolution 401s,
download the jar and `mvn install:install-file -DgroupId=nuxeo-studio -DartifactId=<Project>
-Dversion=<ver> -Dpackaging=jar -Dfile=<path>`, then build with `-nsu`.

### commons-io (silent runtime breaker)
Remove any explicit `commons-io` pin older than 2.17 — let the BOM provide it (2.21.0). An old
pin compiles fine but fails at **runtime** in Nuxeo's `SQLDirectory` with
`NoClassDefFoundError: org/apache/commons/io/input/UnsynchronizedBufferedReader`.

---

## Source code

### Jakarta EE 10 (`javax` → `jakarta`)
Every Jakarta-EE namespace moves. These are the ones seen in practice:
```
javax.inject.*      → jakarta.inject.*
javax.ws.rs.*       → jakarta.ws.rs.*        (JAX-RS / WebEngine)
javax.servlet.*     → jakarta.servlet.*
javax.mail.*        → jakarta.mail.*
```
Mechanical sweep (safe — these four are all Jakarta EE):
```bash
for f in $(grep -rlE "javax\.(inject|ws\.rs|servlet|mail)" --include=*.java .); do
  perl -0pi -e 's/javax\.(inject|ws\.rs|servlet|mail)/jakarta.$1/g' "$f"; done
```
Do **not** move JDK `javax.*` (e.g. `javax.naming`, `javax.crypto`, `javax.xml`, `javax.security`).
Then the servlet-api dependency:
```xml
<groupId>javax.servlet</groupId><artifactId>javax.servlet-api</artifactId><version>3.0.1</version>
  → <groupId>jakarta.servlet</groupId><artifactId>jakarta.servlet-api</artifactId>  <!-- 6.0.0 via BOM -->
```

### Joda-Time → java.time
Joda is no longer in the distribution. Typical usage is just the current year/date:
```java
import org.joda.time.DateTime;     →  import java.time.Year;  (or java.time.LocalDate)
new DateTime().getYear()           →  Year.now().getValue()
DateTime d = new DateTime();        →  LocalDate d = LocalDate.now();   // d.getYear() unchanged
```

### Lombok (breaks on JDK 21)
Old Lombok throws `NoSuchFieldError: ... JCImport ... qualid` during compile. If usage is light,
remove it. Common case `@SneakyThrows` on a method that can't declare checked exceptions
(e.g. `AbstractWork.work()`): wrap the body and rethrow unchecked.
```java
@SneakyThrows
public void work() { ...body that throws checked... }
  →
public void work() {
    try { ...body... }
    catch (NuxeoException e) { throw e; }
    catch (Exception e) { throw new NuxeoException(e); }
}
```
Remove `import lombok.*;`. (Alternative: bump Lombok ≥ 1.18.30, but removing is cleaner if usage is small.)

### jetbrains @NotNull
Usually transitive and gone in 2025. Remove `import org.jetbrains.annotations.NotNull;` and the
`@NotNull` usages (they're hints only — behavior-neutral).

### Elasticsearch core removed → nuxeo-core-search
In poms (compile **and** `test-jar`):
```xml
<groupId>org.nuxeo.elasticsearch</groupId><artifactId>nuxeo-elasticsearch-core</artifactId>
  → <groupId>org.nuxeo.ecm.core</groupId><artifactId>nuxeo-core-search</artifactId>
```

**Deployment — search packages (IF on OpenSearch).** The 2025 search backend is modular: the
distribution no longer bundles a search client by default. **If** your deployment uses **OpenSearch**
(the typical case), you must explicitly deploy the OpenSearch client packages — otherwise search and
audit silently come up unconfigured. Check the current Nuxeo docs for the exact set, but at minimum:
- **`nuxeo-search-client-opensearch1`** — the repository search backend
- **`nuxeo-audit-opensearch1`** — the audit backend
Add them to the marketplace package `<require>`s / the distribution's installed packages (not a Maven
compile dependency). (If you run a different search engine — e.g. MongoDB Atlas Search — deploy the
corresponding client instead.)

### WebEngine FormData / getForm() removed
`WebContext.getForm()` and `org.nuxeo.ecm.webengine.forms.FormData` are gone. A `@POST` handler
that read form values now receives a `MultivaluedMap`, and overrides return `Template`:
```java
import org.nuxeo.ecm.webengine.forms.FormData;   // remove
import jakarta.ws.rs.core.MultivaluedMap;         // add
import org.nuxeo.ecm.webengine.model.Template;    // add

public Object validateTrialForm() {                       // old
    FormData f = getContext().getForm();
    String x = f.getString("RequestId");
}
  →
public Template validateTrialForm(MultivaluedMap<String,String> parameters) {   // new
    String x = parameters.getFirst("RequestId");
}
```
Check the parent class signatures (e.g. `UserInvitationObject`): several methods changed return
type `Object` → `Template`. For request-scoped reads outside a form handler, use
`getContext().getRequest().getParameter(...)` (jakarta `HttpServletRequest`).

### TaskObject package move (routing REST)
```
org.nuxeo.ecm.restapi.server.jaxrs.routing.TaskObject
  → org.nuxeo.ecm.restapi.server.routing.TaskObject     (drops ".jaxrs")
```

### nuxeo-platform-comment-core merged
`nuxeo-platform-comment-core` no longer exists; its classes merged into `nuxeo-platform-comment`
(which has both jar and test-jar in the BOM). Remove `-core` deps; if `nuxeo-platform-comment` is
already a dependency, the merged classes come with it.

### commons-lang → commons-lang3
```
import org.apache.commons.lang.StringUtils;  →  import org.apache.commons.lang3.StringUtils;
<groupId>commons-lang</groupId><artifactId>commons-lang</artifactId>
  → <groupId>org.apache.commons</groupId><artifactId>commons-lang3</artifactId>
```

### PDFBox 2 → 3
```java
import org.apache.pdfbox.pdfparser.PDFParser;   →  import org.apache.pdfbox.Loader;
PDDocument.load(inputStream)                    →  Loader.loadPDF(file)   // try-with-resources
```
Add `org.apache.pdfbox:pdfbox-io` if needed.

### commons-logging (optional)
`commons-logging` is still managed in the 2025 BOM, so migration is **not required**. Only migrate
to log4j2 if you hit a runtime classpath issue:
```java
org.apache.commons.logging.Log / LogFactory.getLog(X.class)
  →  org.apache.logging.log4j.Logger / LogManager.getLogger(X.class)
```

### Other removed APIs (grep; replace per Nuxeo upgrade notes)
`CoreSession#close()`, `Event#isLocal/isPublic`, `DownloadService#downloadBlobStatus`,
`AsyncBlob`, `FileManager#createDocumentFromBlob` → `createOrUpdateDocument(FileImporterContext)`,
`Batch#addChunk`, `CommentManager#getComments(doc,doc)` → `getComments(doc)`, `LifeCycleTrashService`
→ `PropertyTrashService`, `WorkManager#find/listWork/listWorkIds`, `org.mockito.Matchers` →
`org.mockito.ArgumentMatchers`. (Verify each against the 2025 jar before assuming.)

---

## Tests

### Search test feature
```
org.nuxeo.elasticsearch.test.RepositoryLightElasticSearchFeature
  → org.nuxeo.ecm.core.test.CoreSearchFeature        (in nuxeo-core-test)
```

### TransientStoreFeature package move
```
org.nuxeo.transientstore.test.TransientStoreFeature  →  org.nuxeo.transientstore.TransientStoreFeature
```

### Mockito
```
org.mockito.Matchers.anyX  →  org.mockito.ArgumentMatchers.anyX
```

### Servlet 6 HttpSession — removed methods
A mock implementing `jakarta.servlet.http.HttpSession` must drop the methods removed in Servlet 6
(they had `@Override` and no longer compile): `getSessionContext()` (+ `HttpSessionContext` import),
`getValue`, `getValueNames`, `putValue`, `removeValue`.

### CollectionFeature / SQLDirectoryFeature dependencies
Some test features that were transitive in 2023 need explicit (BOM-managed) test-jar deps in 2025:
```xml
<dependency><groupId>org.nuxeo.ecm.platform</groupId>
  <artifactId>nuxeo-platform-collections-core</artifactId><scope>test</scope></dependency>
<dependency><groupId>org.nuxeo.ecm.platform</groupId>
  <artifactId>nuxeo-platform-collections-core</artifactId><type>test-jar</type><scope>test</scope></dependency>
```
`CoreSearchFeature` (was `RepositoryLightElasticSearchFeature`) lives in `nuxeo-core-test`.

### mock-javamail → SmtpMailServerFeature
`org.jvnet.mock-javamail` is gone in 2025 (and its transitive `javax.mail:mail:1.4` is **banned**
by the enforcer). Remove the dependency and migrate the tests:
- `@Features(... org.nuxeo.mail.SmtpMailServerFeature.class)`; inject
  `SmtpMailServerFeature.MailsResult mailsResult;`
- `Mailbox.clearAll()` → `mailsResult.clearMails()`.
- If the code under test sends raw SMTP from `mail.transport.*` properties, point the port at the
  in-memory server and disable TLS/auth (set **all** props the sender reads — a missing one NPEs in
  `Properties.put`):
  ```java
  Framework.getProperties().setProperty("mail.transport.host", "localhost");
  Framework.getProperties().setProperty("mail.transport.port",
          Framework.getProperty("nuxeo.test.mail.smtp.port"));   // dumbster's random port
  Framework.getProperties().setProperty("mail.transport.auth", "false");
  Framework.getProperties().setProperty("mail.transport.usetls", "false");
  Framework.getProperties().setProperty("mail.transport.ssl.protocol", "TLSv1.2"); // keep if sender reads it
  ```
- Read results via `mailsResult.getMails()` / `getMailsByRecipient(addr)` / `getMailsBySubject(s)`.
  `MailMessage` has `getSubject/getContent/getSenders/getRecipients`.
- **Multi-recipient gotcha**: for one email sent to several recipients, the server returns the
  `To:` header as a single concatenated string, so `getMailsByRecipient` exact-matches only the
  first. Assert with `String.join(",", mail.getRecipients()).contains(addr)` instead.

### DublinCore contributor overwrite (test data)
If a test sets `dc:contributors` and then asserts on it, the DublinCore listener overwrites it with
the saving principal on save. Preserve the explicit value:
```java
doc.putContextData("disableDublinCoreListener", true);
doc.setPropertyValue("dc:contributors", contributors);
session.saveDocument(doc);
```

### Bulk scroller `elastic` in tests
Production bulk actions often use `defaultScroller="elastic"` (Elasticsearch). In tests running on
`CoreSearchFeature` (no ES) the `elastic` scroller doesn't exist → `IllegalArgumentException:
Unknown Document Scroller ... scroller=elastic`. **Do not change the production contributions.**
Instead deploy a test-only alias that maps `elastic` to the repository scroll, and `@Deploy` it in
the affected tests:
```xml
<!-- src/test/resources/test-elastic-scroll-contrib.xml -->
<component name="<your.pkg>.test.scroll.elastic-alias">
  <require>org.nuxeo.ecm.core.scroll.contrib.default</require>
  <extension target="org.nuxeo.ecm.core.scroll.service" point="scroll">
    <scroll type="document" name="elastic" class="org.nuxeo.ecm.core.scroll.RepositoryScroll"/>
  </extension>
</component>
```
```java
@Deploy("<your-test-bundle>:src/test/resources/test-elastic-scroll-contrib.xml")
```

### Quarantining unmigratable tests
For large tests on removed infra (e.g. the Jersey 1.x REST client: `com.sun.jersey.*`,
`org.nuxeo.ecm.restapi.test.BaseTest`, `CloseableClientResponse`, `org.nuxeo.jaxrs.test` — all
replaced by `RestServerFeature` + the new HTTP test client), it's acceptable to exclude them from
test compilation and track them, rather than block the upgrade:
```xml
<plugin><artifactId>maven-compiler-plugin</artifactId>
  <configuration>
    <!-- TODO(2025-upgrade): migrate to RestServerFeature + new HTTP test client -->
    <testExcludes><testExclude>**/SomeLegacyRestClientTest.java</testExclude></testExcludes>
  </configuration>
</plugin>
```

### SQL directories / vocabularies in tests (sharp edge)
Test directory **templates** and datasources moved into `*-tests` jars:
- `template-directory` + the test datasource → `nuxeo-platform-directory-sql` test-jar, deployed by
  `org.nuxeo.ecm.directory.sql.SQLDirectoryFeature` (add it to `@Features` and the test-jar dep).
- The `vocabulary` schema → `org.nuxeo.ecm.directory.types.contrib` (deploy it).
Studio-vocabulary test directories (e.g. a feature-flag vocab) can still be stubborn to register;
if one won't come up, surface it to the user as a known sharp edge rather than spending the whole
budget on it.
