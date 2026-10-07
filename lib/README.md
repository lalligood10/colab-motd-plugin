# lib/ — local Maven repository for SailPoint jars

`pom.xml` declares this directory as a file-based Maven repository. It holds
SailPoint jars that are not published to Maven Central.

## About the 9.0 jars

`lib/sailpoint/identityiq/9.0/identityiq-9.0.jar` and
`lib/sailpoint/openconnector/9.0/openconnector-9.0.jar` are **locally-transformed
stand-ins, not official SailPoint binaries**. Until SailPoint publishes official
IdentityIQ 9.0 jars, they are the 8.1 jars with the `javax.*` → `jakarta.*`
package rename applied to bytecode and resources by the
[Eclipse Transformer](https://github.com/eclipse/transformer) — the same tool
Apache Tomcat uses for Jakarta EE migration. The SailPoint API itself is
unchanged.

SHA-256 of the generated jars:

| Jar | SHA-256 |
|---|---|
| `identityiq-9.0.jar` | `72a3f6b80d31a6cd4fe0344bde94c08a194175d1687e4ee917a697d55858cfec` |
| `openconnector-9.0.jar` | `e9550b68ce0216069926a4851b63f866c8046b520541ac2c354586ce3c24bac6` |

## How they were produced

Tool: `org.eclipse.transformer:org.eclipse.transformer.cli:1.0.0`, main class
`org.eclipse.transformer.cli.JakartaTransformerCLI` (fetched with its
dependencies via `mvn dependency:copy-dependencies` into a scratch directory
outside the repo), default Jakarta rules, run on JDK 21:

```bash
# From the repository root, with the CLI and its deps on the classpath:
java -cp "<scratch>/deps/*" org.eclipse.transformer.cli.JakartaTransformerCLI \
     lib/sailpoint/identityiq/8.1/identityiq-8.1.jar /tmp/identityiq-9.0.jar -o \
     -ts lib/transformer/identityiq-selection.properties
java -cp "<scratch>/deps/*" org.eclipse.transformer.cli.JakartaTransformerCLI \
     lib/sailpoint/openconnector/8.1/openconnector-8.1.jar /tmp/openconnector-9.0.jar -o
```

The `-ts` selection file
[`transformer/identityiq-selection.properties`](transformer/identityiq-selection.properties)
selects everything (`*=`) except `sailpoint/tools/xml/XMLClasses.MF` — that file
is a plain class list, not a JAR manifest, and the transformer fails trying to
parse it as one. The `!` goes in the value (`XMLClasses.MF=!`); a leading `!`
key would be read as a .properties comment. Unselected entries are copied into
the output unchanged.

## Swapping in official jars

When SailPoint ships official IdentityIQ 9.0 jars, replace
`lib/sailpoint/<artifact>/9.0/<artifact>-9.0.jar` with the official jar (same
coordinates, `sailpoint:identityiq:9.0` / `sailpoint:openconnector:9.0`). No
pom or metadata changes are needed — Maven will resolve the new file on the
next build.
