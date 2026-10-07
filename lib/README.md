# lib/ – local Maven repository for SailPoint jars

`pom.xml` declares this directory as a file-based Maven repository (`<repository><id>lib</id>`).
It holds SailPoint jars that are not published to Maven Central:

| Artifact | Version | Used for |
|---|---|---|
| `sailpoint:identityiq` | `8.1` | Original IdentityIQ 8.1 jar (`javax.*` namespace). Kept as the transformer input; no longer referenced by `pom.xml`. |
| `sailpoint:identityiq` | `9.0` | **Jakarta stand-in** generated from `8.1` with the Eclipse Transformer (see below). Compile dependency. |
| `sailpoint:openconnector` | `8.1` | Original openconnector 8.1 jar. Kept as the transformer input; no longer referenced. |
| `sailpoint:openconnector` | `9.0` | Stand-in generated from `8.1` with the Eclipse Transformer. Test dependency. |

## Why the 9.0 jars exist

IdentityIQ 9.0 runs on Tomcat 10 / Jakarta EE 10, so the plugin now compiles against
`jakarta.servlet`, `jakarta.validation` and `jakarta.ws.rs`. The vendored 8.1 jars still
reference `javax.*` (e.g. `sailpoint.rest.BaseResource`, the superclass of the plugin's
`BasePluginResource`, exposes `javax.servlet.http.HttpServletRequest` and
`javax.ws.rs.core.UriInfo`), so with only the Jakarta APIs on the classpath the plugin and its
tests fail with missing `javax.*` classes.

The `9.0` jars are **not** official SailPoint 9.0 binaries. They are the 8.1 jars with the
`javax.*` → `jakarta.*` package rename applied to bytecode and resources by the
[Eclipse Transformer](https://github.com/eclipse/transformer) — the same tool Apache Tomcat
uses (`tomcat-jakartaee-migration`) to migrate EE 8 applications to Jakarta EE 9+. The
SailPoint API itself is unchanged. If you have access to the real IdentityIQ 9.0
`identityiq.jar`, install it in place of the stand-in (same coordinates) and the build will
use it as-is.

## How they were produced

Tool: `org.eclipse.transformer:org.eclipse.transformer.cli:1.0.0` (distribution jar from
Maven Central, SHA-1 verified), default Jakarta rules, run on JDK 21.

```bash
# 1. Get the CLI
curl -O https://repo1.maven.org/maven2/org/eclipse/transformer/org.eclipse.transformer.cli/1.0.0/org.eclipse.transformer.cli-1.0.0-distribution.jar
unzip org.eclipse.transformer.cli-1.0.0-distribution.jar -d transformer
CLI=transformer/org.eclipse.transformer.cli-1.0.0.jar

# 2. Transform (from the repository root)
java -jar $CLI lib/sailpoint/identityiq/8.1/identityiq-8.1.jar /tmp/identityiq-9.0.jar -o \
     -ts lib/transformer/identityiq-selection.properties
java -jar $CLI lib/sailpoint/openconnector/8.1/openconnector-8.1.jar /tmp/openconnector-9.0.jar -o

# 3. Install into this repository (creates the 9.0 pom + updates maven-metadata-local.xml)
for a in identityiq openconnector; do
  mvn org.apache.maven.plugins:maven-install-plugin:3.1.2:install-file \
      -Dfile=/tmp/$a-9.0.jar -DgroupId=sailpoint -DartifactId=$a -Dversion=9.0 \
      -Dpackaging=jar -DlocalRepositoryPath=lib
done
```

[`transformer/identityiq-selection.properties`](transformer/identityiq-selection.properties)
excludes one entry, `sailpoint/tools/xml/XMLClasses.MF`: despite the `.MF` extension it is a
plain list of class names (no `javax` references), and the transformer fails trying to parse
it as a JAR manifest. Excluded entries are copied through byte-for-byte.

## Results

| Jar | Entries | Changed by transformer | Failed | Return code |
|---|---|---|---|---|
| identityiq-8.1 → 9.0 | 4704 | 597 (content) | 0 | 0 (success) |
| openconnector-8.1 → 9.0 | 66 | 0 (it has no `javax` EE references) | 0 | 0 (success) |

`openconnector-9.0.jar` is functionally identical to 8.1; it is re-versioned only so that
both SailPoint artifacts move to `9.0` together.

Checks on `identityiq-9.0.jar`:

- Same entry list as `identityiq-8.1.jar`.
- Files containing Jakarta-EE-family `javax/...` strings (`servlet`, `ws/rs`, `faces`,
  `validation`, `el`, `inject`, `mail`, ...): 576 in 8.1 → 9 in 9.0.
- The 9 remaining matches are **orphaned constant-pool UTF-8 entries** (old generic
  signatures that the transformer replaced with new `jakarta` entries but did not remove).
  `javap -v` shows no attribute or constant referencing them, so they have no effect on
  linking or reflection.
- Java SE `javax.*` packages (`javax.naming`, `javax.sql`, `javax.xml`, `javax.crypto`, ...)
  are intentionally left unchanged, as in the default Jakarta rules.

| Jar | SHA-256 |
|---|---|
| identityiq-8.1.jar | `aa3e04278901d634dc33f18f52bd362b0645b3eb46d5e65dd2f3827049c378c8` |
| identityiq-9.0.jar | `de86c16960233ed1f4176ff28ff977aa13f173fedc1959e944a350d3930aae6f` |
| openconnector-8.1.jar | `6aaee503be8dc3d27c181cd013c5158b536d452c295a352292fc2d520f598ba7` |
| openconnector-9.0.jar | `e9550b68ce0216069926a4851b63f866c8046b520541ac2c354586ce3c24bac6` |

The transformer output is not byte-reproducible (zip entry timestamps), so re-running it
gives a different checksum with the same contents.
