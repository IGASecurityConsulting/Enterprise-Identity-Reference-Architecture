# Enterprise Identity Reference Architecture

## Executive Summary

Most identity architecture fails the same way. It gets designed around whatever product was purchased last, not around a coherent set of principles that hold regardless of vendor. The result is federation that works, privileged access that works, secrets management that works, each built by a different team, at a different time, with no shared logic tying them together. When something breaks, or when an examiner asks a hard question, there is no single answer. There are six partial answers from six disconnected systems.

This document defines a reference architecture built the other way. It starts from a small set of principles that apply whether the workload is a human logging into Salesforce, a Kubernetes pod calling an internal API, or an AI agent processing a customer request. Every layer of the architecture, workforce identity, non-human identity, privileged access, secrets, and AI governance, is built on the same underlying logic: verify continuously, enforce deterministically, and produce evidence automatically.

This is not a product recommendation. It does not assume CyberArk, or Okta, or AWS specifically, though the reference implementation notes at the end of this document happen to use that stack. The principles here are meant to outlive any specific vendor decision, the same way a building's structural engineering outlives the specific brand of steel used to frame it.

**Who This Is For**

This document is written for the audience that has to make the actual tradeoff decisions: whether to build continuous verification or accept point-in-time authentication, whether to enforce policy deterministically or lean on AI judgment, whether non-human identity gets governed as a first-class citizen or an afterthought bolted on after the sprawl is already a problem. Those are architecture decisions, not implementation details, and they hold regardless of company size, though the scale of consequence changes dramatically between a 50-person startup and a $340 billion financial institution.

---

## Design Principles

Every layer of this architecture traces back to five principles. None of them are new. All of them are still, repeatedly, not followed in practice, which is why identity remains one of the most common root causes of breaches, examination findings, and operational incidents across the industry.

**1. Verify continuously, not once.**
Identity is not a fact established at login and trusted for the rest of a session. It is a claim that has to be re-evaluated as context changes: new location, new behavior pattern, new device, new resource being accessed. A system that verifies once and trusts indefinitely cannot distinguish between the identity it approved and an identity that has since been compromised, manipulated, or has drifted outside its intended purpose.

**2. Enforce deterministically, reason probabilistically.**
AI and machine learning have a real role in identity architecture, explaining policy, reasoning through ambiguous cases, surfacing patterns a human would miss. They do not have a role in making the actual enforcement decision. A compliance determination has to trace to a specific, codified rule that either held or did not. A probabilistic judgment, however well-reasoned, can always be second-guessed, and second-guessable evidence is not evidence an examiner, auditor, or incident responder can rely on.

**3. Least privilege is a default, not an exception process.**
Every identity, human or machine, gets the minimum access its function requires, scoped to the minimum duration necessary. Standing privileged access is the exception requiring active justification, not the baseline condition everyone starts from and occasionally gets restricted.

**4. Non-human identity is governed with the same rigor as human identity, not less.**
Machine identities already outnumber human identities in most enterprise environments, often by a wide margin, and that ratio is growing faster than governance models built around annual human access reviews can keep pace with. A service account, an API key, or an AI agent is not a lesser identity that gets a lighter governance model. It requires the same ownership, lifecycle, and expiration discipline a privileged human account would.

**5. Evidence is generated continuously, not assembled under deadline pressure.**
Every enforcement decision, every access grant, every credential rotation, produces a record automatically, as a byproduct of the system operating normally. An architecture that requires a manual reconciliation effort to answer "what has access to what, and why" has already failed the moment that question is asked under time pressure, whether from a regulator, an auditor, or an incident response team working against a notification deadline.

These five principles are the spine the rest of this document builds on. Every layer that follows, workforce identity, non-human identity, privileged access, secrets, and AI governance, is an application of these same five ideas to a different category of identity, not five separate philosophies stitched together.

---

## Reference Architecture Overview

**The Shape of the Architecture**

Every access decision in this architecture, regardless of which layer it originates from, workforce identity, non-human identity, privileged access, or an AI agent acting on someone's behalf, flows through the same four functional components. This is deliberate. A single, consistent decision-making shape is what allows five very different categories of identity to be governed by one coherent architecture instead of five disconnected ones.

**Policy Decision Point (PDP).** The component that actually decides whether an access request is allowed. It evaluates identity, requested resource, and current context, device health, location, behavioral pattern, time of day, together, not identity alone. No request is approved on the basis of "who you are." It is approved on the basis of who you are, what you are asking for, and whether the current context is consistent with legitimate use.

**Policy Enforcement Point (PEP).** The component that actually carries out the PDP's decision. This is where access is granted or blocked, where a privileged session is proxied and recorded, where an API gateway validates a token before letting a request through. The PDP decides. The PEP is the only thing capable of acting on that decision, which means an attacker who compromises a PDP's reasoning but not the PEP's enforcement still cannot bypass the control.

**Policy Information Point (PIP).** The component that supplies the context the PDP needs to make a good decision: current risk signals, device posture, HR system status for a human identity, ownership and lifecycle status for a non-human identity. A PDP is only as good as the information it is given. Stale or incomplete information here is a silent failure mode; the PDP will make a confident, wrong decision on outdated data without any indication anything is wrong.

**Policy Administration Point (PAP).** The component where policy itself is authored, reviewed, and versioned, by humans, with change control, not edited ad hoc in a console by whoever needs a quick exception. This is where the Governance Operating Model attaches to the technical architecture: someone owns the PAP, and every rule it contains traces to a decision someone made and can explain.

**How the Five Identity Categories Flow Through This Shape**

Workforce identity, a human logging into an application, hits the PDP through an identity provider, gets evaluated against conditional access policy sourced from the PIP, device compliance, location, HR status, and is enforced at an SSO gateway acting as the PEP.

Non-human identity, a service account, a Kubernetes workload, an API key, follows the identical path, but the PIP supplies different context: ownership status, credential age, scope of originally approved permissions, not device posture or HR status. The PDP and PEP components do not change. Only the information feeding them changes.

Privileged access, a human or automated process requesting elevated access to critical infrastructure, routes through a PAM proxy acting as the PEP, with the PDP evaluating business justification, time-boundedness, and separation-of-duties constraints sourced from the PIP.

Secrets, the credentials, keys, and certificates workloads need to authenticate to each other, are themselves gated by this same shape: a workload requesting a secret is itself an access request, evaluated by the PDP, enforced by a secrets management PEP, informed by the PIP's knowledge of which workload is legitimately entitled to which secret.

AI agents acting autonomously are the newest and least mature category, and the one most tempting to treat as an exception to this shape. They are not an exception. An agent's every action, every tool call, every attempt to access a resource, is itself a request that flows through the same PDP/PEP structure as everything else. The difference is that agents can chain actions and, in poorly designed systems, acquire new permissions dynamically at runtime, which makes the PEP's role of enforcing a fixed, pre-approved scope more important here than anywhere else in the architecture, not less.

**Why One Shape, Not Five**

A separate governance model for each identity category is how organizations end up with the fragmented, six-disconnected-answers problem described in the Executive Summary. One consistent decision-making shape, applied to five different identity categories, is what makes it possible for a single evidence layer to answer "what has access to what, and why" across the entire environment, rather than requiring five separate systems to be manually reconciled every time that question is asked.

---

## Layer Breakdown

### Workforce Identity

**What This Layer Governs**

Every human identity accessing enterprise systems: employees, contractors, and any human user whose access is tied to an employment or engagement relationship. This is the most mature layer of most identity programs, and also the layer where the gap between "mature" and "actually enforced" is most commonly overlooked, because the tooling has existed longest and the assumption that it is handled runs deepest.

**Core Components**

Identity provider and federation. A single source of truth for workforce identity, federated via SAML or OIDC to every downstream application, rather than each application maintaining its own user directory. Federation is not primarily a convenience feature. It is what makes deprovisioning actually work: disable one identity at the source, and access to every federated application ends immediately, instead of requiring someone to remember and manually revoke access across a dozen separate systems.

Conditional access. Authentication is necessary but not sufficient. Every access decision evaluates device compliance, location, and behavioral risk signals as part of the same decision, consistent with the continuous verification principle. A correct password from a compromised or non-compliant device is not the same claim of legitimate access as a correct password from a known, compliant device in an expected location.

Lifecycle automation, Joiner, Mover, Leaver. Access is provisioned, adjusted, and revoked automatically as HR system status changes, not through a ticket someone has to remember to file. A role change should trigger automatic reevaluation of access, not leave prior access silently in place alongside newly granted access, which is how privilege silently accumulates over a career at one organization.

**Where This Layer Commonly Fails**

The most common failure is not weak authentication. It is stale authorization: access granted correctly at hire, never revisited as the person's role changed, still active years later because deprovisioning was tied to termination but never to role change. A workforce identity layer that only acts at hire and termination is only handling two of the three letters in Joiner, Mover, Leaver.

**Design Decision**

Federation and conditional access are treated as inseparable. An identity provider without conditional access policy attached is authentication without authorization, which satisfies "who are you" but never actually answers "should this access be allowed right now, in this context."

### Non-Human Identity

**What This Layer Governs**

Service accounts, API keys, workload identities, certificates, and autonomous AI agents. In most enterprise environments today, this population already outnumbers workforce identity many times over, and it is growing faster than any other identity category, driven directly by cloud adoption and AI agent deployment.

**Core Components**

Continuous discovery. Every non-human identity is discovered and inventoried the moment it is created, across every platform, not reconciled periodically through a manual audit. A quarterly review of a population that changes daily is not a control. It is a sample that is stale before the review is even finished.

Ownership and lifecycle automation. Every non-human identity is assigned an accountable human owner at creation, with credentials that expire and rotate on a defined schedule by default. Static, indefinitely-valid credentials are the tracked exception requiring explicit justification, not the baseline condition.

Scoped, least-privilege execution. A non-human identity, including an AI agent, operates under a defined scope established at deployment. Any action outside that scope requires validation independent of the identity's own request or stated intent, since a compromised or manipulated non-human identity's own claim about what it needs is not a trustworthy signal by itself.

**Where This Layer Commonly Fails**

Governance models built for workforce identity get applied unchanged to non-human identity, and they do not transfer. Annual access reviews, appropriate for a human whose role changes slowly, are meaningless applied to a service account population that can grow by thousands in a single sprint. This layer requires automation as the default operating mode from the start, not automation retrofitted after a manual model has already failed at scale.

**Design Decision**

AI agents are governed as a distinct sub-category within non-human identity, not folded indistinguishably into general service account policy. An agent's capacity to chain actions and, in poorly designed systems, acquire permissions dynamically at runtime is a meaningfully different risk profile than a static API key, and the architecture accounts for that difference explicitly rather than assuming existing NHI controls transfer without modification.

### Privileged Access Management

**What This Layer Governs**

Any access capable of materially affecting critical systems: infrastructure administration, security tooling configuration, database administration, and any access whose misuse, accidental or deliberate, could cause outsized damage relative to a standard user account. This layer governs both human privileged users and automated processes, CI/CD pipelines, orchestration tools, that carry equivalent capability.

**Core Components**

Zero standing privilege. Privileged access exists only for the duration of an approved, time-bound task. A privileged credential that is not actively being used at this moment is a credential that should not exist yet, since every standing privileged credential is a standing target regardless of whether it is currently being misused.

Session isolation and recording. Every privileged session is proxied through a system that isolates the session from direct credential exposure and records the session for later review. This is not primarily about catching misuse after the fact, though it does that too. It is about ensuring the credential itself is never directly exposed to the person or process using it, which removes an entire category of credential theft.

Structural separation of duties. No single identity can both request and approve its own privileged access, and no identity holding privileged access can also modify the logs recording that access. This holds regardless of the individual's intent, because a control that depends on trusting intent is not a control, it is a hope.

Just-in-time elevation. Access is elevated to privileged status only when a specific, approved task requires it, and automatically de-elevated when the task window closes. The default state for every identity, including identities whose job function regularly requires privileged access, is standard, unprivileged access.

**Where This Layer Commonly Fails**

The most common failure is not the absence of a PAM tool. It is scope: a PAM platform deployed for the most obviously critical systems, with a long tail of infrastructure, cloud consoles, database admin panels, CI/CD pipeline credentials, left outside its coverage because bringing every system in felt like a phase two that never arrived. An attacker does not need the single most critical system in the environment. They need any privileged path the PAM program forgot to cover.

**Design Decision**

Automated processes, CI/CD pipelines, orchestration tools, deployment automation, that carry privileged capability are governed under this same layer, not treated as a separate, lighter-weight category because they are not human. A deployment pipeline with the ability to modify production infrastructure carries the same risk profile as a human administrator with that same ability, and gets the same zero-standing-privilege, just-in-time treatment.

### Secrets Management

**What This Layer Governs**

The credentials, API keys, certificates, and tokens that workloads and non-human identities use to authenticate to each other. This layer is closely related to non-human identity governance but distinct from it: non-human identity governs the identity itself, secrets management governs the specific credential material that identity uses to prove who it is.

**Core Components**

Centralized, dynamic secret issuance. Secrets are issued dynamically at the point a workload needs them, from a central secrets management system, rather than generated once and embedded in configuration files, environment variables, or source code where they persist indefinitely and are easily overlooked during rotation efforts.

Short-lived credentials by default. Every issued secret has a defined, short lifetime. A workload that needs ongoing access requests a new credential through the same governed path rather than being issued something that works indefinitely. This is the secrets-management-specific application of the same standing-credential principle applied to non-human identity generally.

No secrets in code or configuration. Static credentials embedded directly in source code, configuration files, or CI/CD pipeline definitions are treated as an incident, not a style preference to be addressed eventually. A secret in source control is a secret that is now part of that repository's permanent history, discoverable by anyone with repository access, indefinitely, regardless of whether it is later rotated.

**Where This Layer Commonly Fails**

Secrets sprawl happens the same way non-human identity sprawl happens: one reasonable decision at a time. A developer hardcodes a credential to unblock a deadline, intending to fix it later. It works, nobody revisits it, and eighteen months later it is one of thousands of embedded credentials nobody can account for. The fix is architectural, making the dynamic, short-lived path easier to use than the hardcoded shortcut, not purely a policy reminder that hardcoding is discouraged.

**Design Decision**

Secrets management is treated as the enforcement mechanism for non-human identity's credential lifecycle principle, not a separate, parallel system. The non-human identity layer decides that a credential should expire on a defined schedule. The secrets management layer is what actually makes that expiration real, technically, rather than a policy statement with no mechanism behind it.

### AI Governance and Policy Enforcement

**What This Layer Governs**

Every point where an AI system, whether a reasoning layer answering questions, an autonomous agent taking action, or a model embedded in a larger workflow, makes or influences a decision with real consequence. This layer is not AI security bolted onto the architecture as a separate concern. It is the specific application of every principle already established, continuous verification, deterministic enforcement, least privilege, to the newest and least mature category of actor in the environment.

**Core Components**

Deterministic enforcement beneath probabilistic reasoning. Every AI system in this architecture is built as two layers, never one. A probabilistic reasoning layer, an LLM, explains, reasons through ambiguous cases, and produces natural-language output. A deterministic policy engine beneath it makes the actual enforcement decision and produces the evidence. The reasoning layer is never the thing standing between a request and an outcome with real consequence.

Scoped execution for agents. Every AI agent operates under a defined, pre-approved execution scope. An agent that only needs to read data and produce a summary is architecturally incapable of writing to a system it was never scoped to reach, regardless of what instructions it receives at runtime, including instructions crafted specifically to convince it otherwise.

Untrusted input handling. Input originating from outside the system, a customer submission, a scraped document, content from an external source, is treated and segregated as untrusted, never blended indistinguishably with trusted system instructions. This is the direct architectural defense against prompt injection: an attacker who can craft input cannot use that input to issue instructions the system treats as authoritative.

Human validation on consequential actions. Any AI-initiated action with real consequence, modifying data, executing a transaction, changing access, requires validation independent of the AI's own output before it executes. This is not a lack of trust in the technology. It is the recognition that an AI system's own judgment about whether its proposed action is safe is not, itself, a control an adversary cannot influence.

**Where This Layer Commonly Fails**

The most common failure is treating the AI system's own stated reasoning as sufficient evidence that a decision was sound. A well-written explanation from a language model is genuinely useful for a human trying to understand a situation. It is not, by itself, proof that a control was enforced, because a probabilistic explanation can always be second-guessed, and evidence that can be second-guessed does not hold up under regulatory or incident-response scrutiny.

**Design Decision**

This layer does not get its own separate governance model. It is built from the same PDP, PEP, PIP, PAP structure as every other layer in this architecture, with the specific addition that the PDP's reasoning component, the LLM, is explicitly and permanently separated from the PEP's enforcement component, the deterministic policy engine. AI governance, in this architecture, is not a new category of control. It is the existing architecture applied to a new category of actor.

---

## Key Architecture Decisions

**ADR-001: Deterministic Policy Enforcement Over Probabilistic AI Judgment**

Status: Accepted

Context. Every layer of this architecture eventually needs to answer a specific, checkable question: was this access compliant, was this credential current, was this action within scope. AI reasoning is valuable for explaining these questions to a human. It is not, on its own, capable of serving as evidence that a control held, because a probabilistic answer can always be second-guessed.

Decision. Every enforcement decision in this architecture traces to a deterministic rule, evaluated by a policy engine, not to an AI system's judgment. AI reasoning layers are positioned above the enforcement layer, for explanation and escalation handling, never as the enforcement mechanism itself.

Consequence. This requires policy logic to be explicitly codified in advance, which is real upfront work and cannot flexibly cover every conceivable edge case the way a reasoning model can. Edge cases outside codified rules route to a defined human escalation path rather than an automatic AI determination. This tradeoff is accepted because the alternative, treating AI judgment as sufficient evidence of enforcement, does not survive direct examiner or auditor scrutiny.

**ADR-002: Continuous Verification Over Point-in-Time Authentication**

Status: Accepted

Context. Identity risk does not surface at the moment of login. It surfaces later, in behavior that does not match the identity a system already approved: an account accessing resources inconsistent with its normal pattern, a credential used from an unexpected context, an agent acting outside its established scope.

Decision. Every identity in this architecture, human or non-human, is evaluated continuously across its lifecycle, not verified once at authentication and trusted indefinitely afterward.

Consequence. This requires instrumentation across the full lifecycle of every identity category, not a single checkpoint, which is a larger design surface than point-in-time authentication alone. This is accepted because point-in-time authentication cannot, by definition, detect risk that only manifests after the initial check has already passed.

**ADR-003: Non-Human Identity Governed as a First-Class Category, Not an Extension of Human Identity Governance**

Status: Accepted

Context. Non-human identity now outnumbers human identity in most enterprise environments, often by a wide margin, and grows at a materially different rate. Governance models built around human review cycles, annual access certification, manual periodic sampling, do not transfer to a population that can grow by thousands within a single sprint.

Decision. Non-human identity is governed through its own dedicated discovery, ownership, and lifecycle automation, built for continuous, automated operation from the start, not adapted from a human-identity governance model after that model has already failed to scale.

Consequence. This requires separate tooling and processes from workforce identity governance, rather than a single unified system handling both. This is accepted because the alternative, applying human-paced review cycles to a non-human population, has already been observed at scale to produce exactly the ungoverned sprawl this architecture is designed to prevent.

**ADR-004: Zero Standing Privilege Over Persistent Administrative Access**

Status: Accepted

Context. A privileged credential that exists continuously, whether or not it is actively being used, is a standing target regardless of the intent of whoever holds it. Persistent administrative access is the traditional default in most environments specifically because it is operationally convenient, not because it is the more defensible design.

Decision. Privileged access, for both human and automated identities, exists only for the duration of an approved, time-bound task, and is enforced through just-in-time elevation rather than standing grants.

Consequence. This introduces real friction into workflows that previously relied on persistent access, and requires a functioning just-in-time elevation system to be in place before this can be enforced without disrupting operations. This is accepted because persistent privileged access is one of the most consistently exploited conditions in real-world incidents, and the operational convenience it offers does not offset that exposure.

---

## Governance Operating Model

**Why This Section Exists**

An architecture document that stops at technology is half a document. The other half is who actually runs this thing once it's built. Most identity programs don't fail because the technology was wrong. They fail because nobody owned the parts that don't show up on an architecture diagram: who approves an exception, who reviews the policy engine's rules once a year, who gets a phone call at 2am when something breaks. Skip this section and you've built a system nobody's actually accountable for.

**Ownership**

Every layer in this architecture has exactly one accountable owner. Not a team. Not "security." One named function.

Workforce identity is owned by the identity and access function, full stop. Non-human identity is owned jointly by security architecture and whichever business unit is generating the identities, because security architecture can build the automation, but only the business unit actually knows why a given service account exists and whether it's still needed. Privileged access is owned by security operations. Secrets management is owned by platform engineering, working under policy set by security architecture. AI governance is owned jointly by security and whatever function is deploying the AI system, because neither one alone has the full picture.

If you can't name who owns a layer, you don't have governance. You have a diagram.

**Review Cadence**

Two clocks run here, and they're not the same clock.

The first is continuous and automated, the system checking itself. Is discovery still finding new identities. Are credentials still rotating on schedule. Is the policy engine still firing correctly. This isn't a meeting. It's alerts that fire the moment something stops working the way it's supposed to.

The second is annual, and it's a human conversation, not automation. Once a year, whoever owns each layer sits down and asks a harder question: is this architecture still right, or has the business changed underneath it. New AI adoption might mean the non-human identity layer needs new rules it didn't have last year. A new regulation might mean the evidence layer needs to capture something it currently doesn't. This is the review that produces a new ADR, not a status update.

**Exception Handling**

Every rule in this architecture will eventually run into a real business situation where the strict version causes a real problem. That's not a flaw in the design. That's reality.

What matters is how the exception gets handled. Documented, in writing, by the person who actually owns that layer, not improvised by whoever's under deadline pressure that week. Time-boxed, with a real expiration date, not until further notice, because until further notice is how a temporary exception becomes permanent institutional debt nobody remembers approving. When the expiration hits, the exception either gets renewed on purpose or it dies. It doesn't just keep existing because nobody got around to cleaning it up.

**Metrics**

You can't manage what you can't measure, and you can't prove a control works with a feeling. Three numbers carry real weight here:

Percentage of non-human identities with a current, active owner. This tells you whether the ownership model is real or just written down somewhere.

Percentage of privileged access operating as just-in-time versus standing. This tells you whether zero standing privilege is an actual practice or a slide in a deck.

Time to produce a complete access answer for any given identity, measured against whatever regulatory clock actually applies to your business. This is the number that tells you whether you survive an audit or an incident, or whether you're the team frantically pulling spreadsheets together after the deadline's already passed.

**Escalation**

A gap found by the automated review goes to the layer owner first. If it touches regulatory exposure, it goes to compliance and executive leadership immediately, not at the next scheduled meeting. The whole point of catching something early is that it stays a manageable problem instead of becoming the thing an examiner finds first.

---

## Roadmap

**Why Sequence Matters More Than Speed**

Nobody adopts this whole architecture on day one. Trying to would be a mistake, not ambition. Each phase below builds on the phase before it, and skipping ahead just means you're building on ground that isn't there yet.

**Phase 1: Discovery and Inventory**

Before you can govern anything, you have to know what exists. This phase stands up continuous discovery across every identity category, workforce, non-human, privileged, secrets, in whatever order reflects your actual exposure. For most organizations, non-human identity discovery is the phase that produces the biggest, most uncomfortable surprise, because that's the population nobody's been counting.

You're not fixing anything yet. You're building the map. Skip this phase and every later phase is guessing.

**Phase 2: Lifecycle Automation and Ownership**

Now you assign owners and automate the boring, repetitive work: credential rotation, access expiration, deprovisioning tied to actual status changes instead of someone remembering to file a ticket. This is the phase that eliminates the standing, indefinitely-valid access that represents most of your actual risk.

This phase touches real workflows people rely on every day. Roll it out in waves, lowest-risk systems first, and don't touch anything mission-critical until the automation's been proven somewhere it can't do real damage if it's wrong.

**Phase 3: Deterministic Policy Enforcement**

This is where the policy engine goes in, the layer that turns "we have a policy" into "we can prove the policy holds." Every access decision now traces to a codified rule, not a document nobody's checked in two years.

This phase depends entirely on Phase 1 and 2 being real. A policy engine evaluating an inventory that's incomplete or a lifecycle process that's still manual isn't enforcing anything. It's decorating a gap.

**Phase 4: Continuous Evidence and AI Governance**

Last phase, evidence generation goes live across every layer, and AI governance gets the same deterministic-enforcement treatment as everything else. This is the phase that answers the question that actually matters when someone official asks it: what has access to what, why, and how do you know the control held.

**What Doesn't Wait**

Every phase produces something usable the moment it ships. You don't need to wait until Phase 4 to have something worth showing a board or a regulator. Phase 1 alone, a complete inventory where there wasn't one, is already a materially stronger position than most organizations are in today.

---

## Reference Implementation

Everything above is architecture, principles and tradeoffs that hold regardless of specific tooling. Select components of this architecture have a working reference implementation, built to validate that the design holds up against real infrastructure, not just on paper.

That implementation is available at github.com/IAM-AI-Security, including a deterministic policy engine paired with RAG-grounded reasoning for identity governance, automated non-human identity lifecycle management, and continuous privileged access drift detection.
