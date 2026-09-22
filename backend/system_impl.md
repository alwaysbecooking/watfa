# Recognition, indexing and search

This is a working implementation outline, not a fixed specification. The first field pilots should decide which parts survive.

## System boundary

WATFA needs two connected systems:

1. An observation system that records what was seen, where, when and under what survey conditions.
2. An identity-resolution system that suggests when observations may show the same animal.

The observation system must remain useful when no identity can be assigned. A sighting is evidence; an individual profile is a revisable interpretation of several sightings.

The recognition goal is:

> Given one or more photographs, return a short, ranked set of possible matches and enough evidence for a person to review them.

Every search must allow `no reliable match`. New animals will enter the dataset constantly, so forcing each photograph onto an existing profile would quickly corrupt the data.

## Data concepts

- **Observation:** An event at a time and place. It may come from a planned survey or an ad-hoc submission.
- **Media asset:** An original photograph or video attached to an observation.
- **Animal instance:** One dog or cat visible in a media asset. A group photograph may produce several instances.
- **Individual:** A working profile that groups instances believed to show one animal.
- **Match assertion:** A proposed, confirmed, rejected or disputed relationship between instances or individuals.
- **Lost/found case:** A search request containing reference photographs, last-known details and contact rules.
- **Survey session:** The route, area, duration, conditions and people involved in a planned survey, including sessions where no animals were recorded.

Observations, original media and their revision history should remain available even when an identity decision changes. Individual profiles need reversible merge and split operations. Names should be aliases because different neighbourhood groups may use different names for the same animal.

## End-to-end flow

```text
photos + time + location + survey context
                    |
                    v
          store the original observation
                    |
                    v
       detect and crop each animal separately
                    |
                    v
      assess species, view and image quality
                    |
                    v
  create visual representations of useful crops
                    |
                    v
       retrieve and rank possible matches
                    |
                    v
       human: same / different / uncertain
                    |
                    v
 update identity links, profiles, timelines and alerts
```

Image processing should run asynchronously. A submission can be accepted immediately while detection and matching continue in the background.

## Collection and intake

The capture process affects recognition quality as much as the model. Organised surveys should try to collect:

- a full-body photograph;
- left and right flank views;
- a face or head view when it is safe;
- visible scars, coat patches, ears and tail shape.

Ad-hoc photographs should still be accepted. The interface can warn about blur, distance or a hidden animal without rejecting the observation.

For each submission, retain:

- the original file;
- capture time and submitted time;
- exact internal location and a separate public location representation;
- source, uploader and photographer;
- survey session when applicable;
- image licence and consent information;
- notes about health, behaviour and visible attributes.

Strip public EXIF data. Faces, number plates, house numbers and other identifying background details may need blurring before publication.

## Image processing

A background worker would perform several independent jobs:

1. Find every dog or cat in the image.
2. Create a crop or segmentation for each animal.
3. Estimate whether the view is front, left, right, rear or unknown.
4. Measure blur, occlusion, crop size and other quality signals.
5. Detect exact and near-duplicate photographs.
6. Create visual embeddings for usable animal regions.
7. Extract tentative attributes such as coat colour and pattern.

Detection and identification are separate jobs. A detector can find a dog without knowing which dog it is. If automatic detection misses an animal, a reviewer should be able to draw or correct its region manually.

Poor images can still document presence, location or welfare. Mark them as unsuitable for identification instead of deleting them.

## Visual matching

An embedding model converts an animal crop into a vector. Photographs of the same animal should sit close together in that vector space. A new animal can then be added without retraining a classifier with one class per animal.

The first experiments should compare existing models rather than train one from scratch. Likely baselines include animal re-identification models such as MegaDescriptor and general visual models such as DINOv2. Face-specific models can contribute another signal, but street photographs often show only a flank, back or partial body.

The matcher should eventually combine several kinds of evidence:

- whole-body appearance and coat pattern;
- face and muzzle when visible;
- ears, tail, scars and local markings;
- viewpoint and image quality;
- manually or automatically recorded attributes;
- distance and elapsed time between sightings.

Store several embeddings per individual rather than one average vector. An average of front, rear and side views may represent none of them well. Search against image-level vectors, then group and score the results by individual.

### Candidate retrieval

For each new instance:

1. Search compatible species and views for visually similar images.
2. Group those images by their assigned individual.
3. Add plausible nearby and recently seen individuals.
4. Re-rank the combined candidates using visual, geographic and temporal evidence.
5. Present a small candidate set for review.

Geography should influence rank without becoming a hard identity rule. WATFA wants to detect animals that have moved, gone missing or been relocated. A strict neighbourhood filter would hide those cases.

Run two retrieval paths:

- A local search for routine re-sightings, with a strong location and time prior.
- A city-wide search for lost, found and unexpectedly relocated animals.

The final candidate set is the union of both. Animals appearing together in one photograph create a reliable `different individual` constraint. Proximity alone does not create a reliable `same individual` label because packs often share feeding and resting sites.

### Human review

The review interface should show:

- query and candidate images side by side;
- relevant views at useful zoom levels;
- dates and approximate distance;
- earlier confirmed sightings;
- model and metadata scores without presenting them as certainty;
- actions for `same`, `different`, `uncertain`, `new individual` and `bad crop`.

Store every decision with the reviewer, timestamp, evidence, model version and previous state. Rejections are useful training data and should not disappear.

Automatic linking should wait until the system has been measured on local data. Even then, the threshold should be high and the link reversible. A false merge can attach the wrong health, movement and lost/found history to an animal.

## Storage and indexes

A pilot does not need a separate service for every concern. A small backend can use:

- object storage for original and derived images;
- PostgreSQL for observations, individuals, decisions and permissions;
- PostGIS for exact and fuzzy geographic queries;
- pgvector for visual embeddings;
- a queue plus Python workers for image processing and matching;
- an API used by the website, forms and future bots or apps.

PostgreSQL can support the initial search surfaces:

- ordinary indexes for species, dates, states and attributes;
- spatial indexes for radius, route and neighbourhood queries;
- vector indexes for appearance search;
- full-text search for notes and aliases;
- perceptual hashes for duplicate-image search.

Every derived result should record the model and preprocessing version that produced it. When a better model arrives, workers can create a new set of embeddings and rebuild the appearance index without changing the original observations.

## Search surfaces

The project needs several kinds of search.

### Dataset search

Browse observations and individuals by species, time range, public area, attributes, health notes and review state. Public maps should use delayed, aggregated or fuzzy locations.

### Search by image

Upload a photograph and receive possible matching observations or individuals. Results should say `possible match`, not `identified as`, until reviewed.

### Individual history

Show confirmed sightings as a timeline, with proposed or disputed sightings clearly separated. Profile pages should expose why each observation is attached.

### Lost/found search

Run reference photographs against existing observations and continue comparing them with new submissions. This is a saved search, not a one-time query.

### Research export

Provide versioned, documented exports with stable identifiers, field definitions, missing-value conventions and location rules. Restricted exports may contain more detail than public exports.

## Lost-animal matching

A lost report should accept several photographs from different dates and viewpoints. Those photographs form a query gallery.

When the case is created, the system searches historical observations. Each suitable new sighting is also checked against active cases. Appearance, time since disappearance and movement plausibility can change the ranking, but a long distance must not suppress a strong visual candidate.

Alerts should go through review before exposing a recent precise location. Owner confirmation is evidence, but distress and hope can bias a decision. High-stakes matches may need a second reviewer or physical verification.

## Public and restricted data

An open dataset does not require publishing every raw field. WATFA can publish its schema, code, methods and privacy-safe records while restricting precise recent locations and contact details.

The system should maintain separate representations:

- a protected record with exact coordinates and original media;
- a reviewer view with the detail needed for matching;
- a public view with fuzzy location, redacted media and delayed or aggregated movement data;
- a research export governed by an explicit access policy.

Access rules should apply at query time and export time. Hiding a map marker is not enough if exact coordinates remain available through an API or image metadata.

## Evaluation and training

The first useful training set will come from repeated surveys where local volunteers can establish identity across days and viewpoints. Random internet pet photographs will not measure performance on Guwahati street animals.

Evaluate the system on photographs separated by time, camera, weather, location and pose. Do not put adjacent frames from the same burst or survey session on both sides of a train/test split.

Useful measures include:

- whether the correct animal appears in the first few results;
- how often the system proposes the wrong animal with high confidence;
- how well it rejects animals absent from the index;
- how many candidates a reviewer must inspect;
- how often confirmed profiles later require a split;
- performance by image quality, viewpoint, species and area.

The operating target should favour missed links over false merges. Candidate search can cast a wider net because a human reviews it. Profile assignment needs much stronger evidence.

Confirmed matches and rejections become local same/different training pairs. Fine-tuning should begin only after enough reliable pairs exist. Nearby sightings should not be treated as automatic positive pairs; several different animals may occupy the same patch.

## Research use

Population work needs more than recognised animals. Planned surveys should record routes, duration, weather, observer effort and zero-sighting stretches. Ad-hoc community photographs over-represent friendly animals, feeding points and busy roads.

Population estimates and individual histories also tolerate different levels of matching error. Health histories, relocation claims and lost-pet alerts need confirmed identity. Population models may consume probabilistic links, but they must account for identity uncertainty and uneven survey effort.

## Expected problems

- New animals constantly enter the dataset, so closed-set classification will fail.
- Many local dogs share similar colour, build and facial features.
- One animal can look different after rain, injury, weight change, pregnancy, ageing or seasonal coat change.
- Community photographs will contain blur, occlusion, crowds and inconsistent viewpoints.
- Pet-photo datasets may transfer poorly to Indian street animals.
- Models may learn a tea stall, feeder or street corner instead of the animal.
- Similarity scores are not probabilities unless calibrated on local data.
- False matches accumulate as the city-wide gallery grows.
- Identity review may become a larger workload than model inference.
- Dogs and cats may need different models, capture advice and thresholds.
- Public photographs and movement histories can expose animals and people to harm.

## Existing work to examine

### Wildbook and WBIA

[Wildbook](https://wildbook.docs.wildme.org/introduction/) is the closest mature reference architecture. It separates sightings, encounters, image annotations and individuals. Its matching pipeline returns ranked candidates for human confirmation. [WBIA](https://github.com/WildMeOrg/wildbook-ia) is its open-source image-analysis backend.

Wildbook is worth testing before deciding to build the full workflow from scratch. Its data model and review process are useful even if its application proves too heavy or its species models do not fit WATFA.

### Namma Indies

[Namma Indies](https://nammaindies.org/technical.html) describes almost the same open-set problem for free-ranging Indian dogs. Its proposed design combines appearance embeddings, geographic and temporal priors, PostGIS, pgvector and human confirmation.

WATFA should talk to them and inspect their code. Their local geo/time prefilter is useful for routine sightings, but WATFA also needs a global path so relocation and lost-animal matches are not filtered out.

### Petco Love Lost

[Petco Love Lost](https://support.lost.petcolove.org/hc/en-us/articles/1500007704782-How-does-Petco-Love-Lost-work) searches lost and found pet listings using size, colour, facial features and coat attributes. It shows that photo-based lost-pet retrieval can operate as a public service, though the model, thresholds and error rates are proprietary.

### Models and datasets

- [MegaDescriptor and WildlifeTools](https://wildlifedatasets.github.io/wildlife-tools/megadescriptor/) provide animal re-identification models, training tools and baselines.
- [PetFace](https://arxiv.org/abs/2407.13555) is a large animal-face identification benchmark covering seen and unseen individuals.
- [DogFaceNet](https://github.com/GuillaumeMougeot/DogFaceNet) is an older dog-face metric-learning project. It may serve as a baseline or dataset source, but its original implementation is not a production system.
- [IndieCare AI](https://indiecare.ai/projects/) reports free-roaming-dog survey work at IIT Guwahati and in Udalguri. Its field methods and local experience may be more useful to WATFA than its publicly described technical details.

## Build sequence

### 1. Observation foundation

Build intake, media storage, privacy controls, survey records, manual profiles, reversible identity links and exports. Recognition should be optional at this stage.

### 2. Repeated field pilot

Survey a few neighbourhoods over several weeks. Collect multiple views and establish a small set of identities with help from people who know the animals.

### 3. Offline matching experiment

Run several pretrained models over the pilot data. Compare candidate ranking, unknown rejection and performance across viewpoints. This experiment should decide whether arbitrary community photographs contain enough identity signal and whether the capture protocol needs to change.

### 4. Reviewer workflow

Add detection correction, candidate comparison, `same/different/uncertain` decisions and profile merge/split tools. Keep all links human-confirmed.

### 5. Local tuning

Use reviewed pairs to train or calibrate the best baseline. Re-index the data under a new model version and compare it with the previous one.

### 6. Lost/found alerts

Add saved image searches, matching against new sightings, controlled notifications and precise-location access rules.

### 7. Limited automation

Consider automatic links only after measuring false merges on local, months-apart data. It may turn out that ranked candidates plus quick human review are safer and useful enough without automatic identity assignment.

## Questions for the pilot

- Are ordinary ad-hoc photographs good enough, or does useful matching require guided multi-view capture?
- Which body regions work best for local dogs and cats?
- How often does the correct individual appear in the first few candidates?
- How much does location improve routine matching, and how often would it hide real movement?
- Who can review uncertain matches, and how much review work can the community sustain?
- Should the first recognition experiment cover dogs only?
- Can Wildbook be adapted, or is a smaller custom system easier to maintain?
- Can WATFA share methods or data standards with Namma Indies and IndieCare AI?
- Which fields and images can be public without exposing precise animal movements or people?

The first implementation milestone is a search that regularly puts the correct prior sighting in a short candidate list while leaving unseen animals unmatched. That would already make manual curation and lost-animal searches much faster.
