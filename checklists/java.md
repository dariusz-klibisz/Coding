# Java Review Checklist

> Use with [`../languages/java.md`](../languages/java.md) and
> [`general.md`](general.md).

- [ ] Formatter + Error Prone/SpotBugs clean (`JAVA-TOOL-*`).
- [ ] `record`/sealed types + pattern-matching `switch` used for data holders
      and closed alternatives (`JAVA-IDIOM-01/02`).
- [ ] No raw generic types; `Optional` used only as a return type
      (`JAVA-TYPE-02/03`).
- [ ] No swallowed exceptions; cause chained on wrap (`JAVA-ERR-02/03`).
- [ ] try-with-resources for every `AutoCloseable` (`JAVA-RES-01`).
- [ ] `java.util.concurrent` preferred over hand-rolled `synchronized`/
      `wait`/`notify`; thread pools explicitly bounded (`JAVA-CONC-01/04`).
- [ ] Cooperative interruption used, never `Thread.stop()`/`suspend()`
      (`JAVA-CONC-03`).
- [ ] Parameterized SQL; no native-Java deserialization of untrusted data;
      `SecureRandom` for security-relevant values (`JAVA-SEC-*`).
- [ ] Dependency scanning in CI; lockfile/dependency management committed
      (`JAVA-SEC-04`).
- [ ] JUnit 5 + AssertJ; Testcontainers for realistic integration tests
      (`JAVA-TEST-*`).
