# Using CRDTs Beyond Text Editors

[Video Recording](https://www.youtube.com/watch?v=4QkLD7JhD_I) |
[Slides](https://github.com/helsing-ai/talks/blob/master/using-crdts-beyond-text-editors/using-crdts-beyond-text-editors.pdf)

---

I recently gave a talk on Conflict-free Replicated Data Types (CRDTs) at the
2025 [Rustikon](https://www.rustikon.dev/) conference in Warsaw 2025.

The talk explores how Conflict-free Replicated Data Types (CRDTs) can be used
for much more than just collaborative text editing. It begins by highlighting
the general principles of CRDTs—distributed systems that synchronize data
automatically without a central server or a consensus protocol like Raft. 
While text editing has emerged as the most popular
example, CRDTs are equally valuable for general-purpose state sharing between
applications (think JSON semantics).

From there, the talk dives into two main approaches for implementing CRDTs.
Operation-based CRDTs rely on replicating a single-writer append-only log,
making them straightforward for well-connected systems where logs are easy to
share. However, for networks with limited bandwidth or intermittent
connectivity, delta-based CRDTs allow for sending only minimal changes. The talk
then explores this method’s advantages for constrained environments.

The third part of the talk is about API considerations exposing impartial and/or
potentially conflicted data to domain experts without bothering them too much.


---

## Literature

1. Shapiro et al., “Delta State Replicated Data Types” (2018)  
2. van der Linde et al., “Making δ-CRDTs Delta-Based” (2016)  
3. Liangrun Da & Martin Kleppmann, “Extending JSON CRDTs with Move Operations,” PaPoC ’24  
4. Paulo Sérgio Almeida, “Approaches to Conflict-free Replicated Data Types” (2023)  
5. Frédéric Guidec, Yves Mahéo, Camille Noûs, “Supporting Conflict-free Replicated Data Types in Opportunistic Networks” (2022)
6. Rinberg et al., "DSON: JSON CRDT using delta-mutations for document stores" (2022)
