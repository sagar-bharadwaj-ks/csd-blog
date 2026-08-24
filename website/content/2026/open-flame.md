+++
# The title of your blogpost. No sub-titles are allowed, nor are line-breaks.
title = "OpenFLAME: Enabling the Federated Spatial Web"
# Date must be written in YYYY-MM-DD format. This should be updated right before the final PR is made.
date = 2026-09-01

[taxonomies]
# Keep any areas that apply, removing ones that don't. Do not add new areas!
areas = ["Systems"]
# Tags can be set to a collection of a few keywords specific to your blogpost.
# Consider these similar to keywords specified for a research paper.
tags = ["spatial-web", "indoor-mapping"]

[extra]
author = {name = "Sagar Bharadwaj", url = "https://sagar-bharadwaj-ks.github.io/" }
# The committee specification is  a list of objects similar to the author.
committee = [
    {name = "Srinivasan Seshan", url = "https://www.cs.cmu.edu/~srini/"},
    {name = "Anthony Rowe", url = "https://users.ece.cmu.edu/~agr/"},
    {name = "Committee Member 3's Full Name", url = "Committee Member 3's page"}
]
+++

It is 2040, and you’re a hungry, busy PhD student in your office who has to attend a talk in thirty minutes. You place a DoorDash order from your favorite restaurant. An autonomous robot picks up your order from a restaurant in a mall. It then drives itself from the mall to your university, finds your office, and hand-delivers the food to you. You eat and then use your Augmented Reality (AR) glasses to find your way to the conference room in time to attend the talk. More broadly, this points toward a next generation of the Web in which digital information is anchored to physical locations. We refer to this as the Spatial Web.

Realizing such a future requires advances across multiple areas, including multi-modal perception, human–robot interaction, robot manipulation, and augmented reality displays. A largely ignored piece of the puzzle is the underlying mapping infrastructure, i.e., the system responsible for assigning semantic labels to physical spaces and representing relationships among them. In the scenario above, both the delivery robot and the AR glasses depend on maps spanning multiple environments, including the mall, public road networks, and the university’s indoor spaces. 

At first glance, this vision may appear to be already addressed by existing mapping platforms such as Google Maps, Apple Maps, Mapbox, and OpenStreetMap. However, these systems are fundamentally centralized: each is hosted, maintained, and controlled by a single organization. For example, Google Maps is operated entirely by Google. As illustrated in Figure 1, a centralized mapping architecture consists of a centrally managed map database coupled with location-based services that operate over this data to serve application requests. A routing service, for instance, executes shortest-path algorithms on the centralized map to support navigation applications.

<center><figure><img src="./centralizedMapping.png" width="400" alt="Centralized map architecture."><br> <figcaption><b>Figure 1.</b> Centralized map architecture.</figcaption></figure></center>

A key limitation of today’s centralized mapping infrastructure is that applications can only access the information curated and exposed by the organizations that operate these platforms. Private entities such as malls and universities are often unwilling to upload their maps to centralized repositories maintained by third parties. Indoor map data is often sensitive, and ownership of such data is a primary concern for private organizations. Beyond these practical concerns, there is also a broader philosophical issue: we should be cautious about a future in which large portions of physically grounded digital applications (e.g., robotics, augmented reality) are reliant on a small number of dominant corporations.

Our work, [OpenFLAME](https://www.open-flame.com/), proposes a systems infrastructure in which disparate organizations can maintain maps of their own spaces and serve location-based services to applications. In the above scenario, the university, for example, would maintain its own map and services on its own servers. A robot wishing to navigate within the university would request access to the university’s map server and use the services it provides. For instance, it might use a *localization* service to determine its position within the university campus and a *navigation* service to determine where to go. Importantly, access can be fine-grained: the robot does not need access to the entire map, but only to specific services or limited portions of the data required for its task.

In this post, I discuss the design principles behind OpenFLAME and outline its implementation. OpenFLAME draws inspiration from the architecture of the Web, which has several parallels to our mapping problem. On the Web, independent organizations host their own websites and enforce their own access control policies. Analogously, OpenFLAME enables organizations to host maps for their own physical spaces, retaining full ownership of their data and complete control over access. The Web also provides a robust discovery mechanism through DNS: given a human-readable domain name, DNS resolves it to the servers that host the corresponding website. Maps require an analogous mechanism -- given a physical location, the system must be able to discover and identify all maps hosted for that location.

# Design of OpenFLAME

<center><figure><img src="./design.png" width="400" alt="Design of OpenFLAME showing map servers and OpenFLAME client."><br> <figcaption><b>Figure 2.</b> OpenFLAME architecture.</figcaption></figure></center>

Figure 2 shows the simplified architecture of OpenFLAME. Maps of different regions are stored on separate *map servers* maintained by independent organizations. Map servers also provide location-based services on top of the maps that they store. An OpenFLAME client discovers potentially zero or more map servers covering the region of interest using a federated spatial database. It then contacts these discovered servers to obtain services such as routing and localization, combining results from multiple maps when needed.

## Abstractions: Maps and Map Servers

**Map server.** A map server is a system that stores the *map* of a region and provides *map services* on it. For example, a university building's map server would store all the offices and navigable hallways in the building and might provide an image-based localization service to support AR applications. A map server can impose fine-grained security and privacy policies on users and applications.

**Map.** A Map is a representation encoding relationships and attributes of spatial entities in a geographic region. While traditionally, a map refers to the visual representation of geographic features, in our context it is the data that underlies such visual representations. We do not restrict the format in which map data is stored in individual map servers.

**Map services.** Location-based services built on top of maps are called map services.  The green boxes in Figure 2 show some examples of map services. The tile service, for example, returns a visual representation of the map. Clients access map data only through map services.

**Map zone.** A set of map servers are grouped to form a map zone for organizational convenience and ease of delegation. For example, the map servers of the different departments of a university form the map zone for the university.

Each map server is registered under a zone. Both map zones and servers define their own coverage---the spatial extent they are responsible for. The coverage of a map server must lie entirely within the coverage of its zone. However, the coverage of a zone may extend beyond the combined coverage of its servers, meaning that zones can include “empty” regions that are not served by any map server. A zone can delegate the responsibility of parts of its coverage down to sub-zones. A zone can be registered with one or more parent zones. 

For example, consider a university setting up its map on OpenFLAME. The university first defines the coverage of its zone, which spans all buildings on its campus. This allows the university to manage its zone independently of other map zones. It registers its zone with a parent zone, e.g., the city zone. Initially, much of the university zone’s coverage may consist of empty regions not served by any map server. Over time, individual university departments can populate the zone by registering their own servers, each covering their respective areas. Within the zone, these map servers operate independently of one another and their coverages can overlap with each other. The university facilities department, for example, could set up its own sub-zone and maintain all of its maps (e.g., electricity and plumbing plans) within this sub-zone.

The fundamental unit of discovery for spatial applications is the map server. Map zones exist primarily for organizational and administrative convenience of delegation. Applications do not need to interact with zones directly. However, zones can serve as an optional filtering mechanism; i.e., applications may restrict discovery to specific zones if they wish.

## Organization of Map Servers and Zones

<center><figure><img src="./data-organization.png" width="500" alt="OpenFLAME's data organization and query model."><br> <figcaption><b>Figure 3.</b> OpenFLAME's data organization and query model.</figcaption></figure></center>

Map zones form a nested inclusion hierarchy; a child zone must be completely contained within its parent zone. The coverage of map zones can overlap with each other. Map servers are the leaves of this hierarchy. They always have a parent zone, and their coverages may also overlap.

An obvious starting point for organizing the hierarchy of map zones would be to reuse existing geopolitical hierarchies such as countries, states, and cities. However, such traditional hierarchies are not suited for our purpose. First, they are fraught with disputes. National boundaries are contested, and even property lines are frequent sources of legal conflict. Anchoring a technical system to these boundaries risks inheriting political disagreements. Second, geopolitical hierarchies generally assume exclusive ownership, where a region belongs to exactly one parent. This prevents benign overlaps, such as a university and a commercial provider both maintaining maps of the same campus. 

The key difference between our approach and traditional hierarchies is the explicit allowance of overlaps. Multiple zones may cover the same physical region, and each can delegate to its own set of servers. This flexibility enables incremental deployment, where new maps can be attached under existing zones without requiring reorganization. Figure 3 shows an example map zone hierarchy with map servers within zones.

While overlaps enable easier integration of new maps, they also introduce challenges for map server discovery. A discovery query must now search across multiple branches of the hierarchy, increasing the complexity of discovery. We elaborate on this and discuss a feasible implementation in later sections of this post.

Why hierarchy? Well, instead of organizing as a hierarchy, every map server could register its coverage with a centralized service that allows overlaps. This service could leverage existing spatial databases (e.g., MongoDB, PostGIS) to store the coverage of each map server simplifying the discovery process. However, this undermines the goals of federation, since new map registrations and updates to coverage would be controlled by a single entity. A peer-to-peer design would avoid centralization but struggle to scale for complex discovery queries. A hierarchical system provides structure, supports federation through delegation, and scales more naturally with growth.

## Expressing Coverage

Polygons are the most intuitive representation to express the coverage of map servers and zones, but operations such as intersection checks require complex spatial indexes that are costly to build and query in distributed settings. Furthermore, polygon-based queries are not amenable to distributed caching as they rely on exact spatial boundaries. Small differences in query boundaries can cause cache misses, reducing cache reuse and increasing recomputation costs.

We can instead express coverage as a collection of primitive shapes. Existing spatial indexing systems like [H3](https://h3geo.org/) and [S2](https://s2geometry.io/) represent regions using primitive shapes. H3 and S2 decompose the world into hierarchically organized hexagons and squares respectively. The advantage of using spatial indexing systems is that discovery operations can be performed directly on the indices of primitive shapes. Moreover, queries for the same region can be cached using these indices as keys. Precisely expressing an arbitrary region might require a large number of primitive shapes. However, in practice, map coverage boundaries are rarely exact. While it is practical to guarantee that the coverage lies within a boundary, it is often difficult to assert that coverage of a zone or server extends up to every point of that boundary. We can exploit fuzzy boundaries to represent regions approximately and bound the number of primitive shapes required to cover them.

<center><figure><img src="./s2CellsAirport_border.png" width="200" alt="S2 cell covering for a given map boundary."><br> <figcaption><b>Figure 4.</b> <span style="color: red;">S2 cell covering</span> for a given <span style="color: blue;">map boundary</span>.</figcaption></figure></center>

Unlike squares, hexagons do not exactly subdivide into child hexagons. As a result, the inclusion of children in parents is only approximate in H3, making S2 more suitable for representing an inclusion hierarchy of map zones. Therefore, we use the S2 spatial indexing system to express coverage as a collection of S2 cells, together with an optional altitude above sea-level. Figure 4 shows an example of S2 cells representing a map zone. Checkout our tool, [Geodomain Explorer](https://www.open-flame.com/geo-domain-explorer/), to play with automatic generation of S2 coverage. The documentation on how to use the tool can be found [here](https://www.open-flame.com/pages/tools/geodomain-explorer.html).

## Discovery Query Model

A map server discovery query takes as input a list of S2 cells and optionally, altitude and a list of accepted map zones. It returns the set of map server addresses whose coverage intersects with the input S2 cells. Discovery is limited to a specific altitude, if provided. It is also restricted to map servers registered within the accepted map zones list, if provided.

<a name="discoveryQuery"></a>
\\[
    \text{discover(S2 Cells, [altitude], [accepted map zones])} \rightarrow [\text{map server}_1, \text{map server}_2, \cdots]
\\]

For convenience, an application can represent the search region as a 3D bounding volume, i.e., an arbitrary 2D geographic shape (e.g., a polygon or circle) together with an altitude range. The OpenFLAME client library invoked by the application converts the bounding volume to a collection of S2 cells and altitude. 

An application that already knows map zones relevant to its context can restrict the discovery process to those zones. For example, a university navigation application can restrict discovery to the university zone. If no list is provided, the discovery query is answered by first recognizing the set of map zones that cover a region followed by identifying all the map servers within these zones that cover the queried region. A consequence of allowing overlaps of map zones is that discovery queries now have to explore all branches of the map zone hierarchy that have coverage over the queried bounding region. Figure 3 shows an example where a query within a university returns maps across zones such as the city waterworks zone, university's own zone and Google Maps. An advantage of using S2 cells is that the query results can be cached across multiple layers such as client devices, Content Distribution Networks (CDNs), and Internet Service Providers (ISPs).

**Limitations.** The discovery query model we adopt is inherently limited. It does not accept custom search terms and therefore cannot support nuanced queries such as "find only maps of shopping complexes in a city". Importantly, it does not support a ranking criteria to order the discovery results. As a result, querying over a region with many overlapping areas may yield large, unordered result sets. We envision that, once deployed, discovery of maps will evolve much like the Web. While the maintenance and updating of maps must be federated, much like websites on the Web are maintained independently, the discovery of maps can be centralized, analogous to a Web search engine. Centralized search engines would act as curators that crawl regions for available maps, index metadata and the services provided by map servers, and support richer search capabilities. Additionally, mechanisms such as external whitelists, blacklists, and ranking systems (e.g., crowd-sourced voting, auctions) can be employed to further tune discovery results. The discovery mechanism implementation described later in this blog post would still be essential to allow such out-of-band mechanisms to discover maps in the first place. We leave the study of nuanced map searches, filtering, and ranking mechanisms to future work.

## Security Model

**Threat Model.** Once federated mapping is deployed at scale, it will likely reveal a wide and diverse attack surface. Map servers, for example, are vulnerable to attacks such as denial of service (DoS), reflection, and amplification. We believe that existing methods on DoS protection or anomaly detection could mitigate such attacks. In this work, we choose to focus on a subset of threats unique to OpenFLAME that stem from identity spoofing. Identity spoofing in this context refers to an adversary masquerading as a legitimate map server or zone, tricking applications into trusting falsified associations between regions and services. This would lead to threats such as cache poisoning and man-in-the-middle attacks. Identity spoofing may also result in denial of service for spatial applications as they could be bombarded with fake discovery results. Since caching is heavily relied upon in our model, and cached records may not always originate from the authoritative source, ensuring the authenticity of the discovery results is crucial.

**Chain-of-trust model for spaces.** Several existing models provide inspiration for how such identity authentication can be accomplished. For example, the Web relies on [Public Key Infrastructure (PKI)](https://datatracker.ietf.org/doc/html/rfc5280), where trust anchors are global Certificate Authorities (CAs) that issue digital certificates binding identities to public keys. In contrast, [DNSSEC](https://datatracker.ietf.org/doc/html/rfc4033) and [BGPSEC](https://datatracker.ietf.org/doc/html/rfc8205) use hierarchical chain-of-trust systems, where authority is delegated step by step (e.g., root to top-level domain to child domain), and each link in the chain validates the next.

We adopt a hierarchical chain-of-trust model for OpenFLAME in which each zone signs all map servers and child zones registered with it. Clients performing discovery have configured trust anchors and validate results by ensuring that the signature chain extends to those anchors. We argue that this model is suitable for spaces as proving ownership of a map of a physical space requires proximity, which is something local or parent zones are more likely to possess than distant CAs. Furthermore, it is easier for new participants to prove their relationship to a parent zone than to undergo validation from an external CA (e.g., it is easier for a student club wanting to host a map of the university to request for validation from the university admins than from a centralized CA). Our data model is inherently hierarchical, making it natural for a parent zone that references a child to also provide a signed attestation of that child’s validity.

*Limitations.* Hierarchical chain-of-trust is inherently easier to compromise than a tightly controlled centralized PKI, as careless or malicious child zones may issue faulty signatures compromising the rest of the chain below them. As described in the previous section, we expect future spatial applications to rely on external curators, filter lists, and ranking mechanisms to select maps in a region. The security mechanism here serves to establish a baseline for identity.

**Communication with Map Servers.** Once map servers are discovered, clients can authenticate with them using standard mechanisms like password-based login or [OAuth](https://datatracker.ietf.org/doc/html/rfc6749). As map servers are discovered dynamically, it becomes increasingly important for the client to authenticate the server's identity (e.g., through [X.509 certificates](https://datatracker.ietf.org/doc/html/rfc5280) or [DNSSEC](https://datatracker.ietf.org/doc/html/rfc4033)). Subsequent communication between clients and servers to obtain map services can leverage whatever security protocols are most appropriate for the application context. For instance, an image-based localization service might incorporate privacy-preserving features to preserve confidentiality of both the server and the client device (e.g., [privacy-preserving VSLAM](https://xdspacelab.github.io/lcvslam/)). 

# Implementation on the DNS

## Why DNS?

Existing spatial databases, such as PostGIS and MongoDB, can support OpenFLAME [discovery query](#discoveryQuery) off-the-shelf. They provide datatypes to represent polygons, can index geospatial coordinates, and have fast intersection operators to answer discovery queries. Hierarchy and delegation of zones can be implemented as a collection of spatial databases. However, using such databases would require developing mechanisms for federated and distributed operations, including query routing, response merging, and response caching. We believe that it is better to begin with a design that lends itself to distributed operation from the start. 

The [data model](#design-of-openflame) of OpenFLAME map servers and zones is analogous to the DNS model in several ways. 

- The map zone is akin to a DNS zone representing an autonomous zone of administration. A link to a map server from a map zone is akin to a DNS record within a DNS zone. 
- DNS has a similar model of hierarchy and delegation for its zones as is described for map zones.
- The DNSSEC model follows the chain-of-trust mechanism with pre-configured trust anchors similar to the security model described in [earlier](#security-model).

Beyond the data model, DNS infrastructure also aligns well with the needs of OpenFLAME discovery.
- DNS has pervasive distributed caching across multiple layers such as local caches, ISP resolvers and CDNs which could benefit OpenFLAME discovery queries. 
- Discovery queries are exclusively read queries and do not need a full-fledged database with transaction processing. 
- Map servers and zones are not expected to change very frequently so we do not need low update latencies making DNS suitable.

While the models of OpenFLAME and DNS share many similarities, key differences make it non-trivial to adopt DNS directly.
- DNS does not natively support location-based queries or registrations. Later in this post, we show how regions can be encoded as collections of domain names compatible with DNS. 
- Unlike DNS where each zone is delegated to a single owner, OpenFLAME permits overlaps between the coverages of map zones. As a result, discovery queries may need to traverse multiple delegation chains rather than a single path. We later introduce additional records and query mechanism to support such overlaps.

## Geo-domains

*Geo-domains* are the DNS-compatible addresses representing a physical region. We use the spherical geometry library [S2](https://s2geometry.io/) to generate geo-domains from a bounding region. S2 projects the Earth's surface onto a perfect mathematical sphere. The sphere is then decomposed into a hierarchy of cells called S2 Cells. Each S2 Cell is a quadrilateral bounded by four spherical geodesic lines---lines along the shortest path on a sphere. The highest level in the S2 hierarchy at level 0 is called a *Face*. Faces divide the Earth into six large quadrilaterals. The top-level faces are recursively subdivided at each level into 4 smaller children. Cell levels range from 0 to 30, and the smallest cells at level 30 have an area of about 1 cm<sup>2</sup>. The six top-level cells are numbered from 0 to 5. At each level, the four child cells are numbered from 0 to 3. Each S2 cell has a 64-bit index that is essentially a concatenation of the hierarchical cell numbers in binary form with padding to extend the length to 64 bits. The S2 library also provides the region coverer algorithm---an API that returns the set of S2 cells that approximately cover a given bounding region.

<center><figure><img src="./s2IndexToGeodomain.png" width="300" alt="Converting S2 cell index to geo-domain."><br> <figcaption><b>Figure 5. </b>Converting S2 cell index to geo-domain.</figcaption></figure></center>

The process of converting an S2 cell index to a geo-domain is shown in Figure 5. The S2 cell index encodes the full hierarchy of the cell up to the top-level face which we convert to a DNS-compatible format. The geo-domain is organized such that the top-level face is at the right end and the smallest cell is at the left end of the domain name. We need a *domain suffix* to represent the root domain of the geo-domain. In our examples, we use `.loc`. The altitude of the S2 cell, rounded up to the nearest integer in feet, is prefixed to the geo-domain. If the altitude is unknown, the letter `U` is prefixed instead.

## DNS Records

We introduce three new DNS record types to represent map servers, zones, and delegation signers. Although the structures of data in these records are same as the existing DNS records (`NS`, `CNAME`, and `DS` respectively), we introduce new records to support overlaps between zones. The owner names of all of these records are geo-domains.

- **`MAPSERVER`** record holds the address of a map server. Its structure is the same as a `CNAME` record on traditional DNS that holds the canonical name (or alias) of a domain name. Discovery queries request `MAPSERVER` records associated with one or more geo-domains. A geo-domain can have multiple `MAPSERVER` records associated with it to allow overlaps, unlike `CNAME`.  A request for a `MAPSERVER` record informs the resolution system (e.g., DNS recursive resolver) that multiple delegation chains need to be followed to answer this request.
- **`MAPZONE`** acts like an `NS` record in DNS, i.e., it delegates authority for a sub-zone to the appropriate name servers. Unlike `NS`, however, the same geo-domain may have multiple `MAPZONE` records, each leading to a different delegation chain. While DNS allows multiple `NS` records for the same domain, the underlying assumption is that they all lead to the name servers maintained by the same administrative entity with the same data. However, for OpenFLAME discovery queries, resolvers have to follow the delegation chains of all `MAPZONE` records encountered.
- **`MAPDS`** functions like a `DS` record in DNS, which DNSSEC uses to authenticate the delegation from a parent zone to a child zone. The key difference is that a single geo-domain can hold multiple `MAPDS` records, each corresponding to a different delegation chain.

To ensure backwards compatibility with existing DNS servers, we wrap the above records in `TXT` records which can hold arbitrary data and is universally supported by all DNS implementations. As a result, map zones can be hosted on standard DNS nameservers without any changes. We implement map servers as web servers that implement map services such as localization, routing, and geocoding.

## Map registration workflow

To understand the map registration workflow, let us consider an example of hosting an airport's map on OpenFLAME; for clarity, we omit signature and authenticated delegation records here and defer their discussion.

First, a polygon is roughly drawn around the map region (see [Geo-domain explorer tool](https://www.open-flame.com/geo-domain-explorer/)). S2's region-coverer algorithm is used to generate S2 cells covering the polygon. In Figure 3, the blue polygon represents the map's boundary and each red square represents an S2 cell. The covering does not perfectly align with the blue polygon marking the map boundary. This is acceptable since maps on OpenFLAME have fuzzy boundaries. The altitude of the map is mentioned or it is marked as unknown.

For each S2 cell, a geo-domain is generated as shown in Figure 4. A `MAPZONE` record for each geo-domain is registered at the parent zone’s DNS nameserver (e.g., the nameserver hosting the city map zone), enabling queries within the airport to be delegated to the airport’s map zone. As described earlier, the airport can choose any parent that has coverage over the airport. These `MAPZONE` records point to the airport’s DNS nameserver. If altitude information is available, an additional set of geo-domains is registered with the altitude marked as unknown (using the prefix `U`). This allows the zone to remain discoverable even for clients that lack altitude data.

Once the airport zone is established, individual map servers can register `MAPSERVER` records in the airport's DNS server pointing to the corresponding servers providing map services. For instance, each terminal may operate its own map server. The airport zone may also delegate portions of its coverage to sub-zones by publishing `MAPZONE` records in its nameserver.

## Discovery query workflow

As described in the [discovery query model](#discovery-query-model), an application expresses search region as a bounding volume (i.e., a 2D geographic region with altitude). The OpenFLAME client first converts the request to a [discovery query](#discoveryQuery) and then represents it as geo-domains. Figure 5 shows our technique for converting a bounding region to geo-domains.

<center><figure><img src="./genGeoDomainsDiscovery.png" width="400" alt="Generating geo-domains for discovery."><br> <figcaption><b>Figure 6.</b> Generating geo-domains for discovery.</figcaption></figure></center>

1. We use S2's interior region coverer algorithm to get a set of S2 cells that cover the 2D geographic region. We call these the *base S2 cells* (Figure 6b).
2. For each base S2 cell, its corresponding geo-domain is generated (Figure 6c). We call these geo-domains the *base geo-domains*. 
3. For each base geo-domain, we generate all the parent domains by removing sub-domains sequentially from the left. We retain the altitude. For example, the parents of the domain `U.1.3.5.loc` are `U.3.5.loc`, and `U.5.loc`. (Figure 6d). Querying parents is essential to discover maps whose coverages are larger than the queried region. Optionally, we also add the children of base geo-domains upto a predefined level.
4. The set of geo-domains to query includes all the base geo-domains, and their parents with duplicates removed (and optionally, their children). Optionally, a copy of the same geo-domains but with altitude marked as unknown is also queried to discover map servers not registered with a known altitude. Queries for `MAPSERVER` records of all these geo-domains are made in parallel to get a list of map servers for the queried region.

<center><figure><img src="./intersection.png" width="300" alt="Intersection of queried and registered geo-domains."><br> <figcaption><b>Figure 7.</b> <b>Intersection</b> of <b><span style="color: purple;">queried</span></b> and <b><span style="color: red;">registered</span></b> geo-domains.</figcaption></figure></center>

Figure 7 illustrates a discovery query. Red S2 cells show the geo-domains registered for a map server serving a museum, while purple S2 cells show the geo-domains generated and queried for the input region (the small purple circle). The intersection of the registered and queried geo-domains, shown with black stripes, ensures that the address of the map server is returned by the DNS when the discovery query is made. 

**Caching.** In every discovery cycle, the OpenFLAMe client typically queries about 40 geo-domains (about 10 base geo-domains and 30 parent geo-domains). The ubiquitous caching mechanisms in DNS make such a large number of queries feasible. Most of the geo-domain queries are answered by the local DNS cache and never leave the device. This is because some of the top-level parent geo-domains (Figure 5d) rarely change as the queries from a client are usually limited to a small region in the real world. 

**Fan-out queries.** In traditional DNS resolution, a query, such as one for a `CNAME` record, proceeds by following a delegation chain through `NS` records. If multiple `NS` records are present, the resolver arbitrarily selects one and follows only that chain, since all entries are assumed to lead to equivalent answers. In OpenFLAME, this assumption does not hold because overlapping zones may create multiple valid delegation chains for the same geo-domain. To correctly resolve a `MAPSERVER` record, the resolver must therefore traverse all `MAPZONE` records in parallel, exploring every chain of delegation rather than selecting one. In our implementation, to maintain backwards compatibility, we leave standard DNS resolvers unmodified. Instead, the client issues `TXT` queries for the requested geo-domains. It then queries name servers in each `MAPZONE` record (wrapped in a `TXT` record) to get `MAPSERVER` records. 

## Security/Validation

The security model for spaces described in [earlier](#security-model) follows a chain-of-trust approach similar to DNSSEC. In DNSSEC, each record is signed using an `RRSIG` record, and these signatures are validated against public keys stored in `DNSKEY` records. Delegation between zones is authenticated using `DS` records, which establish trust from a parent to a child zone.

Our implementation reuses DNSSEC’s `DNSKEY` and `RRSIG` records as well as its standard zone-signing protocols. The key difference lies in how delegation is handled. To support multiple delegation chains for the same geo-domain, we introduce a new record type, `MAPDS`. A geo-domain may have several `MAPDS` records, each corresponding to a different delegation chain. As a result, authenticating a `MAPSERVER` or `MAPZONE` record may require traversing all relevant delegation chains.

# Conclusion

