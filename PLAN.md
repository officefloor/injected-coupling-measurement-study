# Pre-registration draft: an injection-aware coupling instrument

> **Status (20 October 2026): draft announcement, not registered.** Published before the
> companion [concentration-divergence study](https://github.com/officefloor/concentration-divergence-study)
> runs, so that the instrument it describes is on record before any framework is measured with
> it. Registration is the commit that closes the open decisions below and is tagged `plan-v1`;
> no measurement code exists at this draft. Any later change to the plan is its own commit
> after the tag.

## Why this study exists

The [sinew construct](https://github.com/officefloor/concentration-divergence-study/blob/main/SINEW.md)
measures coupling per method from the parameters it declares and the arguments it passes on.
That count cannot tell data that **rides the control flow** (threaded: declared by the method
and supplied by its caller as an argument) from data a **container supplies** (injected: a
Spring field or constructor parameter, a Guice binding), nor data **bound from a request** (a
`@RequestBody`, a `@PathVariable`). Two consequences, both recorded in `SINEW.md`:

1. A method's injected dependencies look like parameters to a count, while field injection
   is invisible to it, so arms built on different injection styles cannot be compared on `in`.
2. Data bundled into one object is **stamp coupling**: one parameter that carries many items.
   In the petclinic runs the Spring arm's largest god method, `addOwner(OwnerDto)`, has one
   parameter.

Until the count knows which parameters are injected and how much data each carries, no
experiment that compares injection styles can be read. This study builds the instrument and
validates it against **the containers themselves**: a framework knows exactly which parameters
it injects, so the framework is the oracle, and no human has to decide what "injected" means.
**It makes no claim about any framework.** It produces a measurer, its validation, and the
effect of the correction on measurements already made.

## Scope: popular frameworks, and a door for the rest

The instrument covers the frameworks that are **common enough in sampled Java repositories to
validate on**. Which those are is decided by the sample, not by preference: after the draw
below, every framework declared by at least **5% of the usable repositories** (read from the
build file) is in scope, and the rule table and the oracle are built for each. Spring is
expected to dominate; Jakarta/CDI, Guice and Dagger may or may not clear the bar. A framework
below the bar is **recorded, not covered**: its parameters are classified threaded and every
such method carries a `framework_not_covered` flag, so a later analysis can exclude or
re-classify them.

Niche frameworks, OfficeFloor among them, are deliberately **not** in the validated instrument:
there is no corpus to validate on, and the author's own framework should not be the one the
instrument is tuned to. Instead the instrument is **extensible by design**: a framework is a
rule-table entry (its injection and binding annotations, its implicit-injection conventions,
its configuration files) plus an oracle adapter (how to ask that framework what it injects),
and the validation harness runs unchanged on any code the framework's user points it at. A
team that wants the measure for a framework outside the table adds the entry, runs the oracle
check on their own code, and reports the agreement; the harness prints the same tables this
plan prints. OfficeFloor's entry will be written and run on the petclinic application in the
exploratory section below, as the worked example of that door, and reported on its own.

## What was seen before this plan

- Paper one (complexity-concentration study): its confirmatory results, its MLCQ replication
  and its human-label read-out, all read. They motivate the divergence study, not this one.
- The divergence study's plan, whose JavaParser measurer and resolution rules this study
  extends. Its results have **not** been seen; it has not run.
- The petclinic exploratory observations in `SINEW.md`: on the OfficeFloor arm several top
  junctions were artefacts (`throw new …Exception(x)`, setters on injected entities). Those
  observations shape the artefact rules below, so the petclinic runs are **exploration
  material** and are never validation material.

## The question

> For the frameworks common in sampled Java repositories, can a static measure classify every
> parameter a method declares as **threaded**, **injected** or **bound**, from declarations
> alone and without building the project, so that it agrees with what the framework itself
> injects at least 95% of the time in both directions?

## Units and definitions

Measured per method and constructor with a body, on JavaParser with its symbol solver, exactly
the divergence study's units and resolution rules (internal call = resolves to the project's own
sources; JDK and unresolved calls are external). The instrument never builds the project; the
oracle does.

**Each declared parameter is one of:**

| class | rule (decided from declarations, annotations and configuration, never from usage) |
|---|---|
| **injected** | the framework supplies it: a parameter of a constructor the container calls (a component-annotated class, or a class registered in configuration, with the framework's implicit single-constructor rule applied where the framework has one), or a parameter carrying an injection annotation (`@Autowired`, `@Inject`, `@Qualifier`, `@Value`, `@Named`, and the framework's own) |
| **bound** | the framework fills it from a request, a message or an event: `@RequestBody`, `@PathVariable`, `@RequestParam`, `@RequestHeader`, `@ModelAttribute`, JAX-RS `@PathParam`/`@QueryParam` and entity parameters, servlet request and response objects, and the framework's own |
| **threaded** | every other parameter: supplied by a caller inside the project as an argument |

The rule table is one file per framework, versioned, and is the whole rule: an annotation not in
it is threaded until the table is amended in its own dated commit, and the oracle check is
what finds the omissions.

**Each argument passed on an internal call** is threaded, except under the artefact rules.
**Artefact rules**, from the petclinic exploration, fixed here: arguments to `throw new
…Exception(…)` and to constructors of `Throwable` subtypes do not count as outgoing data;
calls on an injected or bound receiver (a setter on an injected entity, a repository call)
count their arguments as outgoing data only when at least one argument is itself threaded or
derived from a threaded value within the method (an intraprocedural dataflow check, by name).

**Stamp coupling**: for each threaded or bound parameter, the number of **distinct data items
read from it in the method body**: field accesses and getter calls (a no-argument method whose
name starts with `get`, `is` or `has`, or a record accessor) on the parameter or on a local
alias of it. A parameter read whole (passed on, compared, stored) counts as one item. Reported
beside the parameter count, never folded into it without saying so.

**Framework detection** per repository from the build file and imports, recorded per
repository and per method (which framework's rule classified each parameter).

## The container oracle

For each covered framework, an **oracle adapter** asks the framework, on the repository's
compiled classes, which constructor and method parameters it injects and which it binds,
without running the application:

- **Spring**: component scanning and bean-definition reading over the compiled classes
  (`ClassPathScanningCandidateComponentProvider`, the configuration-class reader) and the
  framework's own constructor selection (`AutowiredAnnotationBeanPostProcessor`'s candidate
  constructors), plus the request-mapping handler method resolution for bound parameters. No
  `ApplicationContext` is refreshed, so no database or network is needed.
- **Guice**: module introspection (`Elements.getElements`) and `InjectionPoint` for every bound
  type's constructors and members.
- **Dagger**: the generated factories and components, read from the compiled output.
- **CDI/Jakarta**: Weld SE bootstrapped in discovery mode on the archive, bean definitions
  read, no deployment.

A framework whose adapter cannot be written without starting the application is reported as
**oracle unavailable** and leaves the validated set; it stays in the rule table, with that
limit named. The oracle needs the project to **build**: paper one's classpath validation found
that about a third of sampled Java repositories build with their own wrapper and no edits, so
the validation runs on the repositories that build and reports the ones that do not, with the
reason. Building is a selection on buildability, not on any result, and the instrument itself
never needs it.

## Material

**Open source, sampled.** Java repositories drawn by `open-source-repository-sampler` with
**seed 4** (seeds 1 to 3 are used or reserved), stratified by tier as the sampler does, to
**100 usable** repositories, with repositories in paper one's samples, in MLCQ or in the
divergence study's seed-3 sample excluded. The 5% scope bar is read from this draw. Then,
working down the draw order, the repositories that declare a covered framework are built with
their own Maven or Gradle wrapper (tests skipped, a fixed timeout, a fixed JDK order, exactly
paper one's step 4b rules) until **at least 20 built repositories per covered framework**, or
the draw is exhausted; every attempt is logged.

**OfficeFloor, exploratory only.** The petclinic OfficeFloor arm and the OfficeFloor tutorials
(the author's; the only OfficeFloor corpus), with OfficeFloor's rule-table entry and an oracle
adapter on its compiler. Reported on its own, never pooled, and never part of the decision.

## Primary estimand and decision

Over every parameter of every method in the built repositories, the instrument's class
against the oracle's, pooled per framework:

- **Sensitivity** on the injected class: the share of the parameters the oracle injects that
  the instrument classifies injected;
- **Specificity**: the share of the parameters the oracle does not inject that the instrument
  classifies not injected;

each with a 95% percentile bootstrap over **repositories** (2000 resamples, seed 12345).

**Decision, per covered framework.** *Validated* if the lower bound of both is at least
**0.95** on at least 20 built repositories. *Validated with a known limit* if 0.95 is met on
sensitivity and specificity is between 0.90 and 0.95 (the instrument over-classifies as
injected, which under-counts `in`, the conservative direction for the containment claim); the
limit is named. *Not validated* otherwise, or on fewer than 20 repositories, and the instrument
is not used for any comparison involving that framework until it is. The same two numbers for
the **bound** class are a secondary with the same constants.

Every disagreement is classified by cause (annotation not in the table, implicit convention
not in the table, configuration-supplied bean, meta-annotation, generated code, oracle
limitation) and the table is amended in a dated commit **after** the primary is read; the
amended instrument is re-run and reported as a second, labelled pass, never in place of the
first.

## The human audit: small, and only for what no container knows

Two things have no container oracle: the **stamp count** (what a method reads from a
parameter) and the **artefact rules** on outgoing arguments. For these, 200 items frozen
before the classifier is written (100 threaded or bound parameters for their stamp count, 100
internal-call arguments for the artefact rules), drawn by seed from the built repositories,
labelled once by the author and once by an AI agent under a published instruction, with
disagreements reconciled and recorded. Reported as the share of exact matches and the Spearman
correlation of the stamp counts, with no decision attached: the audit finds systematic errors;
it is not a validation and is not presented as one.

## The effect of the correction: exploratory, reported beside the validation

On the petclinic runs already measured (`blind-202608100006`, both arms, every checkpoint) and
on the divergence study's measurements once they exist, recompute each coupling definition
three ways and report the difference:

- **naive**: every parameter counts (the divergence study's measure);
- **injection-aware**: injected parameters excluded from `in`, bound parameters kept;
- **stamp-weighted**: each threaded or bound parameter counts its stamp items.

For each: the share of complexity area in catchments, pipes and junctions per arm; the rank
correlation of the three `in` counts; the identity of the top ten junctions per arm, and how
many of the OfficeFloor artefacts named in `SINEW.md` remain. This is description, not a test:
the petclinic runs shaped the artefact rules, and the OfficeFloor entry is the author's.

## Fixed in advance

- Java only. Parameter classes are decided from declarations, annotations and configuration,
  never from usage; the rule table is the whole rule.
- Seed 4, 100 usable repositories, the 5% scope bar, at least 20 built repositories per
  framework, the 0.95 and 0.90 constants, the 200 audit items, the bootstrap (2000, seed 12345).
- The oracle never starts the application; an adapter that would have to is reported as
  unavailable rather than written.
- Generated data is never edited; a fault is fixed in the code, the output deleted and the
  step re-run, and the re-run recorded.

## Open decisions, to close before registration

1. Whether Spring's bean-definition reading can be driven on compiled classes without a
   refreshed context in every case met (configuration classes with conditions are the risk).
2. The build rules carried from paper one's step 4b, or amended (JDK order, timeout).
3. Whether Kotlin sources in Java repositories are skipped (likely) or measured.
4. The agent instruction for the audit's second label.

## Procedure

1. Close the open decisions; commit; tag `plan-v1`.
2. Draw the sample; read the scope bar; commit the draw, the covered frameworks and the build
   log before any classifier code exists.
3. Build the classifier on **exploration material** only (the petclinic runs, the OfficeFloor
   tutorials); freeze the audit items.
4. Run the instrument and the oracle once on the built repositories; compute the primary;
   commit its hash. Then the audit, the error analysis, the amended second pass.
5. Report, then the exploratory recomputation.

## Dependencies and what comes after

Reuses the divergence study's measurer and resolution rules and paper one's build rules. The
[flow-injection containment experiment](https://github.com/officefloor/flow-injection-containment-experiment)
is gated on this instrument being validated for Spring (its DI arm) and on an OfficeFloor
entry whose oracle check on the petclinic arm is reported; this study is gated on nothing,
since a correct count of what is threaded is useful whatever the construct turns out to be
worth.

## Interests

The author created OfficeFloor. The validated instrument is built and tested on frameworks the
author does not own, against those frameworks' own answers; OfficeFloor enters only through the
extension door, on the author's own corpus, reported separately and never part of the decision.
