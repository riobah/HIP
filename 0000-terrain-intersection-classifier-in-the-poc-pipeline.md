# Terrain Intersection Classifier in the PoC Pipeline

- Author(s): <!-- your GitHub @username -->
- Start Date: <!-- fill me in with today's date, YYYY-MM-DD -->
- Category: technical
- Original HIP PR: <!-- leave this empty; maintainer will fill in ID of this pull request -->
- Tracking Issue: <!-- leave this empty; maintainer will create a discussion issue -->
- Vote Requirements: <!-- veHNT Holders, veIOT Holders, or veMOBILE Holders -->

## Summary

[summary]: #summary

Beacons and data sent over the IOT network do not propagate through significant terrain, such as mountains, or hills. Signals are generally only received if there is a clear line of sight between the transmitter and the receiver. This proposal adds a step to the Helium Proof of Coverage (PoC) pipeline to verify the line of sight between the asserted locations of hotspots and their witnesses. The witness reports will be marked as *invalid* if the line of sight is blocked and successful radio communication is unlikely between the beaconer and the witness.

## Motivation

[motivation]: #motivation

On January 13, 2022, the community [approved](https://github.com/helium/HIP/blob/main/0040-validator-denylist.md) that Nova Labs would be allowed to operate and enforce a [deny list](https://docs.helium.com/iot/denylist) to handle widespread abuse of the Helium Proof of Coverage (PoC) system. On [August](https://docs.helium.com/devblog/2023/08/07/denylist-evolution/) and [September](https://docs.helium.com/devblog/2023/09/14/denylist-refine/) of 2023, the deny list process is improved to have automated classifiers analyzing the PoC data to detect abusers.

One of the classifiers used is the *Terrain Intersection Classifier*, which use the NASA Shuttle Radar Topography Mission (SRTM) [data](https://www2.jpl.nasa.gov/srtm/) and the hotspots' asserted location information to check the availability of clear line of sight (LoS). If the LoS is found to be blocked and successful radio communication is unlikely between the beaconer and the witness, the beaconer - witness pair is placed in the edge deny list, to mark their witness reports against each other as invalid with the reason of `denied_edge`. 

The algorithm has already proven it maturity and its implementation is optimized heavily by Nova, allowing it to be ran in the PoC pipeline. With this proposal, this algorithm having a major impact on the PoC process will be moved to the PoC pipeline permanently, taking it into the regular governance model of Helium network. 

While mostly automated, the deny list is still a weekly process governed by human review. This change will enable the process run fully automatically too.

## Stakeholders

[stakeholders]: #stakeholders

Hotspots owners, in particular those which already have denied edges with the `terrain_intersection` reason will be impacted by this proposal. There will be no change in terms of their rewards. The only change will be the reason of the invalid witness reports. Instead of `denied_edge`, the reason will be `terrain_intersection`.

## Detailed Explanation

[detailed-explanation]: #detailed-explanation

- Introduce and explain new concepts.
- It should be reasonably clear how the proposal would be implemented.
- Provide representative examples that show how this proposal would be commonly used.
- Corner cases should be dissected by example.


### Terrain-Aware Signal Verification

[terrain-aware-signal-verification]: #terrain-aware-signal-verification

Beacons and data sent over the IOT network do not propagate through significant terrain, such as mountains, or hills. Signals are generally only received if there is a clear line of sight between the transmitter and the receiver. The public NASA Shuttle Radar Topography Mission (SRTM) [data](https://www2.jpl.nasa.gov/srtm/) will be used to determine the terrain profile between hotspots. This terrain profile is used to evaluate the line of sight between the asserted locations of hotspots and their witnesses. This concept is illustrated below:

<img src="files/0000-terrain-intersection-classifier-in-the-poc-pipeline/distance-vs-terrain.png" alt="Distance vs Terrain">

The line of sight between the transmitter and the receiver is shown in orange, and the terrain between them is blue. In this example, the line of sight is blocked and successful radio communication is unlikely. The amount of terrain blocking communication is measured by taking the area under the terrain curve and above the line of sight. This number will be zero for a perfect line of sight between the transmitter and the receiver, and large if it intersects significant terrain features. This measurement is called as “terrain intersection”.

Here is an example of the terrain intersection for a hotspot with abnormal behavior, and one which is correctly asserted:

<img src="files/0000-terrain-intersection-classifier-in-the-poc-pipeline/abnormal-terrain-intersection.png" alt="Abnormal Terrain Intersection">

Note the line of sight in the abnormal case goes through a very large amount of terrain.

<img src="files/0000-terrain-intersection-classifier-in-the-poc-pipeline/normal-terrain-intersection.png" alt="Normal Terrain Intersection">

Note that the line of sight is well above the terrain in the normal case. ~~The classifiers will allow some terrain intersections to account for some terrain overlap.~~ (*)

The witness report is marked as invalid when the terrain intersection for beacon and witness pair is unusually high.

### New Invalid Witness Reason: `terrain_intersection`

There will be a new invalid reason, `terrain_intersection`, which will signal the hotspot is reporting an impossible witness report for the beacon, because of terrain obstruction.

## Drawbacks

[drawbacks]: #drawbacks

The main drawback of this proposal is that it increases the complexity of the `iot_verifier` oracle and the poc verification pipeline in general. As this proposal primarily moves an existing mechanism (currently as part of the denylist) to the oracle pipeline we believe that the increased complexity is less of a maintenance burden than the manual intervention needed in the denylist mechanism.

## Rationale and Alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

The alternative to this solution is to maintain the status quo where an edge is added to the denylist. We think an automated system within the PoC pipeline is a better solution because,

1. This will decrease the size of the edges deny list.
2. The process will be fully automatized 
3. A major algorithm having an impact on the PoC process will be take into the regular Helium governance model.

## Unresolved Questions

[unresolved-questions]: #unresolved-questions

- What parts of the design do you expect to resolve through the HIP process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature?
- What related issues do you consider out of scope for this HIP that could be addressed in the
  future independently of the solution that comes out of this HIP?
- Are there dependencies, milestones, or dates that need to be met for this HIP to succeed?

## Deployment Impact

[deployment-impact]: #deployment-impact

When this proposal passes and is deployed the `iot_verifier` oracle will be upgraded to the version including the proposed changes. After the `iot_verifier` update is successfully deployed the denylist process will cease to include the `terrain_intersection` metric the first denylist iteration _after_ deployment.

The explorers will have to implement the new invalid reason `terrain_intersection` to provide the user with correct feedback on why their hotspot is currently not considered for rewards.

The documentation will have to be changed to reflect the new invalid reasons `terrain_intersection`. 

## Success Metrics

[success-metrics]: #success-metrics

What metrics can be used to measure the success of this design? Are any new ETL reports needed to
measure the success?

- What should we measure to prove a performance increase?
- What should we measure to prove an improvement in stability?
- What should we measure to prove a reduction in complexity?
- What should we measure to prove an acceptance of this by its users?
