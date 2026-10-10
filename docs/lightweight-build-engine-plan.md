# Lightweight Android Build Engine — Implementation Plan

## Goal

Add an optional on-device build path for simple Android projects written in Java + Kotlin + XML, without requiring Ubuntu/PRoot for that path. Preserve the existing Gradle/PRoot builder as the compatibility fallback and do not change how Mobile Harness itself is built.

## What the repository currently does

- The host app is still built with Android Gradle Plugin, Kotlin Gradle Plugin, and CMake/NDK. This must remain unchanged.
- `app/src/main/java/com/jarves/mh/runtime/AndroidBuildSupport.kt` owns Android build phases, diagnostics, APK discovery/reuse, and the existing `androidGradleCommand(...)`.
- The current command helper selects `bash ./gradlew` or `gradle` and includes the PRoot-specific `/root/.gradle/init.d/pocketdev-android.gradle` and `/root/android-sdk/.../aapt2` paths. That command is not suitable for a direct Android-app process.
- `MainViewModel.kt` imports the build-support helpers and owns the UI/workflow integration. The current builder should be retained while the new path is introduced behind a strategy interface.
- `AndroidProjectTemplateGenerator.kt` currently generates Gradle-based XML and Compose templates. The existing templates should keep working; lightweight mode should be an explicit project/build option rather than silently changing generated projects.

## Recommended architecture

Introduce a separate strategy rather than replacing the current builder:

```text
Build request
   |
   v
AndroidBuildCoordinator
   |-- LightweightBuildEngine (on-device, no PRoot)
   |      |-- Project validator / capability check
   |      |-- Java compiler adapter
   |      |-- Kotlin compiler adapter
   |      |-- AAPT2 resource compiler/linker
   |      |-- D8 (class files -> DEX)
   |      |-- APK packager + signer
   |      |-- Build cache / diagnostics
   |
   |-- Existing Gradle/PRoot engine (fallback)
```

The lightweight engine must not shell out to an arbitrary executable or assume filesystem paths under `/root`. All tool paths should be resolved under app-private storage, and downloads must be version-pinned, integrity-checked, and license-reviewed.

## Important technical constraints

1. D8 does not compile Java or Kotlin source code. It converts JVM class files to DEX.
2. XML resources require AAPT2 compile/link, Android platform resources, a manifest, and a compatible Android API/platform package.
3. Kotlin needs a real Kotlin compiler and its runtime/dependencies. Kotlin compiler artifacts may be large and may rely on JVM/JDK APIs that are not available in Android's ART process. Validate compiler execution on supported Android versions before committing to an in-process design.
4. APK assembly also requires packaging and signing tools; an unsigned or incorrectly signed APK is not a successful build.
5. General Gradle projects can use plugins, AAR/JAR dependencies, annotation processors, Compose compiler plugins, manifest mergers, resource generators, and custom tasks. A small custom engine cannot claim universal Gradle compatibility.
6. C/C++ remains on the existing NDK/CMake path initially. It is outside the first lightweight Java/Kotlin/XML milestone.
7. Never remove the existing Gradle path until parity and fallback tests pass.

## Phased delivery

### Phase 0 — Inventory and compatibility contract
- Identify the current build entry points in `MainViewModel.kt`, progress/error models in `AndroidBuildSupport.kt`, and project template metadata.
- Define a build-engine selection setting: Automatic, Lightweight, or Gradle/PRoot.
- Automatic mode may select lightweight only when the project passes capability checks; otherwise it must use the current builder or explain why fallback is unavailable.
- Establish tests for cancellation, progress, stale APK detection, error reporting, and preserving existing projects.

### Phase 1 — Toolchain proof of concept (Java + XML)
- Prove on a real ARM64 Android device that pinned Java compiler, AAPT2, D8, APK packaging, and signing components can run without PRoot.
- Support a deliberately limited project contract first: one app module, Java sources, standard XML resources, manifest, platform APIs, and a controlled set of JAR dependencies.
- Build in a temporary staging directory; only publish the APK after successful packaging, verification, and signing.
- Do not download or execute tools without checksum/signature validation.

### Phase 2 — Kotlin support
- Test Kotlin compiler compatibility on-device separately before integrating it.
- Resolve Kotlin stdlib and compiler dependencies deterministically.
- Add Kotlin-to-class-file compilation, then feed class files into the same D8/packaging pipeline.
- If the Kotlin compiler requires unsupported JVM facilities, use a small bundled/runtime-managed JVM in app-private storage or keep Kotlin projects on the existing Gradle/PRoot path. Do not claim no-PRoot Kotlin support until this passes device tests.

### Phase 3 — Build cache and incremental work
- Cache compiled classes and resource intermediates using content hashes of source, resource, manifest, tool versions, SDK platform, and dependencies.
- Never rely only on timestamps for cache correctness.
- Bound cache size and provide a clear-cache action.
- Keep cache entries per project and toolchain version.

### Phase 4 — UI integration and fallback
- Show the selected engine and explain fallback reasons in the existing build UI.
- Map engine events into the existing `AndroidBuildPhase`, `AndroidBuildStage`, and `AndroidBuildIssue` models.
- Preserve cancellation, foreground task notifications, APK discovery, install/run flow, and current error diagnostics.
- Gradle/PRoot remains available for unsupported plugins, Compose, native C/C++, multi-module projects, and other advanced builds.

### Phase 5 — Acceptance tests
- Java + XML starter app builds offline after toolchain installation.
- Java + Kotlin + XML starter app builds offline after toolchain installation.
- APK passes package/signature verification and installs on the supported Android API range.
- A source/resource/dependency change invalidates the correct cache entries.
- Cancellation stops work and removes temporary files.
- Low-storage, missing SDK, corrupt tool download, compiler error, and signing error produce actionable messages.
- Existing Gradle/PRoot builds continue to work unchanged.

## Proposed code layout

- `runtime/build/AndroidBuildEngine.kt`: request/result/events and engine interface.
- `runtime/build/BuildCapabilityDetector.kt`: conservative project feature detection.
- `runtime/build/LightweightAndroidBuildEngine.kt`: orchestration only; no UI code.
- `runtime/build/toolchain/ToolchainManager.kt`: pinned downloads, checksums, versions, storage accounting.
- `runtime/build/toolchain/JavaCompilerAdapter.kt`
- `runtime/build/toolchain/KotlinCompilerAdapter.kt`
- `runtime/build/toolchain/Aapt2Adapter.kt`
- `runtime/build/toolchain/D8Adapter.kt`
- `runtime/build/toolchain/ApkPackager.kt`
- `runtime/build/cache/BuildCache.kt`
- `runtime/build/GradlePr ootBuildEngine.kt`: adapter around the existing PRoot/Gradle path (name can be normalized to `GradlePRootBuildEngine.kt` in implementation).

## First implementation boundary

The first code milestone should be a non-invasive engine interface plus capability detection and tests. Do not advertise the feature as working until the device proof-of-concept for compiler execution, resource linking, APK packaging, and signing passes. This plan document does not claim that the new build engine has been implemented or tested.
