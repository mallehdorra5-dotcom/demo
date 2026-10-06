# Security Testing Report

## 1. Introduction

This report presents the security testing performed on the Spring Boot application.

Three security testing methods were implemented:

- SAST using Semgrep
- SCA using OWASP Dependency-Check
- DAST using OWASP ZAP

The three security tests were integrated into GitHub Actions in order to automate the security analysis when changes are pushed to the `main` branch.

---

## 2. Security Testing Approach

The security analysis was divided into three levels.

### SAST

SAST (Static Application Security Testing) analyzes the source code and configuration files without executing the application.

**Tool:** Semgrep

### SCA

SCA (Software Composition Analysis) analyzes the third-party libraries and dependencies used by the application.

**Tool:** OWASP Dependency-Check

### DAST

DAST (Dynamic Application Security Testing) analyzes the application while it is running.

**Tool:** OWASP ZAP

The tests are executed automatically using GitHub Actions.

---

# 3. SAST – Static Application Security Testing

## 3.1 Objective

The objective of the SAST test is to identify potential security issues in the source code and configuration files without executing the application.

## 3.2 Tool Used

Semgrep was used to perform the static security analysis.

The scan was integrated into the GitHub Actions workflow:

`.github/workflows/sast.yml`

## 3.3 Test Execution

The Semgrep scan is automatically executed when a change is pushed to the `main` branch.

The scan uses the Semgrep automatic rules:

```text
semgrep scan --config auto /src
```
## 3.4 Results

The Semgrep scan completed successfully.

The scan analyzed 12 targets using 203 security rules.

The following results were obtained:

- Total findings: 7
- Blocking findings: 7

All identified findings are related to the use of mutable GitHub Actions references in the CI/CD workflow files.

## 3.5 Findings

Semgrep detected the following mutable GitHub Actions references:

| File | Reference |
|---|---|
| `.github/workflows/dast.yaml` | `actions/checkout@v4` |
| `.github/workflows/dast.yaml` | `actions/setup-java@v4` |
| `.github/workflows/dependency-scan.yml` | `actions/checkout@v4` |
| `.github/workflows/dependency-scan.yml` | `actions/setup-java@v4` |
| `.github/workflows/dependency-scan.yml` | `dependency-check/Dependency-Check_Action@main` |
| `.github/workflows/dependency-scan.yml` | `actions/upload-artifact@v4` |
| `.github/workflows/sast.yml` | `actions/checkout@v4` |

These findings concern the use of mutable tags or branch references in GitHub Actions.

According to Semgrep, these references can potentially be changed by the action owner and may introduce a software supply-chain risk.

## 3.6 Recommendation

The recommended solution is to pin GitHub Actions to a specific commit SHA instead of using mutable tags or branches.

For example:

```yaml
uses: actions/checkout@<commit-sha>
```

The same approach should be applied to the other GitHub Actions used in the workflows.

---

# 4. SCA – Software Composition Analysis

## 4.1 Objective

The objective of the SCA test is to identify known security vulnerabilities in the third-party dependencies used by the Spring Boot application.

## 4.2 Tool Used

OWASP Dependency-Check was used to perform the dependency security analysis.

The scan was integrated into:

`.github/workflows/dependency-scan.yml`

## 4.3 Test Execution

The project is first built using Maven.

Dependency-Check is then executed to analyze the project dependencies.

The generated reports are uploaded to GitHub Actions as an artifact named:

```text
dependency-check-report
```

## 4.4 Results

The Dependency-Check scan was completed successfully.

The report was generated using Dependency-Check version 13.0.0.

The analysis scanned 35 dependencies, including 28 unique dependencies.

The scan identified:

- 1 vulnerable dependency
- 11 vulnerabilities
- 0 suppressed vulnerabilities

The highest severity identified was CRITICAL.

## 4.5 Findings

Dependency-Check identified one vulnerable dependency:

| Dependency | Version | Vulnerabilities | Highest Severity |
|---|---|---:|---|
| `tomcat-embed-core` | `11.0.24` | 11 | CRITICAL |

The vulnerable dependency is part of Apache Tomcat.

The highest severity identified was CRITICAL, with a CVSS score of 9.1 for CVE-2026-68525.

Other vulnerabilities identified include:

- CVE-2026-65183 – HIGH – CVSS 8.1
- CVE-2026-66422 – HIGH – CVSS 8.1
- CVE-2026-68569 – HIGH – CVSS 8.1
- CVE-2026-65927 – HIGH – CVSS 7.5
- CVE-2026-68763 – HIGH – CVSS 7.5
- CVE-2026-73180 – MEDIUM – CVSS 6.8
- CVE-2026-66299 – MEDIUM – CVSS 5.3

The Dependency-Check report recommends upgrading Apache Tomcat to a fixed version.

For the 11.0.x branch, the recommended fixed version is 11.0.25.

## 4.6 Recommendation

The main recommendation is to update the vulnerable Apache Tomcat dependency from version `11.0.24` to a fixed version such as `11.0.25`.

The project dependencies should also be reviewed regularly to detect newly published vulnerabilities.

Dependency-Check should remain integrated into GitHub Actions so that dependency security can be checked automatically after changes to the project.
# 5. DAST – Dynamic Application Security Testing

## 5.1 Objective

The objective of the DAST test is to analyze the Spring Boot application while it is running and identify potential security issues from the application's external behavior.

## 5.2 Tool Used

OWASP ZAP was used to perform the dynamic security analysis.

The scan was integrated into:

`.github/workflows/dast.yaml`

## 5.3 Target

The Spring Boot application was started on:

```text
http://localhost:8080
```

The main endpoint tested by OWASP ZAP was:

```text
http://localhost:8080/hello
```

## 5.4 Test Execution

The application is started automatically by GitHub Actions using Maven:

```bash
./mvnw spring-boot:run
```

The workflow then verifies that the application is available:

```bash
curl -f http://localhost:8080/hello
```

OWASP ZAP is then executed using its Docker image:

```bash
docker run --rm \
  --network host \
  ghcr.io/zaproxy/zaproxy:stable \
  zap-baseline.py \
  -t http://localhost:8080/hello \
  -I
```

## 5.5 Results

The OWASP ZAP scan completed successfully.

The results were:

| Result | Number |
|---|---:|
| PASS | 64 |
| WARN | 3 |
| FAIL | 0 |

No `FAIL-NEW` findings were detected.

Three warnings were reported.

## 5.6 Findings

### Finding 1 – X-Content-Type-Options Header Missing

**ZAP Rule:** `10021`

The application does not return the recommended `X-Content-Type-Options` HTTP security header.

**Recommendation:**

Add the following HTTP response header:

```text
X-Content-Type-Options: nosniff
```

### Finding 2 – Storable and Cacheable Content

**ZAP Rule:** `10049`

ZAP reported cacheability-related warnings for the following URLs:

```text
/
/hello
/robots.txt
/sitemap.xml
```

This warning indicates that the responses can be stored or cached.

**Recommendation:**

Review the cache-control policy of the application and ensure that sensitive responses are not cached by browsers or intermediary systems.

### Finding 3 – Cross-Origin-Resource-Policy Header Missing or Invalid

**ZAP Rule:** `90004`

The application does not provide a valid `Cross-Origin-Resource-Policy` security header.

**Recommendation:**

Review the cross-origin policy of the application and configure an appropriate `Cross-Origin-Resource-Policy` header.

---

# 6. Security Testing Summary

The security testing of the application was performed using three different approaches.

| Test | Tool | Result |
|---|---|---|
| SAST | Semgrep | 7 findings |
| SCA | OWASP Dependency-Check | 1 vulnerable dependency, 11 vulnerabilities |
| DAST | OWASP ZAP | 64 PASS, 3 WARN, 0 FAIL |

The SAST analysis identified 7 findings related to mutable GitHub Actions references.

The SCA analysis identified one vulnerable Apache Tomcat dependency with 11 associated vulnerabilities. The highest severity was CRITICAL.

The DAST analysis identified three warnings related to HTTP security headers and cacheability. No `FAIL-NEW` findings were reported.

---

# 7. Conclusion

The Spring Boot application was tested using SAST, SCA and DAST techniques.

The three security tests were integrated into GitHub Actions, allowing security checks to be performed automatically when changes are pushed to the `main` branch.

The main issues identified during the tests are:

- Mutable GitHub Actions references detected by Semgrep.
- A vulnerable Apache Tomcat dependency detected by Dependency-Check.
- 11 vulnerabilities associated with the vulnerable dependency.
- Missing X-Content-Type-Options security header.
- Missing or invalid Cross-Origin-Resource-Policy header.
- Cacheability warnings reported by OWASP ZAP.

The main remediation actions are to update the vulnerable Tomcat dependency, review the identified GitHub Actions references, and improve the HTTP security headers.

The security testing process provides an automated first level of security verification for the Spring Boot application.