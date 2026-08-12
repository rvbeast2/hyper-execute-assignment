# HyperExecute Assignment — Submission Notes

## Task 1: Fixed the broken YAML

I compared the broken YAML against LambdaTest's own working reference file in the sample repo (`yaml/win/v1/testng_hyperexecute_autosplit_sample.yaml`) to confirm each fix against a known-correct baseline rather than guessing.
And Created a yaml file which has the fixes (fixed.yaml)

**Bugs found:**

1. **`conCurrency: 1` → `concurrency: 1`** — YAML keys are case-sensitive. `conCurrency` isn't a recognized HyperExecute parameter, so it was silently ignored, meaning the intended concurrency setting never actually applied.

2. **`env: TOKEN: anvdegtod-asdaasda0asda-asda` — invalid YAML syntax.** A nested key can't be written on the same line as its parent. Fixed by properly nesting it:
```yaml
   env:
     TOKEN: anvdegtod-asdaasda0asda-asda
```

3. **`testDiscovery` children not indented.** `type`, `mode`, and `command` were at the same indentation level as `testDiscovery:` itself, so YAML parsed them as unrelated top-level keys instead of children of `testDiscovery`. Fixed by indenting them 2 spaces under `testDiscovery:`.

4. **`mode: dynamic` → `mode: remote`.** The working reference uses `remote`; `dynamic` isn't a valid mode value in HyperExecute's `testDiscovery` schema.

5. **Missing Maven flag in the `pre` step.** Broken: `mvn dependency:resolve`. Fixed: `mvn -Dmaven.repo.local=./.m2 dependency:resolve`. Without this flag, dependencies resolved to Maven's default cache location, but `testRunnerCommand` later expects them at `./.m2` specifically — a path mismatch that would cause the test run to fail finding dependencies.

6. **Missing `runtime` block.** The broken YAML had no `runtime` specification at all. I discovered this the hard way — my first "fixed" run failed on the cloud with a Java compile error (`invalid target release: 11`), and I reproduced the exact same error locally once I set up Maven, which confirmed the remote VM was defaulting to an incompatible JDK version. Adding:
```yaml
   runtime:
     language: java
     version: 11
```
   resolved it on both local and cloud runs.

**Deliberately left unchanged (verified, not a bug):** the `testRunnerCommand` line (`` mvn test `-Dplatname=win `-Dmaven.repo.local=./.m2 dependency:resolve `-DselectedTests=$test ``) looks unusual, but it's identical to LambdaTest's own working sample. I confirmed `-Dplatname=win` is a real, necessary property — when I ran a plain `mvn test` locally without it, I got `testng_${platname}.xml is not a valid file`, proving this flag is required, not a typo.

**Result:** job ran successfully, 100% pass across discovery/pre/test stages, 0 failed tasks.
Job link: `[paste your successful job link here]`

---

## Task 2: Environment Variables

Added a custom env var to the fixed YAML:
```yaml
env:
  TOKEN: anvdegtod-asdaasda0asda-asda
  ENVIRONMENT: staging
```

Printed it in the `pre` step:
```yaml
pre:
  - mvn -Dmaven.repo.local=./.m2 dependency:resolve
  - echo $env:ENVIRONMENT
```

Read it inside a test case (`src/test/java/Test1.java`):
```java
public static String environment = System.getenv("ENVIRONMENT");
...
System.out.println("ENVIRONMENT value from test: " + environment);
```

**Result:** value printed successfully in both the pre-step logs and the test execution logs — see screenshots.

---

## Task 3: Forced Failure + Retry

Added an intentionally failing test to `Test1.java`:
```java
@Test(description = "Intentional failure to verify retry behavior")
public void test1_intentional_failure() {
    Assert.fail("Intentional failure to verify retry behavior");
}
```

Retry config (already present from Task 1's fix):
```yaml
retryOnFailure: true
maxRetries: 1
```

**Result:** confirmed from job logs that `Test_1` was attempted twice — the original run plus one retry — both failing as expected, since the assertion fails unconditionally. See screenshot/logs.

---

## Task 4: Linux/Unix Basics

Ran via Git Bash (Windows doesn't natively support grep/awk/sed in PowerShell).

**grep** — find lines containing FAIL or ERROR:
```bash
grep -E "FAIL|ERROR" sample.log
```
Searches for lines matching "FAIL" or "ERROR"; `-E` enables extended regex so `|` works as "or."

**awk** — print the 2nd column:
```bash
awk '{print $2}' sample.log
```
Splits each line by whitespace and prints the second field.

**sed** — find-and-replace:
```bash
sed 's/staging/production/g' sample2.txt
```
Replaces every occurrence of "staging" with "production" on each line.

**Chained with a pipe:**
```bash
grep "FAIL" sample.log | awk '{print $2}'
```
Filters to lines containing "FAIL," then extracts just the 2nd column from those filtered results — a realistic "find the failing tests, then get just their names" use case.

Sample input/output included in `/screenshots` folder.

---

Screenshots documenting successful job runs, log output, and evidence for each task are included in as Task 1, Task 2, Task 3, Task 4 -1, Task 4 -2 in this repo. Please refer to these alongside the notes above for visual confirmation of each result

## Environment notes
- OS: Windows 11
- Java: Temurin 17.0.20
- Maven: 3.9.16
- Unix tools (Task 4): run via Git Bash