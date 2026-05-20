\# Remove Litematica/Fast-Place Conflicting Checks \& Rewrite Build Workflow



\## Background



Grim is a Minecraft anticheat plugin. Mods like \*\*Litematica\*\* (schematic building) and \*\*accurate/fast block placement\*\* mods trigger several "scaffolding" anticheat checks because they:



1\. \*\*Place blocks without looking at them\*\* — Litematica's "easy place" auto-places blocks at schematic positions without requiring the player's crosshair to aim at the correct face. This triggers \*\*RotationPlace\*\*.

2\. \*\*Place multiple blocks per tick\*\* — Fast placement mods send multiple block placement packets in a single tick. This triggers \*\*MultiPlace\*\*.

3\. \*\*Place blocks against hidden faces\*\* — Accurate placement can place against block faces that aren't visible from the player's position. This triggers \*\*PositionPlace\*\*.

4\. \*\*Send fabricated cursor positions\*\* — Auto-placement sends synthetic cursor data that may not match a real raytrace. This triggers \*\*FabricatedPlace\*\*.

5\. \*\*Duplicate rotation patterns\*\* — Automated placement creates repetitive identical rotation deltas. This triggers \*\*DuplicateRotPlace\*\*.



\## Checks to Remove



The following checks in `common/src/main/java/ac/grim/grimac/checks/impl/scaffolding/` will be \*\*deleted\*\*:



| Check | File | What it detects | Why it conflicts with Litematica/fast-place |

|---|---|---|---|

| \*\*RotationPlace\*\* | `RotationPlace.java` | Player placed a block while not looking at it | Litematica "easy place" doesn't require aiming at the block face |

| \*\*MultiPlace\*\* | `MultiPlace.java` | Multiple blocks placed in a single tick | Fast-place mods send rapid placement packets |

| \*\*PositionPlace\*\* | `PositionPlace.java` | Block placed against a hidden/impossible face | Accurate placement places from angles vanilla doesn't allow |

| \*\*FabricatedPlace\*\* | `FabricatedPlace.java` | Out-of-bounds cursor position sent | Auto-placement sends calculated cursor positions that can be slightly out of bounds |

| \*\*DuplicateRotPlace\*\* | `DuplicateRotPlace.java` | Duplicate rotation deltas while placing | Automated rapid placement creates repetitive rotation patterns |



\### Checks to KEEP (these are legitimate anti-exploit/crash checks):



| Check | File | Reason to keep |

|---|---|---|

| \*\*InvalidPlaceA\*\* | `InvalidPlaceA.java` | Catches NaN/Infinity cursor — prevents crashes, not a legitimate mod behavior |

| \*\*InvalidPlaceB\*\* | `InvalidPlaceB.java` | Catches impossible block face IDs (out of 0-5 range) — anti-crash check |

| \*\*AirLiquidPlace\*\* | `AirLiquidPlace.java` | Detects placing against air/liquid — prevents real exploits, not triggered by Litematica |

| \*\*FarPlace\*\* | `FarPlace.java` | Detects placing from beyond vanilla reach distance — anti-exploit, Litematica works within normal reach |



> \[!IMPORTANT]

> \*\*FarPlace\*\* is kept because Litematica respects vanilla block reach distance. \*\*AirLiquidPlace\*\* is kept because it prevents actual block-against-air exploits. Both are valid anti-exploit checks that won't affect legitimate mod usage.



\## Proposed Changes



\### 1. Delete Scaffolding Check Files



\#### \[DELETE] \[RotationPlace.java](file:///d:/Projek-Github/Grim/common/src/main/java/ac/grim/grimac/checks/impl/scaffolding/RotationPlace.java)

\#### \[DELETE] \[MultiPlace.java](file:///d:/Projek-Github/Grim/common/src/main/java/ac/grim/grimac/checks/impl/scaffolding/MultiPlace.java)

\#### \[DELETE] \[PositionPlace.java](file:///d:/Projek-Github/Grim/common/src/main/java/ac/grim/grimac/checks/impl/scaffolding/PositionPlace.java)

\#### \[DELETE] \[FabricatedPlace.java](file:///d:/Projek-Github/Grim/common/src/main/java/ac/grim/grimac/checks/impl/scaffolding/FabricatedPlace.java)

\#### \[DELETE] \[DuplicateRotPlace.java](file:///d:/Projek-Github/Grim/common/src/main/java/ac/grim/grimac/checks/impl/scaffolding/DuplicateRotPlace.java)



\---



\### 2. Update CheckManager.java



\#### \[MODIFY] \[CheckManager.java](file:///d:/Projek-Github/Grim/common/src/main/java/ac/grim/grimac/manager/CheckManager.java)



Remove the 5 deleted checks from the `blockPlaceChecks` builder (lines 218-234). The remaining valid checks (`InvalidPlaceA`, `InvalidPlaceB`, `AirLiquidPlace`, `FarPlace`, etc.) stay.



\---



\### 3. Remove Config Entries from All Locale Files



\#### \[MODIFY] Config files in `common/src/main/resources/config/`



Remove the `RotationPlace:`, `FabricatedPlace:`, and `PositionPlace:` config sections from all locale YAML files (en.yml, de.yml, es.yml, etc.). These are the only removed checks that have config entries.



\---



\### 4. Rewrite GitHub Actions Build Workflow



\#### \[NEW] \[build.yml](file:///d:/Projek-Github/Grim/.github/workflows/build.yml)



A simplified workflow that:

\- Triggers on push to `main` branch and on manual `workflow\_dispatch`

\- Sets up JDK 21 (Temurin)

\- Uses Gradle with caching

\- Builds both `bukkit` and `fabric` modules

\- Uploads build artifacts (JARs) as GitHub Actions artifacts



\#### \[DELETE] \[build-and-publish.yml](file:///d:/Projek-Github/Grim/.github/workflows/build-and-publish.yml)

\#### \[DELETE] \[gradle-publish.yml](file:///d:/Projek-Github/Grim/.github/workflows/gradle-publish.yml)

\#### \[DELETE] \[release.yml](file:///d:/Projek-Github/Grim/.github/workflows/release.yml)

\#### \[DELETE] \[codeql-analysis.yml](file:///d:/Projek-Github/Grim/.github/workflows/codeql-analysis.yml)



Replace all existing workflows with a single clean build workflow. The old workflows reference private secrets (`MODRINTH\_ID`, `BUILD\_UPDATE\_URL`, etc.), private runners (`tenki-standard-autoscale`), and publishing logic you don't need.



> \[!WARNING]

> The existing workflows publish to Modrinth and use private API endpoints. The new simplified workflow \*\*only builds and uploads artifacts\*\*. If you later want to publish to Modrinth, you'll need to add those steps back.



\## Open Questions



> \[!IMPORTANT]

> \*\*FarPlace\*\*: Should I also remove FarPlace? Some "accurate placement" mods can place at extended reach, but standard Litematica/accurate-placement work within normal reach. Keeping it is safer for anti-cheat integrity.



\## Verification Plan



\### Automated Tests

\- Run `./gradlew build` locally to verify the project compiles without the deleted checks

\- Verify no remaining references to deleted classes cause compilation errors

