# scan-gradle-plugin ASM Reproducer

Demonstrates that `scan-gradle-plugin:3.1.5` bundles an outdated copy of
`groovyjarjarasm.asm.ClassReader` that fails to read Java 17+ class files,
and shadows Groovy's own copy which supports them correctly.

## The Bug

`scan-gradle-plugin` physically embeds `groovyjarjarasm.asm.ClassReader` inside
its JAR. The bundled version only supports class file major versions up to 60
(Java 16). When the plugin is applied, Java's parent-first classloader delegation
causes the plugin's outdated copy to shadow Groovy's own `ClassReader`
(Groovy 3.0.25 supports up to Java 25). Any code in the same build that uses
`groovyjarjarasm.asm.ClassReader` against a Java 17+ compiled class file will
fail with:

```
java.lang.IllegalArgumentException: Unsupported class file major version 61
    at groovyjarjarasm.asm.ClassReader.<init>(ClassReader.java:195)
```

## Prerequisites

- JDK 17+

## Reproduce

```bash
./gradlew demonstrateBug
```

Expected output:

```
=== scan-gradle-plugin ASM Reproducer ===
Class file   : Hello.class
Major version: 61 (61 = Java 17)
ClassReader  : file:.../.gradle/caches/.../scan-gradle-plugin-3.1.5.jar

RESULT: FAILED — Unsupported class file major version 61

BUG CONFIRMED: scan-gradle-plugin bundles an outdated groovyjarjarasm.asm.ClassReader
that does not support Java 17+ class files (major version >= 61).
Its copy shadows Groovy's own ClassReader, which supports Java 17+.
```

## Fix

The plugin should either:

- **Not bundle** `groovyjarjarasm.asm.*` and rely on Groovy's copy at runtime, or
- **Bundle a version of ASM** that supports at least Java 17 (ASM 7.0+)
