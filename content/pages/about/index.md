+++
title = "About"
template = "info-page.html"
path = "about"
+++

I am a software developer
who enjoys building robust and high-performance systems.
Currently, I am focusing on backend development,
leveraging the uncompromising safety and speed of Rust,
alongside the established presence of Go.

## Architecting the Backend

My current efforts are directed toward backend architecture.
I developed [CommitBridge],
recognizing the potential of Rust within server-side environments.
I am also mastering Go to expand the `CommitBridge` ecosystem
with satellite projects.

## Open Source Stewardship & Ecosystem Development

The open-source community significantly shaped my professional evolution,
particularly within the Rust game development ecosystem.
While exploring the Bevy engine
shortly after its initial release in August 2020,
I discovered that vector graphics support was missing.

Leveraging my prior experience with the [`lyon`] library,
I created [`bevy_prototype_lyon`]:
an ergonomic wrapper designed to seamlessly integrate vector graphics into Bevy.
This crate rapidly became one of the most starred repositories
in the early days of the engine's ecosystem.

As the project scaled, I transferred ownership to a dedicated contributor,
ensuring the plugin's future was in good hands.
This delegation enabled me to join the Entity Inspector working group,
where I dedicated several months to developing [Feathers Inspector].
In addition to these projects,
I have also sporadically [contributed to the core Bevy repository],
mostly through documentation enhancements
and small refactorings aimed at code quality.

## Reverse-Engineering & Formative Pursuits

My approach to software engineering is accompanied by a strong curiosity
about the inner workings of complex systems.
During the early 2020 lockdown,
I successfully reverse-engineered a good part of the slither.io client,
constructing a [custom client] utilizing the [`ggez`] framework.
Before learning Rust,
I cultivated my foundational programming knowledge
through extensive work with C# and C++,
engaging in personal game development projects
both from scratch and via established engines like Unity and Godot.

## Where It All Began

While my current focus
resides between tool development and backend architecture,
my first contact with programming
was sparked by a Sharp EL-5250 scientific calculator
when I was going in middle school.
Thanks to its extensive programming manual,
I wrote a text-based RPG battle simulator
utilizing a rudimentary, Basic-like syntax.
From playing around with household calculators as a child,
to building distributed open-source solutions today,
that insatiable interest and fascination with computation
remain the driving force behind my career.

<!-- LINKS -->
[`bevy_prototype_lyon`]: https://github.com/rparrett/bevy_prototype_lyon
[contributed to the core Bevy repository]: https://github.com/bevyengine/bevy/pulls?q=is%3Apr+author%3ANilirad
[custom client]: https://github.com/Nilirad/sir-eolith
[Feathers Inspector]: https://github.com/alice-i-cecile/feathers_inspector
[`ggez`]: https://ggez.rs/
[`lyon`]: https://github.com/nical/lyon
[CommitBridge]: @/blog/commit_bridge_intro.md
