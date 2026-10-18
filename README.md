# apk2as maven repository

Hosts the `com.reno.plugin.gradleX` Gradle plugin that rebuilds an APK from the
Android Studio project produced by apk2as.

## Use it

Four lines in the decoded project's `settings.gradle` — apk2as already writes
them, so a freshly decoded project needs no editing:

```groovy
pluginManagement {
    repositories {
        maven { url 'https://raw.githubusercontent.com/XiaoJiu-nun/apk2as-maven/main' }
        google()
        mavenCentral()
    }
}
dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        maven { url 'https://raw.githubusercontent.com/XiaoJiu-nun/apk2as-maven/main' }
        google()
        mavenCentral()
    }
}
include ':app'
```

Then build with the wrapper that ships with the project:

```
gradlew assembleDebug          # Windows: gradlew.bat assembleDebug
```

## Requirements

| | |
|---|---|
| JDK | 17 or newer (`JAVA_HOME`) |
| Android SDK | platform 34, build-tools 34.0.0 |
| NDK | 25.2.9519653 (only for projects with native code) |
| Gradle | 8.0, downloaded by the wrapper |

First build downloads Gradle 8.0 and the AGP/SDK components, so it needs
network access. The plugin itself only touches the project directory.

## What is published

```
com/reno/plugin/gradleX/com.reno.plugin.gradleX.gradle.plugin/1.0.5/*.pom
com/reno/plugin/Plugin/1.0.5/Plugin-1.0.5.jar          # ~32 MB
com/reno/plugin/Plugin/1.0.5/Plugin-1.0.5.pom
```

Every file carries `.md5`, `.sha1`, `.sha256` and `.sha512` checksums.

The jar is a **thinned** fat jar. The upstream plugin is 167 MB because it
bundles all of AGP, Gradle, Groovy, ICU4J, Netty and BouncyCastle; a static
reachability pass over the constant pool from the plugin's own entry points
keeps only the ~10 k classes it can actually load, plus every non-class
resource. Measured: the same APK repacks byte-for-byte identically
(64112/64112 classes, 0 missing) on the thinned jar.

## Class integrity check

Every build that re-runs the dex merge compares the classes declared by the
project's smali sources against the classes actually present in the merged dex:

```
class integrity: 64112 declared by smali, 64112 in merged dex, 0 missing
```

A shortfall fails the build and lists the offending types, because an APK that
is missing classes still installs and launches - it only dies when one of them
is loaded, which is far harder to diagnose than a build error.

Classes present in the dex but absent from the source are expected (AGP
regenerates R, the javac step adds BuildConfig, instrumentation adds helpers)
and are ignored.

Set `apk2as.classIntegrity` in `gradle.properties` to tune it:

```properties
apk2as.classIntegrity=warn   # report but keep building; default is "fail"
apk2as.classIntegrity=off    # skip the check entirely
```

## Publishing a new version

```bash
python publish.py <github-owner>       # thin jar + maven layout + checksums
git init && git add . && git commit -m "release 1.0.5" && git push
```

`publish.py` recompiles the plugin from `plugin-src`, thins it, writes the
layout below `apk2as-maven/`, and rewrites the decoder template's repository URL
to the owner you pass. Rebuild `apk2as.jar` afterwards so the embedded template
matches.

`apk2as-maven/` **is** the repository root: push its contents (not the directory)
so that `<raw-url>/com/reno/plugin/...` resolves. Serving from
`raw.githubusercontent.com` needs no extra files and works with a plain
`maven { url ... }`.

Note the hard limit: **raw refuses files over 100 MB**, which is why the jar is
thinned.

## Troubleshooting

**`Could not find com.reno.plugin.gradleX:...`** — the URL is wrong or the
branch is not `main`. Check that
`.../main/com/reno/plugin/gradleX/com.reno.plugin.gradleX.gradle.plugin/maven-metadata.xml`
exists; if it 404s, the repository layout was not committed at the branch root.

**Dependency resolution failures mentioning guava / animal-sniffer** — the POM
declares no dependencies on purpose: the jar is self-contained. A POM that lists
its bundled contents as dependencies makes Gradle fetch a second copy of each.
