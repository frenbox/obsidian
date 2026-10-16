---

title: Argus × BOOM Meeting Notes tags:

- meeting
- argus
- boom attendees:
- Hank
- Niall
- Antoine
- Will
- Jon
- Donovan
- Katie
- Thomas
- Michael

---

# Argus × BOOM Meeting Notes

> [!todo] Easy wins to focus on
> 
> - [ ] Stand up a test instance for the prototype — on Aurora?

> [!question] Open questions
> 
> - Cadence priorities?
> - Limiting science cases?

---

## Day 1

### Simulation

**Attendees:** Hank, Niall

- 3000 testing packets available; one week of testing data would be ideal — a set of Avro files with some linked IDs between them and some new photometry
- Simulation harness? (BOOM can mock Kafka producers/channels for testing)
- `MESA_Argus` to generate _N_ Avro files at 1 s, 1 min, and 15 min cadences
    - 1 week, 100 million per night, 50k sources
    - Source ID and detection ID
    - Perturbation agnostic

### SkyPortal

**Attendees:** Antoine, Will, Jon, Donovan, Katie

_No notes recorded._

---

## Day 2

### Hardware

**Attendees:** Hank, Niall

- Continue to use `highpoint` for testing; no real GPU need right now
- Need more storage if we want to store catalogs there (move to another system)
- What hardware does Argus need to achieve this goal? (Or what goal can Argus achieve with the given hardware?)

### Prototype integration

**Attendees:** Hank, Will, Thomas

- How do we pass observations / follow-up requests back and forth?
- How can we safely navigate the various firewalls in our way?

### Schema

**Attendees:** Donovan, Hank, Jon, Niall, Katie

#### Agenda

- How will schemas be changed?
- Idea of different schemas for SSO and other alerts
- Enrichment: to what extent does Argus compute (on the mountain) vs. send to BOOM?
    - Obvious splits:
        - Argus should do real/bogus
        - BOOM should do Pan-STARRS associations
    - TBD: SSOs, photometric histories, cutouts, time

#### Discussion

- Different schemas for SSOs and other alerts?

##### Schema changes

- Schema and changelog will be published to a schema registry
    - Semantic versioning: `major.minor.fix`
- We will be notified of schema changes and coordinate between us
- Upon a new breaking schema version (whether major or minor):
    - Let the BOOM team know a week in advance
    - Make the change during the day:
        - Update the schema on HDPS (and the public repo before HDPS is public)
        - Publish to a new Kafka topic

##### Enrichment

Most things will be best done on the mountain or best done by BOOM. The split is as follows:

- **Cutouts**
    - Should Argus send regular minipixes or centered cutouts?
        - If minipix: will need surrounding minipixes for photometry and context
        - Centering may not be feasible at 1 min cadence
    - What size should the science / difference cutouts be?
        - Power of 2, for GPU
        - 64×64, 32×32?
    - Can we get away with not sending references?
        - Yes, if BOOM hosts the templates
        - For now, ship all three until references are better settled
    - At what cadences should we be sending cutouts?
- **Photometric histories**
    - If there is one broker, not eight, can we not ship histories?
        - Since the stream off the mountain is private and BOOM will curate histories anyway, then yes
        - Keeps the system stateless, makes packets slimmer, etc.
        - Requires some more safeguards and backups to prevent data loss
- **IDs** — the current schema is wrong. The new schema needs:
    - Alert ID: UUID
    - Object ID: HEALPix-derived ID
    - Human-readable: 4-word hash (nice-to-have)
- **Time**
    - Argus uses Argus epochs internally: terrestrial time (TT) seconds since midnight of 2025-01-01
    - Argus should ship alerts with timestamps in JD, using the same time scale as BOOM
    - A conformance test needs to exist
        - BOOM uses JD
        - [ ] Sushant: check astropy code for JD ↔ UTC conversion

> [!success] Decisions **Schema** — single schema. Stick with the current setup: an alert contains a candidate with either an event association or an SSO association as an additional child, with the other association nulled.
> 
> **Cutouts**
> 
> - 32×32 pixels, from the science and difference images; references will be accessed on BOOM's side by accessing …
> - Argus will explore stateful alert generation
> 
> |Cadence|Cutout format|Send cutouts?|
> |---|---|---|
> |15 min and up|Centered|Yes|
> |1 min|**A:** centered<br>**B:** just send the minipix|**A:** yes<br>**B:** first detection only<br>**C:** stateful alert generation?|
> |1 s|Minipix only|**A:** first detection only<br>**B:** stateful alert generation?<br>**C:** none|
> 
> **Photometric histories**
> 
> - Argus will not send photometric histories with each alert
> 
> **Alert IDs** — alerts will have two identifiers:
> 
> - A unique identifier, which can decode into an associated human-readable hash (implementation TBD)
> - The associated event/SSO ID
>     - Event IDs are HEALPix-derived, allowing stateless alerts from the same event to be associated on astrometry alone
>     - SSO ID is the condensed MPC identifier

### Machine learning

**Attendees:** Hank, Niall, Michael, Jon

_No notes recorded._

### SSO

**Attendees:** Antoine, Donovan

#### To discuss

- How best to record provenance of the orbit catalog source and processing system version?

#### Discussed

- Argus ships:
    - Catalog association: MPC, JPL (add ESS)
    - Geometry: r, phase angle, distance
    - Ephemeris, magnitude, source, residuals
- New object discovery?
    - Argus-side: no
- Shared infrastructure?
    - Due to a fairly clean split in coverage, nothing seems necessary yet: BOOM will be able to ingest Argus SSO alerts with Argus-computed information and provide further enrichment using catalogs
    - Fixing on the same schedule to ingest orbit catalog sources is a good idea, though