# One Gerrit tree, two servlet worlds: the EE8/EE10 flavoured release

*By David Ostrovsky — Gerrit and JGit maintainer.*

## The question

JGit moved to `jakarta.servlet` (EE10) — so consuming current JGit already forces
a servlet-namespace decision. Jetty 12 ships EE8 and EE10 side by side. Gitiles is
adding an EE10 (jakarta) flavour of its own (uploaded for review). Meanwhile most
of Gerrit — and the large body of existing plugins — still lives on
`javax.servlet` (EE8). So the ecosystem is already
split, and every operator faces the same fork in the road:

- move to `jakarta.servlet` now and break every `javax.servlet` plugin you
  depend on, or
- stay on `javax.servlet` and get left behind as the libraries advance.

A single default can't serve both groups. Moving the default to EE10 breaks the
EE8 majority; staying EE8-only forever strands the early adopters.

So we asked a different question:

> **What if we produced *both* worlds — EE8 (`javax.servlet`) and EE10
> (`jakarta.servlet`) — from a single Gerrit source tree?**

No long-lived fork. No `next` or `servlet-4` branch. No duplicated codebase. Just one tree that
builds two releases. This is the announcement that it now works, end to end.

## The answer: `bazel build release release-ee10`

That one command produces **both** flavours, side by side:

```
bazel-bin/release.war        # EE8  — javax.servlet, Jetty 12 EE8, Guice 6  (the unchanged default)
bazel-bin/release-ee10.war   # EE10 — jakarta.servlet, Jetty 12 EE10, Guice 7
```

No flag, no second checkout, no configure step. The EE8 `release.war` is
**byte-for-byte unchanged** from before the work; the EE10 WAR is built right
next to it. A single runtime loads exactly one flavour — and it has to: the
generated EE10 classes keep the **same Java fully-qualified names** as their
canonical EE8 siblings, so the two flavours can never share a classpath. That is
deliberate. It is what makes each flavour a drop-in replacement for the other,
and selection happens entirely at build time — never at runtime.

## How it works, in four ideas

1. **A flavour build setting, owned by the build tooling.** The servlet flavour
   is a single Bazel setting (`ee8` default, `ee10`), and every flavour-bearing
   dependency — servlet API, Jetty adapter, Guice tier, the JGit/Gitiles servlet
   libraries, Gerrit's own servlet boundary — selects on it.

2. **Self-transitioning `-ee10` targets.** The `-ee10` targets carry their own
   build configuration transition, so they build the jakarta subgraph *regardless
   of the command line*. That's why both flavours build in one invocation: EE8
   and EE10 each compile consistently on their own flavour.

3. **The servlet boundary is generated, not hand-maintained.** Gerrit's `httpd`
   package is mechanically rewritten from the canonical `javax.servlet` sources to
   `jakarta.servlet` by a small, shared, dependency-free transform that touches
   only servlet/Jetty import prefixes and preserves package names and line
   numbers. A handful of genuinely-divergent Jetty-12 files get hand-written EE10
   overlays. A structural guard test fails if any `javax.servlet` residue leaks
   into the generated flavour.

4. **Bridges in both directions.** Each library takes the shortest path from its
   own canonical flavour: JGit is already `jakarta.servlet`-canonical, so it
   bridges *backward* to javax; Gerrit and Gitiles are still `javax.servlet`-canonical,
   so they bridge *forward* to jakarta. As each project's canonical flavour
   eventually flips to jakarta, the asymmetry quietly disappears — and the project
   converges back to a single flavour.

## Plugins come along — including standalone builds

Servlet-coupled plugins ship one artifact per flavour, each marked with a
`Gerrit-Flavour: ee8|ee10` manifest entry; audited servlet-neutral plugins ship a
single `any` artifact. Adding the EE10 flavour to a plugin is **one line** — a
sibling target with `flavour = "ee10"` — and the shared build macro runs the
transform, injects the marker, selects the jakarta plugin API, and wraps the jar
in the EE10 transition.

Three plugins are already migrated, one of each kind:

- **`gitiles`** — core plugin (consumes the generated jakarta `gitiles-servlet-ee10`);
- **`plugin-manager`** — core plugin;
- **`javamelody`** — a custom, servlet-coupled plugin that also exposes a
  **standalone build mode** (built outside the Gerrit tree against the published
  plugin API).

To make the standalone EE10 build work, the jakarta plugin API is published to
Maven Central **side by side** with the EE8 one — same group and version,
distinguished by an `-ee10` artifactId suffix:

| EE8 (default) | EE10 (jakarta) |
|---|---|
| `com.google.gerrit:gerrit-war` | `com.google.gerrit:gerrit-war-ee10` |
| `com.google.gerrit:gerrit-plugin-api` | `com.google.gerrit:gerrit-plugin-api-ee10` |

The whole loop is proven: javamelody's standalone build, fed
`gerrit-plugin-api-ee10`, transforms its `javax.servlet` sources to jakarta and
links them against a **jakarta-only** plugin API — exactly what a non-transformed
javax import could never do.

## How "both worlds from one tree" holds up

The flavour boundary is remarkably thin. Comparing the EE8 and EE10 plugin-API
jars byte for byte: of **36,893** shared entries, **36,610 are byte-identical** —
only **283** differ, and every one of them sits on the flavour boundary: the
servlet classes and the Guice 6→7 cascade they pull in (`com.google.inject.*`
plus the plugin-loader classes that touch Guice). The two WARs differ in exactly
the expected places (servlet API 4.0.1 vs 6.1.0, Jetty EE8 vs EE10, Guice 6 vs 7,
the JGit/Gitiles servlet jars, and the per-flavour plugin jars) and nowhere else.
The EE8 default is untouched.

## The historic backdrop: why `javax` became `jakarta`

None of this would be necessary if a package namespace hadn't moved out from under
the entire Java server ecosystem. The short version:

- Java EE was built by Sun and then Oracle under the `javax.*` namespace.
- In 2017 Oracle handed Java EE to the Eclipse Foundation, where it was renamed
  **Jakarta EE**. Oracle kept the `javax` trademark, so Eclipse could not evolve
  the old packages.
- **Jakarta EE 9 (2020) renamed every `javax.*` package to `jakarta.*`** — a
  "big bang" with essentially no functional change. `javax.servlet` became
  `jakarta.servlet`, and that single rename rippled through Servlet 5/6, Tomcat
  10+, Jetty 12, Guice's servlet extension (DI), Spring 6, and everything built
  on them.

So a project that wants a current servlet container or a modern Jetty has to be on
`jakarta.servlet`; a project that must keep a large body of existing
`javax.servlet` integrations cannot move yet. JGit has already made the jump and
is `jakarta.servlet`-canonical; Gitiles is adding a jakarta flavour of its own (in
review) but, like Gerrit and most of its plugins, remains `javax.servlet`-canonical.
The ecosystem is split straight down that namespace line — which is exactly the
split this work lets a single Gerrit tree straddle, instead of forcing a flag day.

## Not a new idea for Gerrit

Carrying two paths through a migration — and retiring the older one once the move
is complete — is familiar territory. ReviewDb and NoteDb, GWT and PolyGerrit,
ChangeScreen and ChangeScreen2 each ran side by side during their transitions.
The lesson from all of them: every path has to be a real, supported product —
built, tested, documented — not a hidden build variant, and the superseded one is
eventually removed. The EE8 flavour is on exactly that trajectory.

The shape of the solution mirrors Jetty 12's own EE8/EE10 architecture, which
ships both servlet flavours from a single project. The idea was raised by Nasser
Grainawi in Gerrit community discussion:

> Hmm, that's an interesting idea. Another idea geared towards the future support
> problem … was what if we followed the jetty approach in jgit/gitiles/gerrit and
> have in-tree support for both servlet-4 and servlet-6+. It's absolutely more
> overhead to maintain both and maybe it falls apart when you start looking at
> other dependencies, but it could be a path to providing a newer servlet
> version …

This is the answer to that "what if": it does not fall apart, the overhead is a
single transform plus a thin per-flavour boundary, and both worlds ship from one
tree.

## Status

The work spans JGit, Gitiles, Bazlets, Gerrit, the core and custom plugins, and
both build modes — in-tree and standalone. Both major changes were thought
through before any code: detailed design proposals — *Add first-class EE8
servlet modules to JGit* and the *Flavoured Gerrit release* (this announcement) —
were written and submitted first, and the full change series is now uploaded for
review (all linked in References below). Publishing the EE10 flavour to operators
is the rollout step that remains; once all stakeholders have moved, the EE8
flavour will be retired and the project will converge back to one servlet world.

A welcome side effect: this also untangles the long-standing **Gerrit JGit
submodule maintenance**. Until now Gerrit consumed JGit through a dedicated
`servlet-4` branch (kept on `javax.servlet`), which had to be kept alive by
continually merging up from JGit `master` — a standing maintenance burden. With
first-class EE8 modules in mainline JGit, each Gerrit branch can track its
natural JGit branch directly: Gerrit `master` → JGit `master`, and a Gerrit
stable branch → its matching JGit stable branch (e.g. `gerrit-3.12` → JGit
`stable-7.4`). The `servlet-4` branch, and the merge-up churn it required, can be
retired.

One tree. Two worlds. One command.

## References

The flavoured release was designed before it was coded. The design proposals
come first, then the change series that implement them:

- **Gerrit EE8/EE10 flavoured release — design proposal** (this work):
  <https://www.gerritcodereview.com/design-docs/ee8-ee10-flavoured-release.html>
- **JGit EE8 servlet bridge — design proposal**:
  <https://github.com/davido/jgit-ee8-servlet-bridge-design-proposal>
- **JGit issue** — *[FEATURE] Add first-class EE8 (`javax.servlet`) modules to
  JGit*: <https://github.com/eclipse-jgit/jgit/issues/272>
- **JGit EE8 servlet bridge change series (in review)** — the foundational
  prerequisite the whole "two flavours from one tree" idea is built on; without
  first-class EE8 JGit modules there is nothing for the EE8 flavour to consume.
  GerritHub topic `ee8-servlet-bridge`:
  <https://review.gerrithub.io/q/topic:%22ee8-servlet-bridge%22>
- **Gerrit servlet bridge change series (in review)** — once JGit offers the EE8
  bridge, Gerrit adopts it: the next step in the chain. Gerrit topic
  `ee8-servlet-bridge`:
  <https://gerrit-review.googlesource.com/q/topic:%22ee8-servlet-bridge%22>
- **De-leaking incidental `javax.servlet` from core/plugins** — the groundwork
  that makes the swap possible: it minimizes the servlet surface, separating code
  that is genuinely servlet-coupled from code that only incidentally touches the
  Servlet API, so only the real boundary has to carry two flavours. Topic
  `de-leak-servlet-api`:
  <https://gerrit-review.googlesource.com/q/topic:de-leak-servlet-api>
- **EE10 flavour change series (in review)** — and only then the "two flavours
  from one tree" story: the follow-up that adds the second flavour on top,
  spanning Gerrit, Bazlets, Gitiles, and the core & custom plugins. Topic
  `ee10-flavour`:
  <https://gerrit-review.googlesource.com/q/topic:ee10-flavour>
- **JGit maintenance options analysis** — the trade-off study behind switching
  Gerrit's JGit submodule from `servlet-4` to `master`:
  <https://davido.github.io/gerrit-jgit-maintenance/>

