# Church Platform Architecture Document

**Status:** Draft
**Audience:** Engineering team
**Date:** 2026-09-10

---

## 1. Introduction

### Context and Problem Statement

The church currently manages its content, member records, and messages using a mix of manual steps and separate tools. Member lists are kept in spreadsheets, announcements are sent as one-off messages, and attendance is counted by hand. This causes some problems that keep happening:

- **Old, duplicate data.** Member and team information is kept in many places: spreadsheets, paper sign-up sheets, someone's phone contacts. There is no **source of truth** (the one place everyone treats as correct, so there is no question about which copy is right). A change in one place does not show up anywhere else.
- **Attendance tracked by hand, often wrong.** Headcounts and sign-ins are done by hand. This is slow at the door and hard to trust for reports, like who comes regularly and who has stopped coming.
- **Messages sent in a scattered way.** There is no steady way to reach one team, a group of members, or the whole church, on the channels people actually use, like text, WhatsApp, or email.
- **Data that cannot move.** Data stays stuck in whatever tool it was entered into, so it cannot be reused. You cannot build attendance reports from a spreadsheet a leader keeps on their own computer, and you cannot send a message on its own when a paper sign-up sheet changes.

This document proposes a small set of services built for these jobs. They replace the manual steps with a system that has one place for content, one place for people and attendance, and one steady way to reach people. The system also has to stay simple enough for one volunteer engineer to build and run.

One thing shapes every choice in this document: this is built and run by one person, using Docker containers rather than a large managed platform. Each extra service is extra work: one more thing to set up, watch, and fix, maybe at 11pm on a Saturday before Sunday service. So this design aims for the fewest moving parts that still keep the system easy to understand.

---

## 2. Logical Diagram

*(Diagram to be added separately.)*

### Architectural Principles and Trade-offs

Three ideas guided how the system was split into services. Each one has a name used across the software industry, so it's given here in full, with a plain definition attached. Knowing these names lets you describe a design decision in one or two words instead of a paragraph, and lets you follow along when other engineers use them.

**Bounded Context.** *A bounded context is an area of a system where one set of words and rules applies consistently. Outside of that area, the same word can mean something different, or not apply at all.* Split services by what they mean, not by technical layers. "Content" (sermons, articles, the service program) and "people" (members, teams, attendance, messages) are two different bounded contexts. They use different words, have different owners, and work in different ways. That is a real line between them, so it stays a real line between services.

**Cohesion, and the Common Closure Principle.** *Cohesion is how closely related the things inside one part of a system are. High cohesion means the things that belong together are grouped together. The Common Closure Principle is the rule that follows from this: group together the things that tend to change for the same reason.* Two parts of the system belong together when one real event tends to cause changes in both. A new member joining, a team list changing, or someone being marked absent all touch both "who is involved" (engagement) and "how we reach them" (notifications). They react to the same kind of event, so grouping them gives high cohesion. Content is different. It changes on its own writing and publishing schedule and is not caused by those same events, so it has low cohesion with the other two and is best kept on its own.

**Coupling.** *Coupling is how much one part of a system depends on another, and how much a change in one forces a change, or extra work, in the other. Tight coupling is when two parts depend on each other so heavily that they are hard to change or run independently. Loose coupling is the opposite, and is usually what you want between separate services.* If one service has to ask another one for something on almost every request, that is tight coupling, a sign the two are not really independent. They are one system split in two for no real benefit. Sending a message almost always needs to know who to send it to: a team, a group of people who missed church, or everyone who checked in. That lookup needs engagement's data almost every time a message is sent. Splitting these into two services would turn that lookup into an extra network call (a request sent from one service to another over the network, which is slower and less reliable than calling a function inside the same program) on the most common action in the whole system. Keeping them together removes that call and keeps the coupling loose where it matters.

**What this means for the shape of the system:**

| Question | Decision | Why |
|---|---|---|
| Should content be its own service? | Yes | It's a different bounded context, on a different schedule for change, with no shared data with the others. |
| Should engagement and notifications be combined? | Yes | They have high cohesion (they share data and react to the same events). Splitting them would introduce tight coupling, a network call on almost every action, without giving any real benefit at this size. |
| Is the monitoring tool a fourth service to weigh here? | No | It watches whatever services exist. It does not own a bounded context of its own. |
| Should we aim for the most possible separation between services? | No | One engineer runs this system using containers, not a platform that shrinks services down automatically when they are not busy. Every extra service is an ongoing cost. It must be built, set up, and kept running all day and night, even when it is not busy. |

**The trade-off, in plain terms:** more services give better fault isolation (a failure in one service is less likely to spread into another) and let each piece scale on its own. Fewer services mean less to run, lower cost just to keep things going, and fewer places where the same piece of data can get out of sync. Given the team size (one person) and the traffic (one church, not thousands of people at once), keeping things simple with fewer services matters more here than maximizing fault isolation. So the system ends up as two services plus one monitoring layer that watches both, not the fully split version.

**Key terms used in this section:**

| Term | Definition |
|---|---|
| Bounded context | An area of a system where one set of words and rules applies consistently. |
| Cohesion | How closely related the things inside one part of a system are; high cohesion means related things are grouped together. |
| Common Closure Principle | Group together the things that tend to change for the same reason. |
| Coupling | How much one part of a system depends on another. Tight coupling is hard to change independently; loose coupling is easier. |
| Fault isolation | How well a failure in one part of a system is kept from spreading into another part. |
| Source of truth | The one place everyone treats as correct, so there is no question about which copy of a piece of data is right. |
| Module | A self-contained part of code, with its own data and a clear boundary, that can be built and tested somewhat independently even when it deploys as part of a larger service. |
| Observability | The ability to understand what is happening inside a system by looking at its logs, metrics, and traces, instead of guessing. |

### How Services Communicate

Content has its own database. Engagement and Notifications share a single database between them, since they are one service, as decided in the table above.

Cross-service communication happens as a **live API call at request time**: when one service needs something from another, it calls that service's API right when it needs it and waits for the answer, rather than sending an event to be picked up later.

**Worked example: notifying people when a program is created.** When Content publishes a Sunday service program, people often need to be told, for example the team on duty. The trigger for this, the fact that a program was just published, belongs to Content, since Content is the only service that knows when that happens. But working out who to notify, and actually sending the message, needs the audience and roster data, which lives entirely in Engagement & Notifications. So the flow is: Content publishes the program, then makes a live API call to Engagement & Notifications (something like "a program was published, here is its ID and date"), and Engagement & Notifications resolves the right audience from its own data and sends the message. The sending logic and the "who to notify" decision stay in Engagement & Notifications, not Content. Content's only job is to make the call at the right moment.

One thing worth deciding explicitly before this is built: because this is a live call, not a queued event, Content's publish action depends on Engagement & Notifications being reachable at that moment. If that call fails, does the program still get published and the notification just get skipped, or does the whole publish fail. That's a small decision, but worth writing down.

### Access and Identity

Access and identity checks are enabled on both services, not just one. Each service checks who is making a request and what they are allowed to do, based on a role (for example, admin, leader, or member). A request carries proof of who is making it, and both Content and Engagement & Notifications check that proof independently, rather than one service trusting the other blindly.

### Handling Third-Party Failures

Engagement & Notifications depends on outside providers to actually deliver a text, email, or WhatsApp message. Those providers can be slow or fail. To improve resilience (a system's ability to keep working, or recover quickly, when something outside it goes wrong), failed sends are retried automatically rather than being dropped. A message that keeps failing after retries is marked failed and shows up in the Control Centre, rather than disappearing silently.

### Data Migration

This document does not cover moving old data (existing spreadsheets, paper sign-up sheets, contact lists) into the new system. The services start recording new data from the point they go live, rather than importing history. If that turns out to be needed later, it is a separate, smaller piece of work.

---

## 3. Service Descriptions

### 3.1 Content Service

*This is the existing content system the church already runs, renamed here to Content Service for consistency with the rest of this document.*

**What it does:** Manages everything the church publishes. This includes sermons, articles, event and activity listings, and the photos, audio, or video that go with them. It also lets a leader plan and publish the order of service for a given Sunday.

**Core capabilities:**

- Upload and organize sermon recordings, articles, and event or activity write-ups, along with their photos, audio, or video.
- A draft, then review, then publish step, so content is checked before anyone outside the team can see it.
- Plan and publish the Sunday service program (order of service, who is leading which part).
- Keep a history of edits, so it is always possible to see what changed and when.

**What it does not do:** It does not hold member records or attendance, and it does not send messages. If the service program needs to show who is on duty that week, it reads that from the engagement service instead of keeping its own copy of who is on which team.

**Why it stands alone:** It follows its own writing and review process, on its own schedule, and it does not share data with the other two services.

**Data and communication:** Content has its own database, separate from Engagement & Notifications. When it needs data it does not own, such as who is on duty for a service program, it makes a live API call to Engagement & Notifications rather than keeping its own copy. When a program is published, it makes a live API call to Engagement & Notifications to trigger the notification (see "How Services Communicate" in Section 2).

### 3.2 Engagement & Notifications Service

**What it does:** Holds the record of everyone who is part of the church. This includes members, households, teams, and the events they take part in. It tracks who showed up, and it sends messages to the right people, on the channel they prefer.

**Core capabilities:**

*Engagement (people and participation):*

- Member and household records: who is who, their contact details, photo, and whether they agreed to be contacted.
- Teams and ministries: who leads them, who is on them, and how that has changed over time.
- Events with repeat schedules (weekly services, regular small groups).
- Attendance tracking: people can check themselves in, a leader can mark people present, or a list can be uploaded after the fact.
- Reports that answer questions like "who has stopped attending" or "how is this team growing."

*Notifications (reaching people):*

- Sending a message to a team, to a group (for example, everyone marked absent recently), or to the whole church.
- Sending across text, WhatsApp, email, and app notifications, so people are reached on the channel they actually check.
- A record of what was sent, to whom, and whether it was delivered.
- Automatic messages, for example a message that goes out on its own when someone has been marked absent for several weeks in a row.

**Why it is one service, not two:** almost every message needs engagement's data to work out who to send it to. Keeping them together means that lookup is a simple read from the same data, instead of a call to another service. It also means one event, like someone joining a team, updates one place, instead of needing to be kept in step across two.

**Kept apart inside, even though it is one service:** the two halves should still be built as clearly separate **modules** (a module is a self-contained part of the code, with its own data and a clear boundary, that can be built and tested somewhat independently even though it deploys as part of the same service). This way, if the church grows enough to need notifications as its own service later, that change stays small and controlled instead of needing a rewrite.

**Data and communication:** Engagement and Notifications share a single database, split internally into separate tables for people and participation versus messages and delivery records. Other services, such as Content, reach this service through a live API. Sends to outside providers (text, email, WhatsApp) are retried automatically on failure to improve resilience; a send that keeps failing is marked failed rather than silently dropped, and shows up in the Control Centre.

### 3.3 Control Centre

**What it does:** Gives the engineering team **observability** (the ability to understand what is happening inside a system by looking at its logs, metrics, and traces, instead of guessing) into everything else. Is content publishing working. Is check-in responding. Are messages actually being delivered.

**Core capabilities:**

- Live view of each service's health (up or down, error rates, response times).
- Alerts when something needs attention, for example a pile-up of unsent messages, or a service that has stopped responding.
- A way to search logs and follow what happened when something goes wrong.

**Why it does not count as a third main service:** it does not own any of the real data itself. No members, no content, no messages. It only watches the other services. Think of it as a tool that sits next to the system, not as another service that content or engagement depend on.

---

## 4. Value Statements

### 4.1 Content Service

#### Reliable data

- One place holds the current, correct version of every piece of published content. No more hunting for the latest copy.
- The review step before publishing catches mistakes before members see them.
- A history of edits means it is always possible to see what changed and undo it if needed.

#### Less work day to day

- Reusable templates for sermons, articles, and events cut down repeated setup work.
- Scheduling a publish in advance removes last-minute manual updates.
- Planning the service program in the same place it gets published removes a manual copy step.

#### Room to grow

- Content can be reused across the website, an app, or future channels without rewriting it for each one.
- New content types (podcasts, devotionals, multiple languages) can be added without redesigning the system.

#### Member experience

- Members see consistent, up-to-date information wherever they look for it.
- Multi-language support, when needed, reaches more people.

### 4.2 Engagement & Notifications Service

#### Reliable data

- One record per member and team, replacing scattered spreadsheets and sign-up sheets.
- Attendance is recorded the moment it happens, at check-in or on a leader's list, instead of a manual headcount and the mistakes that come with it.
- Consent is tracked alongside contact details, so messages only go to people who agreed to receive them.

#### Less work day to day

- Members and leaders can update their own information instead of someone re-typing it from a form.
- Check-in at the door is fast and does not need anyone to enter data by hand afterward.
- One message can reach a team, a group, or the whole church, through several channels at once, instead of sending it on each one separately.
- Follow-up messages, for example to people who have missed a few weeks, go out on their own instead of relying on someone remembering to check.

#### Room to grow

- The same system handles a small group check-in and a large gathering without needing to be redesigned.
- New kinds of groups (small groups, other church locations, discipleship stages) can be added without reworking existing data.
- Adding a new way to send messages later is a small, controlled change, not a rebuild.

#### Member experience**

- Checking in is quick and easy.
- Leaders can see who is present or absent in real time and follow up personally.
- Messages arrive on the channel people actually check, rather than being missed.

### 4.3 Control Centre

#### Reliable data

- Gives an accurate, live picture of what is happening across the system, instead of finding out something broke from a member's complaint.

#### Less work day to day

- Problems are caught and flagged on their own, instead of needing someone to notice by hand.
- Logs are all in one place, so it is faster to find and fix a problem when something goes wrong.

#### Room to grow

- As more services or checks are added, they plug into the same view instead of needing their own separate tools.

#### Member experience

- Indirect, but real. Catching and fixing problems faster means less downtime and fewer failed messages that members would otherwise notice.
