Yes. From a GitHub/PR-review perspective, this CLI is reasonably structured, but I’d flag several things before merging—especially around **configuration precedence, validation, security-sensitive credentials, proxy parsing, exit codes, and testability**.

## Overall review

**Verdict: Request changes / medium priority**

The class is readable and the responsibilities are reasonably clear for a CLI entry point. The comments around configuration precedence are particularly helpful.

The biggest issue is that `ASTFCli` is becoming a **god-ish orchestration class**: Picocli parsing, config merging, authentication setup, endpoint loading, proxy parsing, validation, console rendering, and report generation all live here.

I would aim for:

```text
ASTFCli
 ├── CliOptions
 ├── ScanConfigMerger
 ├── ConfigValidator
 ├── ProxyParser
 ├── ConsoleReporter
 └── Scanner / ReportGenerator
```

That would make the CLI considerably easier to test and evolve.

---

# Review comments I'd leave on the PR

### 1. `--api-key` can result in duplicate headers

This section is potentially problematic:

```java
if (headers != null) {
    for (String header : headers) {
        ...
        config.addHeader(...)
    }
}

...

if (config.getApiKey() != null && !config.getApiKey().isEmpty()) {
    String header = config.getApiKeyHeader() != null ? config.getApiKeyHeader() : "X-API-Key";
    config.addHeader(header, config.getApiKey());
}
```

If the user does:

```bash
--api-key abc123 --header X-API-Key:def456
```

you may end up with two `X-API-Key` headers.

Depending on the HTTP client, this could mean:

* first value vins
* last value wins
* both values are sent
* behavior varies between servers

I'd define explicit precedence.

For example:

> Explicit `--header` overrides automatically generated authentication headers.

Or reject conflicting configuration.

A cleaner implementation would use a map-like header structure rather than a list if duplicate header names aren't intended.

---

### 2. Authentication values should not be exposed accidentally

These options contain secrets:

```java
--token
--secondary-token
--api-key
--password
--client-cert-password
```

The CLI itself doesn't print them, which is good, but I'd review the entire downstream path for logging.

In particular, make sure none of these can reach:

```java
logger.info(...)
logger.debug(...)
logger.error(...)
```

through `ScanConfig.toString()`, HTTP request logging, exception messages, or report generation.

I'd strongly recommend implementing a redacted `toString()` for `ScanConfig`, or avoiding logging the config entirely.

For example:

```java
@Override
public String toString() {
    return "ScanConfig{" +
            "targetUrl='" + targetUrl + '\'' +
            ", bearerToken='[REDACTED]'" +
            ", apiKey='[REDACTED]'" +
            ", basicAuthPassword='[REDACTED]'" +
            '}';
}
```

And ideally use environment variables/configuration files for secrets rather than putting them directly on the command line, because command-line arguments can be visible to other processes depending on the OS.

---

### 3. `parseProxyUrl()` is too permissive and fragile

This is probably the method I'd refactor first.

```java
String[] hostPort = url.split(":", 2);
```

It doesn't properly handle several valid/interesting cases:

```text
http://proxy.example.com:8080
http://user:password@proxy.example.com:8080
http://[::1]:8080
https://proxy.example.com
```

It also accepts things you probably don't want.

More importantly:

```java
catch (Exception e) {
    System.err.println("Warning: could not parse proxy URL: " + proxyUrl);
}
```

means an invalid proxy configuration doesn't fail the scan.

That can produce a particularly confusing result:

> User thinks the scan is going through a proxy, but it isn't.

I'd make invalid proxy configuration a **configuration error**, not a warning.

Use `java.net.URI` rather than manually parsing the URL.

Something along these lines:

```java
private void parseProxyUrl(String value, ScanConfig config) {
    try {
        URI uri = URI.create(value);

        if (uri.getHost() == null) {
            throw new IllegalArgumentException("Proxy host is missing");
        }

        config.setProxyHost(uri.getHost());

        if (uri.getPort() != -1) {
            config.setProxyPort(uri.getPort());
        }

        if (uri.getUserInfo() != null) {
            String[] credentials = uri.getUserInfo().split(":", 2);
            config.setProxyUsername(credentials[0]);

            if (credentials.length == 2) {
                config.setProxyPassword(credentials[1]);
            }
        }
    } catch (IllegalArgumentException e) {
        throw new IllegalArgumentException(
                "Invalid proxy URL: " + value, e);
    }
}
```

I'd additionally validate allowed schemes and port ranges.

---

### 4. `validateConfig()` needs more validation

Currently:

```java
if (config.getTargetUrl() == null || config.getTargetUrl().isBlank()) {
    ...
}

if (!config.getTargetUrl().startsWith("http://")
        && !config.getTargetUrl().startsWith("https://")) {
    ...
}
```

This accepts things like:

```text
https://
https://?
https://#
https://:1234
```

I'd parse the URL with `URI` and validate:

* scheme
* host
* port
* possibly userinfo
* malformed URI syntax

Also validate numerical configuration:

```text
threads > 0
timeoutMinutes > 0
proxyPort between 1 and 65535
```

Otherwise something like:

```bash
--threads 0
```

gets through the CLI and may fail much later inside the scanner.

The principle I'd use is:

> Fail as early as possible at the CLI boundary.

---

### 5. `--threads` and `--timeout` should probably have Picocli validation

Instead of waiting for:

```java
validateConfig(config);
```

you can let Picocli reject obviously invalid values.

For example:

```java
@Option(
    names = {"-t", "--threads"},
    description = "Number of concurrent threads",
    defaultValue = "10"
)
@CommandLine.Range(min = "1")
private Integer threads;
```

And:

```java
@Option(
    names = {"--timeout"},
    description = "Scan timeout in minutes",
    defaultValue = "30"
)
@CommandLine.Range(min = "1")
private Integer timeoutMinutes;
```

This gives users a much better CLI experience.

---

### 6. The config precedence logic should be centralized

This method is doing a lot:

```java
private ScanConfig buildConfig()
```

The precedence rules are important enough that I'd make them explicit in one dedicated component.

Right now there are several independent precedence rules:

```text
CLI
 ↓
config file
 ↓
defaults
```

but endpoint handling has a separate hierarchy:

```text
--endpoints-file
inline endpoints
endpointsFile
automatic discovery
hardcoded fallback
```

This is business logic, not really CLI logic.

I'd extract something like:

```java
ScanConfig config = configMerger.merge(
    fileConfig,
    cliOptions
);
```

Then you can unit test:

```text
CLI URL overrides YAML URL
CLI threads override YAML threads
CLI format overrides YAML format
CLI endpoints override YAML endpoints
YAML format survives when CLI format isn't supplied
defaults apply when neither exists
```

That would be a very valuable test suite.

---

### 7. `format` should ideally be an enum

Currently:

```java
private String format;
```

followed by:

```java
ScanConfig.OutputFormat.valueOf(format.toUpperCase())
```

I'd consider:

```java
private ScanConfig.OutputFormat format;
```

with Picocli conversion.

That removes manual parsing and gives you stronger typing.

You can also provide a case-insensitive converter if you want:

```text
json
JSON
Json
```

all to work.

---

### 8. Test-case parsing needs trimming and validation

This:

```java
List.of(enabledTestCases.split(",\\s*"))
```

handles:

```text
API1, API2, API3
```

but doesn't handle whitespace before the first item particularly well:

```text
" API1, API2"
```

You also potentially accept empty values.

I'd extract:

```java
private List<String> parseIds(String value) {
    return Arrays.stream(value.split(","))
            .map(String::trim)
            .filter(s -> !s.isEmpty())
            .toList();
}
```

Even better, validate IDs against the registered test cases and fail with:

```text
Unknown test case: API99
Available test cases: ...
```

rather than silently doing nothing.

---

### 9. Header parsing should reject malformed input instead of continuing

Current behavior:

```java
System.err.println(
    "Warning: skipping invalid header ..."
);
```

I'd question whether this should be fatal.

For a security scanner, silently ignoring a supplied security-relevant option can produce misleading results.

For example:

```bash
--header Authorization Bearer...
```

instead of:

```bash
--header "Authorization: Bearer..."
```

could cause the scanner to run unauthenticated.

I'd prefer:

```text
Configuration error: invalid header 'Authorization Bearer...'.
Expected format: Key:Value
```

and exit `2`.

At minimum, consider validating header names as well.

---

### 10. The CLI should probably use Picocli's `@Spec` / command execution context for errors

The current pattern:

```java
catch (IllegalArgumentException e) {
    System.err.println("Configuration error: " + e.getMessage());
    return 2;
}
```

works, but you're mixing application execution and CLI presentation.

Picocli can provide cleaner error handling and usage output.

For example, configuration errors could result in:

```text
Configuration error: Target URL is required.

Usage: astf [OPTIONS]
...
```

That is much friendlier than only printing the exception.

---

# Important security concern: URL handling

Because this is an **API security testing framework**, I'd specifically review SSRF implications.

You currently accept:

```java
--url <target>
```

and potentially discover endpoints from the target.

If this tool is ever exposed as a service or integrated into an automated platform, an attacker could potentially turn it into a scanner against:

```text
http://localhost
http://127.0.0.1
http://169.254.169.254
http://10.x.x.x
http://192.168.x.x
```

That's not necessarily a vulnerability in a local CLI—scanning arbitrary URLs is arguably the purpose of the tool—but it becomes important if the CLI is wrapped by another service.

I'd document the trust model clearly and, if there's ever a server/API wrapper, implement target-network restrictions there.

---

# Exit codes could be more expressive

You have:

```java
return result.getTotalFindingsCount() > 0 ? 1 : 0;
```

with:

```text
0 = no findings
1 = findings
2 = error
```

That's perfectly reasonable, but I'd document this as part of the CLI contract.

More importantly, consider whether **INFO findings** should cause exit code `1`.

Right now:

```java
INFO > 0 → exit 1
```

Most CI/CD security tools generally care about findings at or above a configurable severity.

I'd consider:

```bash
--fail-on CRITICAL
--fail-on HIGH
--fail-on MEDIUM
--fail-on LOW
--fail-on INFO
```

Then:

```text
astf ... --fail-on HIGH
```

would return `1` only if HIGH/CRITICAL findings exist.

That would make this much more useful in GitHub Actions/GitLab/Jenkins pipelines.

---

# Reporting has a subtle issue

You do:

```java
String reportContent = generator.generate(result);

if (config.getOutputFile() != null && !config.getOutputFile().isBlank()) {
    generator.generateToFile(result, config.getOutputFile());
}
```

You're generating the report **twice** when an output file is specified.

Potentially:

```text
generate()
generateToFile()
```

could duplicate expensive serialization work.

If the generator API permits it, I'd do either:

```java
String reportContent = generator.generate(result);

if (...) {
    Files.writeString(..., reportContent);
} else {
    System.out.println(reportContent);
}
```

or:

```java
if (...) {
    generator.generateToFile(...);
} else {
    System.out.println(generator.generate(...));
}
```

The second is probably preferable if `generateToFile()` has format-specific behavior.

---

# `System.out` vs logging

You have both:

```java
System.out.println(...)
```

and:

```java
logger.info(...)
logger.error(...)
```

I'd define a deliberate boundary.

For example:

* **stdout** → user-facing scan results/report
* **stderr** → errors/warnings
* **logger** → diagnostic/internal logging

This becomes especially important if users do:

```bash
astf --url ... > report.json
```

At the moment, the banner and status messages could contaminate stdout if JSON output is intended to be machine-consumed.

This is probably one of the **highest-value improvements**.

For machine-readable formats, stdout should ideally contain **only the report**.

For example:

```bash
astf -u https://api.example.com -f json > report.json
```

should produce valid JSON.

You don't want:

```text
___ ...
OWASP API Security Testing Framework...

Target: ...
Format: JSON

{
   ...
}
```

in the redirected file.

I'd send progress information to `stderr`.

---

# `printBanner()` should probably be optional

This:

```java
private void printBanner()
```

is nice interactively, but problematic for automation.

Consider:

```bash
--quiet
```

or:

```bash
--no-banner
```

Even better, automatically suppress it when producing machine-readable stdout.

---

# The `VersionProvider` change is good

This part is a nice improvement:

```java
static class VersionProvider implements CommandLine.IVersionProvider {
    @Override
    public String[] getVersion() {
        return new String[] {
            "OWASP API Security Testing Framework v" + VersionInfo.getVersion()
        };
    }
}
```

I'd keep it.

The comment is also useful because it explains **why** this isn't simply a constant in the annotation.

---

# One architectural improvement I'd strongly recommend

I'd reduce `ASTFCli` to something closer to:

```java
@Command(...)
public class ASTFCli implements Callable<Integer> {

    private CliOptions options;

    private final ConfigLoader configLoader;
    private final ScanConfigMerger configMerger;
    private final ConfigValidator validator;
    private final ScanRunner scanRunner;
    private final ConsoleReporter consoleReporter;

    @Override
    public Integer call() {
        try {
            ScanConfig config = configMerger.merge(options);
            validator.validate(config);

            ScanResult result = scanRunner.scan(config);

            consoleReporter.printSummary(result, config);
            consoleReporter.writeReport(result, config);

            return determineExitCode(result, config);

        } catch (ConfigurationException e) {
            consoleReporter.error("Configuration error", e);
            return 2;
        } catch (Exception e) {
            logger.error("Scan failed", e);
            consoleReporter.error("Scan failed", e);
            return 2;
        }
    }
}
```

Then the CLI class becomes an actual **CLI adapter**, rather than containing most of the application's configuration logic.

---

# Tests I'd expect in the PR

At minimum, I'd add unit tests for:

### Configuration

```text
✓ CLI URL overrides config URL
✓ CLI format overrides config format
✓ config format used when CLI format absent
✓ JSON is default
✓ CLI threads override config threads
✓ CLI timeout overrides config timeout
✓ --no-discovery overrides config
```

### Authentication

```text
✓ bearer token applied
✓ secondary token applied
✓ API key applied
✓ custom API key header applied
✓ API key doesn't duplicate an explicitly supplied header
✓ credentials are not exposed in logs/toString
```

### Headers

```text
✓ Key:Value parsed
✓ whitespace trimmed
✓ malformed header rejected
✓ empty header rejected
✓ duplicate header behavior defined
```

### Proxy

```text
✓ host:port
✓ authenticated proxy
✓ missing host rejected
✓ invalid port rejected
✓ IPv6 proxy
✓ invalid URL rejected
```

### Endpoint precedence

```text
✓ --endpoints-file wins
✓ inline endpoints beat endpointsFile
✓ config endpointsFile works
✓ discovery used when no manual endpoints exist
```

### Exit behavior

```text
✓ zero findings → 0
✓ findings → 1
✓ configuration failure → 2
✓ scanner exception → 2
```

---

## My GitHub review summary

I'd probably leave the PR with something like:

> **Overall:** The CLI is readable and the configuration precedence is documented well. The dynamic `VersionProvider` is a good improvement.
>
> **Changes I'd request before merge:**
>
> 1. Make proxy parsing robust using `URI` and treat invalid proxy configuration as an error.
> 2. Validate URL, thread count, timeout, and other CLI values before starting the scan.
> 3. Define and enforce precedence for API-key-generated headers vs `--header`.
> 4. Ensure credentials are never logged or included in exception/report output.
> 5. Separate machine-readable report output from CLI status/banner output, particularly for JSON/SARIF.
> 6. Avoid generating reports twice when an output file is specified.
> 7. Extract configuration merging/parsing from `ASTFCli` so the precedence rules can be unit-tested independently.
> 8. Add tests covering CLI-over-config precedence, endpoint precedence, authentication, proxy parsing, and exit codes.

**Priority-wise:** I'd call #1–#5 **important**, #6–#8 **medium**, and the larger refactor a **follow-up if you want to keep this PR small**.

The biggest practical improvement for an OWASP scanner, though, would be making the CLI **CI-friendly**: clean stdout, deterministic exit codes, `--fail-on <severity>`, strong config validation, and zero chance of leaking credentials.
