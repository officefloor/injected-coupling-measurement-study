# Injected-coupling measurement study

A static Java instrument that classifies every parameter as **threaded** (rides the control
flow), **injected** (supplied by a container) or **bound** (filled from a request), and counts
the data items a parameter carries (stamp coupling). It is validated against **the containers
themselves**: each covered framework is asked, on compiled classes and without running the
application, which parameters it injects, and the instrument must agree at least 95% of the
time in both directions.

It covers the frameworks common in sampled Java repositories (decided by the sample, Spring
expected to dominate) and is **extensible**: a framework is a rule-table entry plus an oracle
adapter, and the validation harness runs unchanged on any code its user points it at. Niche
frameworks, OfficeFloor included, are not in the validated instrument; OfficeFloor's entry is
run on the author's own corpus as the worked example of that door and reported separately.

It exists because the coupling-divergence measure of the
[concentration-divergence study](https://github.com/officefloor/concentration-divergence-study)
cannot tell those parameter classes apart, and no comparison of injection styles can be read
until it can.

**The plan is [`PLAN.md`](PLAN.md)**, a draft announcement: not yet registered, no code yet.
This repository will hold the method only; its output is never committed.
