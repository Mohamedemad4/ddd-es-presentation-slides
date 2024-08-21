---
theme: gaia
_class: lead
paginate: true
backgroundColor: #fffffd
# backgroundImage: url('https://marp.app/assets/hero-background.svg')
marp: true
---

<!-- ![bg left:40% 80%](https://marp.app/assets/marp.svg) -->

# **DDD, Event sourcing and CQRS**

A Practical Guide

<!-- --- -->

<!-- # End goal

- Different way of thinking about backend.
- How event sourcing can accommodate system evolution be it technical or domain evolution. -->

## <!-- You will see us mention the word domain alot. We use it to express biz. proccesses and constraints. Your systems should be written as close to the biz. domain as possible. Even the abstractions should mirror operational and biz. abstractions your org uses. -->
---

<!-- # Style.

- Practical considerations and operational and implementation details.
- Philosophical underpinnings that will shape your thought process.
-->


# A little bit about us.

---


### Transport for Cairo
<!-- ![:65% 80%](./tfc_intro.png) -->

![w:800 center](./tfc_intro.png)

---

### We put microbuses in google maps
![](./tfc_intro_maps_gtfs.png)

---

# RouteLab
<style scoped>
section {
    font-size: 30px;
}
</style>

<!-- ![:65% 80%](./tfc_intro.png) -->
- Transit Data Collection and analysis **at scale**

- Models the complexity of informal transit

- Used in 10+ african cities

![bg right:65% 90%](./routelab_home.png)

---
<!-- ### From DB-centric design to Event Souring -->

## From DB-centric design to Event Sourcing

<!-- - No relational schema: No service data boundaries/ownership.  reduce service interdependancy for read operations. no contracts no circuts no n+1s. everyone is free to read from the event stream  Maybe this point should be discussed later because at the current time they won't be able to imagine it.-->

<!-- as we mentioned. Nothing is new under the sun all domains were once paper based and based on facts written on sheets paper. This is closer to that which allows you to do things more naturally. Your tech team and your operational team will have the same language with minimal drift. -->

- We used to keep most of the state on the server
- Increasing need for asynchronicity (due to field work conditions)
---
<style scoped>
h1 {
   margin-top:10%;
}
</style>


# Overview of DDD with Functional Event Sourcing

---


# So What is DDD basically?

Align software design closely with the business domain. The goal is to ensure that the code reflects the actual business processes, rules, and language.

<!-- ---

# Ok

و البعدو؟ -->

<!-- This is a very old idea. زي اليقلك الديف لازم يببقا فاهم الbussniess. But we take this a bit further -->

---

# What does software do?

### (mostly...)

<!-- i am looking for that image of the old lady that just says big data-->

It's about modeling reality.
a sequence of immutable facts.

<!--Fundementally all organizational and accounting practicies were doing this for *centries* way before *electricity* was a thing. We just get paid monies becase we allow them to do what they were already good at. But cheaper, faster,bigger and easier.
لما الموظفة تقلك روح اختم من كذا
brother you must understand
you are the RPC.

And when you complain about convuluted paper trail for a gov. operation.

you are complaining about bad system design.

These 2 things are more similar than we give credit for.
-->

---

![bg right:90% 80%](https://6850195.fs1.hubspotusercontent-na1.net/hubfs/6850195/Imported_Blog_Media/turning-the-database-inside-out.svg)

---

## Why should I care

<!-- - No relational schema: No service data boundaries/ownership.  reduce service interdependancy for read operations. no contracts no circuts no n+1s. everyone is free to read from the event stream  Maybe this point should be discussed later because at the current time they won't be able to imagine it.-->

<!-- as we mentioned. Nothing is new under the sun all domains were once paper based and based on facts written on sheets paper. This is closer to that which allows you to do things more naturally. Your tech team and your operational team will have the same language with minimal drift. -->

- Naturally aligns itself with the domain.
- Requirement evolution is a breeze (:shushing_face: no db migrations)
- No drift between log output and code.
- Pure,replayable,reliable,safe,_accident_ proof.

And performant

---

## Our recent project.

300k events/day. With a single small pod

---

## The basic idea:

<style>
blockquote {
    border-top: 0.1em dashed #555;
    font-size: 60%;
    margin-top: auto;
}
</style>

- Commands are intentions.
- Events are facts.

![alt text](image-1.png)

> [Source: Functional decider by Jeremie Chassaing](https://thinkbeforecoding.com/post/2021/12/17/functional-event-sourcing-decider)

---
### Write model and read model

![alt text](https://6850195.fs1.hubspotusercontent-na1.net/hubfs/6850195/ES-Pillar-14.svg)

---
# (The write side): Evolution function and The decider

- Decider looks at the command and the given system state and decides what the next "fact(s)" are going to be.
- For each "entity/service" you are familiar with "service->repository->database"
- Here we define an "aggregate" Each aggregate has it's evolution function that determines it's state from previous events.
- Right now you can think of an aggregate as an entity.
<!-- An aggregate is a transactional boundry. a place where all events in the aggregate are enoguh to infer it's own state. (reasonably) you can have aggregates reacting to other aggregates via reactors. which issue commands. But fundementally aggregates own their own state.-->

---

### In practice: processing commands into new facts.

```typescript
// streamId is entityPrefix-<randomId>
const { events } = await eventStore.readStream(streamId);
const state = events.reduce(evolve, { status: "empty" });

// Either type Left | Right. Success Or failure
const decideResult = decide(command, state);
if (isLeft(decideResult)) {
  return unwrapEither(deciderResult);
}

const resultEither = await eventStore.appendToStream(
  streamId,
  unwrapEither(decideResult)
);
```

## <!--This is your service". -->
---

#### decide ex:1

```typescript
      match(state,command).with(
        {
          command: { type: "AssignedFrsAddFr" },
          state: { status: "planned" },
        },
        ({ command: { data } }) =>
          makeRight({
            type: "TrafficCounts.Survey.AssignedFRs.FRAdded",
            data,
          }),
      )

```

---

#### decide ex:2

```typescript
      match(state,command).with(
        {
          command: {
            type: "ReserveSegmentModePair",
          },
          state: {
            status: P.union("planned", "inProgress"),
          },
        },
        ({ command: { data }, state }) =>
          match(state.segmentModePairsReservations)
            .with(
              { [data.segmentId + data.modeId]: { available: false } },
              () =>
                makeLeft({ type: "ALREADY_RESERVED" as "ALREADY_RESERVED" }),
            )
            .otherwise(() =>
              makeRight({
                type: "TrafficCounts.Survey.SegmentModePairReservations.PairReserved",
                data,
              }),
            )),
```
<!--In memory data structure FTW-->

---
## Summary Diagram

![width:950](https://miro.medium.com/v2/resize:fit:1400/1*zFkyTOpPU-DKOrtOrNHHuw.png)

> [Source: Alberto Brandolini](https://medium.com/@ziobrando/collaborative-process-modelling-with-eventstorming-17ed363650c0)

## <!-- Somewhere. And not exactly sure where that where is. the distinction should be drawn between aggregate and entity -->

---
<!-- ## Reactors,policy and system state, read models. -->

<!-- - How one would run things that span different "entities"? Say a "order received,payment received,order confirmed?" -->

<!-- --- -->
## Projections

![width:900](https://6850195.fs1.hubspotusercontent-na1.net/hubfs/6850195/ES-Pillar-8.svg)
> [Source: Event Store DB Blog](https://www.eventstore.com/event-sourcing)


---

## Projections 2
They can be in-memory.
Use the correct data structure/database for *your* problem.

---

# No 'Database' per service. Single source of truth

No relational schema means no need for service data boundaries/ownership. Reduce service interdependence for read operations. 


---

# Eventual consistency
- The read model technically lags behind the write model
- Solution: The API only accepts commands when the client sends the latest event-id within the aggregate’s stream
---

### Other concerns for a production system

- Event versioning and upgrades.
- Easily spin up different views for different needs.
- Live projections (:thinking:)
- Efficiently projecting a large event stream.

<!-- no contracts no circuts no n+1s. everyone is free to read from the event stream  Maybe this point should be discussed later because at the current time they won't be able to imagine it.-->

---
## Lessons learned from our implementation
- Zod
- Typescript
- TONS of Codegen
- Graphql
- Postgraphile.
- ts-rest
- ts-pattern

---

## Small Example Project

- A very simple aggregate
- Demonstrates the usage of said technologies together
- GitHub repo link: https://github.com/hzmmohamed/ddd-es-example
---
## Our System's Architecture 

![width:1100](./arch_diagram.svg)

---

## Our System's Architecture 

- Read and write API separation
- One GraphQL API gateway
- *TypeScript types + code-gen*
A fully type-safe system from the domain model to the GraphQL API client on the front-end

---

## Further Resources
Invaluable resources for learning event sourcing:

- [Oskar Dudycz's Blog](https://event-driven.io/en/)
- [The Functional Decider Pattern](https://thinkbeforecoding.com/post/2021/12/17/functional-event-sourcing-decider)
- [https://www.eventstore.com/event-sourcing](https://www.eventstore.com/event-sourcing)
- [Greg Young's Book on Event Versioning](https://leanpub.com/esversioning/read)
- Greg Young's talks, generally

---

# Thank you for listening.
TfC is hiring!

- h.fhami@transportforcairo.com
- emad.moh.hamdy@gmail.com 

You can find today's talk slides on GitHub: https://github.com/Mohamedemad4/ddd-es-presentation-slides

