# TeLex-KG: RDF schema, SHACL shapes and case-study graph

This resource accompanies the paper *A Temporal Knowledge Graph Schema to Support Point-in-Time Grounding of LLM Answers over Legal Sources* (KGSWC 2026). It contains the TeLex-KG schema in RDF, the SHACL shapes for the conditions C1, C2 and C3 and for the consistency of the graph, and the RDF graph of the two examples in Section 5 of the paper together with a small set of clearly marked synthetic examples. The point-in-time queries with their expected answers, and the language model questions and responses of Section 5.2, are given below. Version 1.2.0.

## Files

|File|Content|
|-|-|
|`README.md`|this description, the queries with their expected answers and the language model responses|
|`telex.ttl`|the schema: node types, relations and attributes, aligned with ELI|
|`telex-shapes.ttl`|SHACL shapes for C1, C2 and C3 and for the consistency of events, bounds, hierarchy, texts and conditions|
|`case-study.ttl`|Part 1: the two examples of Section 5, built by hand from the official texts; Part 2: clearly marked synthetic examples|

The graph was built by hand from the official texts in the Croatian Official Gazette (Narodne novine). Every act is identified by its ELI, for example `https://narodne-novine.nn.hr/eli/sluzbeni/2022/119/1834`, so every statement can be checked against its source. Components, versions, texts, events and conditions are identified in the namespace `https://w3id.org/telex-kg/id/`, from the ELI path of the act, the component, the entry-into-force date of the version and the language. Classes and properties defined by TeLex-KG use the prefix `tlx:` (`https://w3id.org/telex-kg/ontology#`). These IRIs are identifiers; redirects are not configured, so they do not resolve to documents.

Tested with Python 3.12.3, rdflib 7.6.0, pySHACL 0.40.1 and Oxigraph (pyoxigraph) 0.5.11.

## How to check the graph

```
pip install pyshacl
pyshacl -s telex-shapes.ttl -e telex.ttl case-study.ttl
```

The expected output is `Conforms: True`. The shapes are run without RDFS inference on purpose: domain and range inference would type an undescribed resource (for example the target of HAS\_CONDITION) and hide a missing description. The shapes check the following.

|Shapes|What they check|
|-|-|
|C1|versions of the same component do not overlap in force|
|C2|versions whose application intervals overlap each state an application rule|
|C3|for each interval, a version has either one last day or one reading of the absent end; pending\_condition requires a described termination condition that affects that interval and has no recorded fulfilment; an unresolved condition on an interval without an end requires pending\_condition|
|events|opening and closing events agree with the interval bounds, each relation for its own interval; every stored bound is justified by such an event|
|structure|exactly one start of legal force and one start of applicability, of type `xsd:date`; no end before its start; one parent per component, an act or a component, without cycles; HAS\_CONDITION points to a TerminationCondition; `tlx:text` is a language-tagged string whose tag matches `eli:language`; `tlx:textSource` is an IRI; one text unit per language and version|
|conditions|complete description (text, act, provision, affected interval, effective date); effective date not before the entry into force of the prescribing provision; status agrees with FULFILLED\_BY and with the requirements; FULFILLED\_BY is the event that met the last requirement and closes the affected intervals|

Two quick tests show the shapes at work.

1. **C3.** In `case-study.ttl`, delete the line `tlx:openApplicabilityEnd tlx:NotExtracted ;` from the version of Article 107. The validator reports one C3 violation for applicability.
2. **C1 and C2.** Add the following lines to the end of `case-study.ttl`. The validator reports two C1 and two C2 violations, one of each for each version.

```
<sluzbeni/2022/119/1834/art\_107/v/2022-12-01> a tlx:TemporalVersion ;
    eli:is\_member\_of <sluzbeni/2022/119/1834/art\_107> ;
    eli:first\_date\_entry\_in\_force "2022-12-01"^^xsd:date ;
    tlx:openEnd tlx:NonePrescribed ;
    eli:date\_applicability "2022-12-01"^^xsd:date ;
    tlx:openApplicabilityEnd tlx:NonePrescribed ;
    eli:is\_realized\_by <sluzbeni/2022/119/1834/art\_107/v/2022-10-22/hrv> .

<event/test/2022-12-01> a tlx:ChangeEvent ;
    tlx:eventType tlx:Amendment ;
    tlx:date "2022-12-01"^^xsd:date ;
    tlx:causedBy <https://narodne-novine.nn.hr/eli/sluzbeni/2022/119/1834> ;
    tlx:opens <sluzbeni/2022/119/1834/art\_107/v/2022-12-01> ;
    tlx:opensApplicability <sluzbeni/2022/119/1834/art\_107/v/2022-12-01> .
```

## How to run the queries

Any SPARQL 1.1 engine can be used. With rdflib, save the prefixes and one of the queries below in a file `query.rq` and run:

```
pip install rdflib
python -c "from rdflib import Graph; g = Graph(); g.parse('case-study.ttl'); \[print(r) for r in g.query(open('query.rq').read())]"
```

## Conventions

* Both intervals store their upper bound as the last included day: `eli:date\_no\_longer\_in\_force` for legal force and `tlx:dateNoLongerApplicable` for applicability. The paper uses half-open intervals, whose exclusive end is the day after the stored date; an event that closes an interval falls on that day. Queries therefore compare stored upper bounds inclusively (`?t <= ?last`). Article 107 is in force up to and including 31 December 2022, and the event that ends it takes effect on 1 January 2023.
* `tlx:opens` and `tlx:closes` act on legal force; `tlx:opensApplicability` and `tlx:closesApplicability` act on applicability. The end of legal force does not end applicability by itself.
* A start of applicability equal to the start of legal force means that the act sets no separate date of application. An end of applicability is never copied from the end of legal force: each version records its own end of applicability or the reading of its absence.
* The readings `tlx:NonePrescribed`, `tlx:PendingCondition` and `tlx:NotExtracted` are recorded separately for each interval and refer to the sources included in the graph.
* A termination condition states which interval it ends (`tlx:affects`) and from which day it has legal effect (`tlx:effectiveFrom`, from the source). Before that day it does not qualify the status of the version, and query 4 then gives no reading instead of assuming that no end was prescribed.
* A condition may consist of several requirements, all of which must be met. Fulfilment is not detected automatically: it is entered explicitly (`tlx:fulfilledBy`, `tlx:metBy`) after the sources have been checked. The status `tlx:NoRecordedFulfilment` means only that no fulfilment is recorded. `tlx:checkedOn` is given only when a check was actually made.
* An application rule is text. The graph does not decide whether it covers a particular case.
* Text units with `tlx:text` contain the complete text of the provision, with a language tag and `dcterms:source`; the others point to the official source with `tlx:textSource`, and query 1 returns that link.

## Correspondence with ELI

|Paper|RDF|
|-|-|
|LegalWork, Component, TemporalVersion, TextUnit|subclasses of `eli:LegalResource`, `eli:LegalResourceSubdivision`, `eli:LegalResource`, `eli:LegalExpression`|
|title of a LegalWork|`dcterms:title`, because the domain of `eli:title` is `eli:Expression`|
|HAS\_COMPONENT, HAS\_TEMPORAL\_VERSION, HAS\_TEXT\_UNIT|`eli:is\_part\_of` and `eli:is\_member\_of` (inverses of `eli:has\_part` and `eli:has\_member`), `eli:is\_realized\_by`|
|SUPERSEDED\_BY, BASED\_ON|`eli:repealed\_by`, `eli:based\_on`|
|in\_force\_from, in\_force\_to, applies\_from|`eli:first\_date\_entry\_in\_force`, `eli:date\_no\_longer\_in\_force` (last day), `eli:date\_applicability`|
|applies\_to, application\_rule|`tlx:dateNoLongerApplicable` (last day), `tlx:applicationRule`|
|open\_end|`tlx:openEnd` for legal force and `tlx:openApplicabilityEnd` for applicability|
|OPENS, CLOSES|`tlx:opens`, `tlx:closes` for legal force; `tlx:opensApplicability`, `tlx:closesApplicability` for applicability|
|CAUSED\_BY, HAS\_CONDITION, PRESCRIBED\_BY, FULFILLED\_BY|`tlx:causedBy`, `tlx:hasCondition`, `tlx:prescribedBy`, `tlx:fulfilledBy`|
|attributes of a TerminationCondition|`tlx:condition`, `tlx:prescribingProvision`, `tlx:affects`, `tlx:effectiveFrom`, `tlx:status` (`tlx:Fulfilled` or `tlx:NoRecordedFulfilment`, derived from FULFILLED\_BY), `tlx:checkedOn`; its requirements are `tlx:Requirement` nodes|

## Case study

|Act|ELI|In force from|In the graph|
|-|-|-|-|
|Act on Higher Education and Scientific Activity (OG 119/2022)|`.../eli/sluzbeni/2022/119/1834`|22 Oct 2022|Arts. 107, 108, 121(3), 122|
|Act on Scientific Activity and Higher Education (OG 123/2003)|`.../eli/sluzbeni/2003/123/1742`||repealed on 22 Oct 2022|
|Act on Quality Assurance in Higher Education and Science (OG 151/2022)|`.../eli/sluzbeni/2022/151/2330`|30 Dec 2022|Arts. 8(3), 21(3), 50, 51, 52|
|Act on Quality Assurance in Science and Higher Education (OG 45/2009)|`.../eli/sluzbeni/2009/45/1031`||repealed on 30 Dec 2022|
|Regulation on licences for scientific activity (OG 83/2010)|`.../eli/sluzbeni/2010/83/2375`|13 Jul 2010|kept in force by Art. 50 of OG 151/2022|

The introduction of the euro on 1 January 2023 is one event that ends the legal force of Article 107 and opens Article 108. Article 121(3) states when Article 107 ceases to be in force, but not until when it is applied to cases that arose while it was in force, so its end of applicability is recorded as not extracted. The regulation OG 83/2010 remains in force until the quality standards under Article 8(3) and the decision on the form and content of the licence under Article 21(3) of OG 151/2022 are adopted. The condition affects legal force and has effect from 30 December 2022. The graph records it with its two requirements, but no event that meets either of them and no check for such events. The included sources do not state whether the regulation is applied after it ceases to be in force, so its end of applicability is recorded as not extracted.

## Synthetic examples

Part 2 of `case-study.ttl` contains invented acts under `https://w3id.org/telex-kg/id/synthetic/`, marked as synthetic in their descriptions.

* **A. Force and applicability diverge.** Article 5 of Act A has two consecutive versions whose legal force does not overlap. The first ceased to be in force on 31 December 2023, but under a transitional provision of Act B it still applies to proceedings started before 1 January 2024. From that date both versions apply, each with its own rule, while only the second is in force. Article 6 has a version whose ends have not been extracted.
* **B. A condition is fulfilled.** Regulation R is kept in force and applicable by a condition with two requirements. The first is met on 15 June 2021, the second on 1 March 2022. The second event fulfils the condition and closes both intervals of the regulation, whose last included day is 28 February 2022. Only the final state is stored; the earlier state follows from the event dates at query time.

## Queries

All queries use these prefixes:

```
PREFIX eli:  <http://data.europa.eu/eli/ontology#>
PREFIX tlx:  <https://w3id.org/telex-kg/ontology#>
PREFIX xsd:  <http://www.w3.org/2001/XMLSchema#>
PREFIX lang: <http://publications.europa.eu/resource/authority/language/>
```

The parameters are set in the `VALUES` line: the component `?c`, the date `?t` and, in query 1, the language `?l`. In the tables, component and version identifiers are shortened by leaving out `https://w3id.org/telex-kg/id/sluzbeni/`, and `syn:` stands for `https://w3id.org/telex-kg/id/synthetic/`.

### 1\. Which version is in force on a date, with its text or source

Listing 1 in the paper is this query without `?source`.

```
SELECT ?v ?text ?source WHERE {
  VALUES (?c ?t ?l) { (<https://w3id.org/telex-kg/id/sluzbeni/2022/119/1834/art\_107> "2022-12-31"^^xsd:date lang:HRV) }
  ?v eli:is\_member\_of ?c ; eli:first\_date\_entry\_in\_force ?from .
  OPTIONAL { ?v eli:date\_no\_longer\_in\_force ?last }
  FILTER (?from <= ?t \&\& (!BOUND(?last) || ?t <= ?last))
  OPTIONAL { ?v eli:is\_realized\_by ?u . ?u eli:language ?l .
             OPTIONAL { ?u tlx:text ?text }
             OPTIONAL { ?u tlx:textSource ?source } }
}
```

|`?c`|`?t`|Expected result|
|-|-|-|
|`2022/119/1834/art\_107`|2022-10-21|no row: the act is published but not yet in force|
|`2022/119/1834/art\_107`|2022-10-22|`art\_107/v/2022-10-22`, source OG 119/2022|
|`2022/119/1834/art\_107`|2022-12-31|`art\_107/v/2022-10-22`, source OG 119/2022 (last included day)|
|`2022/119/1834/art\_107`|2023-01-01|no row: legal force ended with the introduction of the euro|
|`2022/119/1834/art\_108`|2022-10-22|no row: not yet in force although the act is|
|`2022/119/1834/art\_108`|2022-12-31|no row|
|`2022/119/1834/art\_108`|2023-01-01|`art\_108/v/2023-01-01`, source OG 119/2022|
|`2022/119/1834/art\_121/par\_3`|2022-12-31|`art\_121/par\_3/v/2022-10-22`, text „Odredbe članka 107. ovoga Zakona prestaju važiti na dan uvođenja eura kao službene valute u Republici Hrvatskoj.“@hr|
|`2022/151/2330/art\_50`|2023-01-01|`art\_50/v/2022-12-30`, complete text of Article 50 (@hr)|
|`2010/83/2375/whole`|2022-12-30|`whole/v/2010-07-13`, source OG 83/2010|
|`syn:act-a/art\_5` (with `lang:ENG`)|2025-06-01|only `syn:act-a/art\_5/v/2024-01-01`, text „An application shall be decided within 30 days.“@en|

### 2\. Which versions apply on a date

```
SELECT ?v ?rule ?last ?openApplicabilityEnd WHERE {
  VALUES (?c ?t) { (<https://w3id.org/telex-kg/id/synthetic/act-a/art\_5> "2025-06-01"^^xsd:date) }
  ?v eli:is\_member\_of ?c ; eli:date\_applicability ?from .
  OPTIONAL { ?v tlx:dateNoLongerApplicable ?last }
  FILTER (?from <= ?t \&\& (!BOUND(?last) || ?t <= ?last))
  OPTIONAL { ?v tlx:applicationRule ?rule }
  OPTIONAL { ?v tlx:openApplicabilityEnd ?openApplicabilityEnd }
}
```

|`?c`|`?t`|Expected result|
|-|-|-|
|`syn:act-a/art\_5`|2023-06-01|`v/2020-01-01` with its rule|
|`syn:act-a/art\_5`|2025-06-01|`v/2020-01-01` (proceedings started before 1 January 2024) and `v/2024-01-01` (proceedings started on or after that date), both with none\_prescribed|
|`2022/119/1834/art\_107`|2023-06-01|`art\_107/v/2022-10-22` with not\_extracted: the end of applicability is unknown, so the answer is qualified|
|`syn:regulation-r/whole`|2022-02-28|`v/2015-02-01`, last day applicable 2022-02-28|
|`syn:regulation-r/whole`|2022-03-01|no row: applicability ended with the fulfilment of the condition|

On 2025-06-01 query 1 returns only the second version of `syn:act-a/art\_5`: two versions apply, but only one is in force. The application rules are returned as text; the query does not decide which of them covers a given case.

### 3\. History of a provision

```
SELECT ?v ?from ?last ?opening ?closing ?appFrom ?appLast ?openApplicabilityEnd WHERE {
  VALUES (?c) { (<https://w3id.org/telex-kg/id/sluzbeni/2022/119/1834/art\_107>) }
  ?v eli:is\_member\_of ?c ; eli:first\_date\_entry\_in\_force ?from ; eli:date\_applicability ?appFrom .
  OPTIONAL { ?v eli:date\_no\_longer\_in\_force ?last }
  OPTIONAL { ?v tlx:dateNoLongerApplicable ?appLast }
  OPTIONAL { ?v tlx:openApplicabilityEnd ?openApplicabilityEnd }
  OPTIONAL { ?opening tlx:opens ?v }
  OPTIONAL { ?closing tlx:closes ?v }
} ORDER BY ?from
```

|`?c`|Expected result|
|-|-|
|`2022/119/1834/art\_107`|one version: in force 2022-10-22 to 2022-12-31, opened by the entry into force of the act and closed by the introduction of the euro; applicable from 2022-10-22, end of applicability not extracted|
|`syn:act-a/art\_5`|two versions: the first in force 2020-01-01 to 2023-12-31, closed by the amendment, still applicable with none\_prescribed; the second from 2024-01-01, opened by the amendment|

### 4\. Qualified status of legal force on a date

A condition qualifies the answer only from its `tlx:effectiveFrom`. Before that day the query returns no reading and says why, instead of assuming that no end was prescribed.

```
SELECT DISTINCT ?v ?last ?readingAtT ?qualification ?cond ?effectiveFrom ?fulfilledOn ?checkedOn WHERE {
  VALUES (?c ?t) { (<https://w3id.org/telex-kg/id/sluzbeni/2010/83/2375/whole> "2011-01-01"^^xsd:date) }
  ?v eli:is\_member\_of ?c ; eli:first\_date\_entry\_in\_force ?from .
  OPTIONAL { ?v eli:date\_no\_longer\_in\_force ?last }
  FILTER (?from <= ?t \&\& (!BOUND(?last) || ?t <= ?last))
  OPTIONAL { ?v tlx:openEnd ?openEnd }
  OPTIONAL { ?v tlx:hasCondition ?cond .
             ?cond tlx:affects tlx:LegalForce ; tlx:effectiveFrom ?effectiveFrom .
             OPTIONAL { ?cond tlx:fulfilledBy/tlx:date ?fulfilledOn }
             OPTIONAL { ?cond tlx:checkedOn ?checkedOn } }
  BIND (BOUND(?effectiveFrom) \&\& ?effectiveFrom <= ?t
        \&\& (!BOUND(?fulfilledOn) || ?t < ?fulfilledOn) AS ?inEffect)
  BIND (IF(?inEffect, tlx:PendingCondition,
        IF(BOUND(?last) || ?openEnd = tlx:PendingCondition, ?noValue, ?openEnd)) AS ?readingAtT)
  BIND (IF(?inEffect, "The end of legal force depends on a termination condition in effect on this date; no fulfilment on or before this date is recorded.",
        IF(BOUND(?last), "The last day in force is recorded.",
        IF(?openEnd = tlx:PendingCondition, "No termination condition in effect on this date is recorded. The recorded condition takes effect later, and the graph does not record how the end of legal force was determined before that date.",
        IF(?openEnd = tlx:NonePrescribed, "None of the included sources provides for an end of legal force.",
           "The end of legal force has not been extracted from the sources.")))) AS ?qualification)
}
```

|`?c`|`?t`|Expected result|
|-|-|-|
|`2010/83/2375/whole`|2011-01-01|in force; no reading; qualification: no condition in effect on this date is recorded; condition with effective date 2022-12-30|
|`2010/83/2375/whole`|2022-12-29|the same as for 2011-01-01|
|`2010/83/2375/whole`|2022-12-30|in force; pending\_condition; condition in effect; no fulfilment and no check recorded|
|`2010/83/2375/whole`|2026-09-27|the same as for 2022-12-30|
|`2022/119/1834/art\_108`|2023-06-01|in force; none\_prescribed|
|`syn:regulation-r/whole`|2020-06-01|in force; last day in force 2022-02-28 recorded|
|`syn:regulation-r/whole`|2021-09-01|in force; pending\_condition on that date; the condition was fulfilled on 2022-03-01|
|`syn:regulation-r/whole`|2022-03-01|no row: no longer in force|
|`syn:act-a/art\_6`|2021-01-01|in force; not\_extracted|

For 27 September 2026 the answer is therefore qualified: the regulation is in force unless both instruments have been adopted, and no check for their adoption is recorded.

### 5\. Requirements of a condition on a date

```
SELECT ?req ?under ?metOn ?metByT WHERE {
  VALUES (?cond ?t) { (<https://w3id.org/telex-kg/id/condition/sluzbeni/2022/151/2330/art\_50/sluzbeni/2010/83/2375> "2026-09-27"^^xsd:date) }
  ?cond tlx:hasRequirement ?req .
  OPTIONAL { ?req tlx:adoptionUnder ?under }
  OPTIONAL { ?req tlx:metBy/tlx:date ?metOn }
  BIND (BOUND(?metOn) \&\& ?metOn <= ?t AS ?metByT)
} ORDER BY ?req
```

|`?cond`|`?t`|Expected result|
|-|-|-|
|condition of Art. 50 for OG 83/2010|2026-09-27|two requirements (Art. 8(3) and Art. 21(3)), neither with a recorded event|
|`syn:condition/act-d/art\_30/regulation-r`|2021-09-01|requirement 1 met on 2021-06-15; requirement 2 not yet met|
|`syn:condition/act-d/art\_30/regulation-r`|2022-03-01|both requirements met|

The query reports only what has been entered. Whether the instruments have been adopted in fact is established by checking the sources and entering the events.

## ChatGPT question and responses, 28 August 2026 (Section 5.2)

ChatGPT Free (OpenAI), 28 August 2026. The question was posed in Croatian.

### Question

```text
Je li Pravilnik o uvjetima za izdavanje dopusnice za obavljanje znanstvene djelatnosti, uvjetima za reakreditaciju znanstvenih organizacija i sadržaju dopusnice (NN 83/10) danas na snazi, s obzirom na to da je Zakon o osiguravanju kvalitete u znanosti i visokom obrazovanju (NN 45/09), na temelju kojeg je donesen, prestao važiti?
```

### Response 1: without supporting documents

```text
Da, Pravilnik o uvjetima za izdavanje dopusnice za obavljanje znanstvene djelatnosti, uvjetima za reakreditaciju znanstvenih organizacija i sadržaju dopusnice (NN 83/10) i danas je na snazi, iako je Zakon o osiguravanju kvalitete u znanosti i visokom obrazovanju (NN 45/09), na temelju kojeg je donesen, prestao važiti 1. siječnja 2023. Razlog je prijelazna odredba Zakona o osiguravanju kvalitete u visokom obrazovanju i znanosti (NN 151/22). Tim je zakonom propisano da:
•	pravilnik iz NN 83/10 ostaje na snazi do donošenja:
o	Standarda kvalitete iz članka 8. stavka 3. novog Zakona i
o	odluke o obliku i detaljnijem sadržaju dopusnice iz članka 21. stavka 3. novog Zakona.
Stoga prestanak važenja zakona koji je bio pravna osnova za donošenje pravilnika nije doveo do prestanka važenja pravilnika, jer je zakonodavac njegovu daljnju primjenu izričito osigurao prijelaznom odredbom novog zakona. Zaključak: Pravilnik NN 83/10 i danas je na snazi te se primjenjuje kao prijelazni provedbeni propis sve dok ne budu doneseni novi provedbeni akti predviđeni Zakonom o osiguravanju kvalitete u visokom obrazovanju i znanosti (NN 151/22).
```

### Response 2: with the two acts supplied as documents

```text
Kratki odgovor je: da, vrlo vjerojatno jest, iako je njegovo pravno uporište danas drukčije nego u trenutku donošenja.
Objašnjenje zahtijeva primjenu općih pravila o kontinuitetu podzakonskih propisa.
Pravilnik iz 2010. godine izričito je donesen na temelju članka 26. stavka 2. tadašnjeg Zakona o osiguravanju kvalitete u znanosti i visokom obrazovanju (NN 45/09). To je jasno navedeno u preambuli Pravilnika.
Međutim, taj je Zakon 2022. godine stavljen izvan snage donošenjem novog Zakona o osiguravanju kvalitete u visokom obrazovanju i znanosti (NN 151/22). Novi zakon uređuje isto područje – sustav osiguravanja kvalitete, inicijalnu akreditaciju, reakreditaciju i izdavanje dopusnica – ali na drukčiji način i s drukčijom normativnom strukturom.
Ključno pravno pitanje glasi:
Prestaje li automatski važiti pravilnik kada prestane važiti zakon na temelju kojeg je donesen?
Odgovor je ne nužno.
U hrvatskom pravnom sustavu vrijedi načelo da podzakonski propis ne prestaje automatski važiti samim prestankom važenja zakona na temelju kojeg je donesen, osim ako:
1.	novi zakon izričito propiše da prestaju važiti provedbeni propisi,
2.	novi zakon drukčije uredi isto pitanje tako da pravilnik više nije primjenjiv,
3.	ili je pravilnik u suprotnosti s novim zakonom.
Ako ništa od toga nije ispunjeno, pravilnik ostaje na snazi u dijelu u kojem nije protivan novom zakonu, sve dok ne bude izmijenjen ili stavljen izvan snage.
U ovom slučaju novi Zakon iz 2022.:
•	i dalje poznaje institut inicijalne akreditacije,
•	i dalje predviđa izdavanje dopusnica,
•	i dalje predviđa donošenje provedbenih propisa,
•	ali uvodi drukčije kriterije i drukčiju koncepciju vrednovanja, osobito kroz Standarde kvalitete koje donosi AZVO.
To znači da je potrebno razlikovati dvije razine:
•	formalnu valjanost Pravilnika – koja nije automatski prestala,
•	materijalnu primjenjivost pojedinih njegovih odredbi – koja može biti ograničena ako su u koliziji s novim Zakonom.
Primjerice, Pravilnik propisuje minimalan broj znanstvenika, sadržaj elaborata, uvjete prostora i opreme te postupak reakreditacije. Novi Zakon iz 2022. dio tih pitanja uređuje drukčije (npr. samoanalizom prema Standardima kvalitete, novim kriterijima za inicijalnu akreditaciju i drukčijim zahtjevima za znanstvene institute).
Zaključak
Po mom mišljenju, koje je u skladu s općim pravilima hrvatskog upravnog prava:
•	Pravilnik NN 83/10 nije automatski prestao važiti samo zato što je prestao važiti Zakon NN 45/09.
•	On se može primjenjivati samo u mjeri u kojoj nije u suprotnosti sa Zakonom o osiguravanju kvalitete u visokom obrazovanju i znanosti (NN 151/22).
•	Ako postoji nesklad između Pravilnika i novog Zakona, prednost ima Zakon, a nesuglasne odredbe Pravilnika ne mogu biti pravna osnova za odlučivanje.
To je upravo razlog zbog kojeg je danas normativna situacija pomalo neuredna: formalno još postoji pravilnik iz 2010., ali dio njegovih rješenja više nije potpuno usklađen s novim zakonskim okvirom.
Mislim da je ovo vrlo zanimljivo pitanje za pravnu analizu jer se ovdje susreću načelo kontinuiteta podzakonskih propisa, načelo zakonitosti i problem tzv. "preživljavanja" provedbenih propisa nakon izmjene ili ukidanja zakonske osnove. Upravo oko toga postoji dosta upravnopravne i ustavnopravne prakse koja može dodatno razjasniti granice primjene takvih pravilnika.
```

### English translation

Question:

```text
Is the Regulation on the Conditions for Issuing a Licence for Scientific Activity, the Conditions for the Re-accreditation of Scientific Organisations and the Content of the Licence (Official Gazette No. 83/10) still in force today, given that the Act on Quality Assurance in Science and Higher Education (Official Gazette No. 45/09), under which it was adopted, has ceased to be in force?
```

Response 1:

```text
Yes, the Regulation on the Conditions for Issuing a Licence for Scientific Activity, the Conditions for the Re-accreditation of Scientific Organisations and the Content of the Licence (OG 83/10) is still in force today, although the Act on Quality Assurance in Science and Higher Education (OG 45/09), under which it was adopted, ceased to be in force on 1 January 2023. The reason is the transitional provision of the Act on Quality Assurance in Higher Education and Science (OG 151/22). That act provides that:
•	the regulation from OG 83/10 remains in force until the adoption of:
o	the Quality Standards under Article 8(3) of the new Act and
o	the decision on the form and detailed content of the licence under Article 21(3) of the new Act.
Therefore, the cessation of the act that was the legal basis for adopting the regulation did not lead to the cessation of the regulation, because the legislator expressly ensured its continued application through the transitional provision of the new act. Conclusion: Regulation OG 83/10 is still in force today and is applied as a transitional implementing regulation until the new implementing acts provided for by the Act on Quality Assurance in Higher Education and Science (OG 151/22) are adopted.
```

Response 2:

```text
The short answer is: yes, very likely it is, although its legal basis today is different from what it was when it was adopted.
The explanation requires applying the general rules on the continuity of subordinate legislation.
The 2010 Regulation was expressly adopted on the basis of Article 26(2) of the then Act on Quality Assurance in Science and Higher Education (OG 45/09). This is clearly stated in the preamble of the Regulation.
However, that Act was repealed in 2022 by the adoption of the new Act on Quality Assurance in Higher Education and Science (OG 151/22). The new act regulates the same area (the quality assurance system, initial accreditation, reaccreditation and the issuing of licences), but in a different way and with a different normative structure.
The key legal question is:
Does a regulation automatically cease to be in force when the act on the basis of which it was adopted ceases to be in force?
The answer is: not necessarily.
In the Croatian legal system, the principle applies that a subordinate regulation does not automatically cease to be in force merely because the act on the basis of which it was adopted has ceased to be in force, unless:
1.	the new act expressly provides that the implementing regulations cease to be in force,
2.	the new act regulates the same matter differently, so that the regulation is no longer applicable,
3.	or the regulation is contrary to the new act.
If none of this is the case, the regulation remains in force to the extent that it is not contrary to the new act, until it is amended or repealed.
In this case, the new 2022 Act:
•	still provides for initial accreditation,
•	still provides for the issuing of licences,
•	still provides for the adoption of implementing regulations,
•	but introduces different criteria and a different concept of evaluation, in particular through the Quality Standards adopted by AZVO.
This means that two levels must be distinguished:
•	the formal validity of the Regulation, which has not automatically ceased,
•	the material applicability of its individual provisions, which may be limited if they conflict with the new Act.
For example, the Regulation prescribes the minimum number of scientists, the content of the supporting study, the requirements for premises and equipment, and the reaccreditation procedure. The new 2022 Act regulates some of these matters differently (for example, through self-evaluation according to the Quality Standards, new criteria for initial accreditation and different requirements for scientific institutes).
Conclusion
In my opinion, which is consistent with the general rules of Croatian administrative law:
•	Regulation OG 83/10 did not automatically cease to be in force merely because Act OG 45/09 ceased to be in force.
•	It can be applied only to the extent that it is not contrary to the Act on Quality Assurance in Higher Education and Science (OG 151/22).
•	If there is a conflict between the Regulation and the new Act, the Act prevails, and the conflicting provisions of the Regulation cannot serve as a legal basis for decision-making.
This is precisely why the normative situation today is somewhat untidy: formally, the 2010 regulation still exists, but some of its solutions are no longer fully aligned with the new legal framework.
I think this is a very interesting question for legal analysis, because it brings together the principle of continuity of subordinate legislation, the principle of legality and the problem of the so-called "survival" of implementing regulations after their legal basis has been amended or repealed. There is a considerable body of administrative and constitutional case law on this that can further clarify the limits of applying such regulations.
```

## Additional runs, 27 September 2026 (GPT-5.5)

The question of Section 5.2 was posed again on 27 September 2026 in ChatGPT, with the model GPT-5.5 and the reasoning level set to high, as shown in the interface. Each setting was run once, in a new conversation. Web search could not be switched off in the interface, so the first line of every prompt asks the model not to search the web. The settings differ only in what the model was given with the question:

1. nothing;
2. the two acts as PDF files, as published in the Official Gazette: OG 83/2010 and OG 151/2022;
3a. the data that the graph records for the regulation;
3b. the same data, with one sentence on the meaning of the recorded status.

An earlier run of setting 3a, made without the instruction not to search the web, gave an answer of the same kind; it is reproduced at the end. These are single runs. They illustrate how the answer depends on what the model is given and are not an evaluation.

|Setting|Given with the question|Conclusion of the response|
|-|-|-|
|1|nothing|describes a generic transitional rule; the regulation "may still be considered valid" until a new regulation replaces it|
|2|OG 83/2010 and OG 151/2022 (PDF)|identifies Article 50 and both requirements; whether the regulation is in force today depends on whether both instruments have been adopted, which has to be checked|
|3a|data recorded in the graph|identifies Article 50 and both requirements; reads the absence of a recorded adoption as evidence that the regulation is still in force|
|3b|the same data and the meaning of the status|identifies Article 50 and both requirements; the current status cannot be confirmed from the data and has to be checked|

### Prompts

Settings 1 and 2 (in setting 2 with the two PDF files attached):

```text
Nemoj pretraživati web.

Je li Pravilnik o uvjetima za izdavanje dopusnice za obavljanje znanstvene djelatnosti, uvjetima za reakreditaciju znanstvenih organizacija i sadržaju dopusnice (NN 83/10) danas na snazi, s obzirom na to da je Zakon o osiguravanju kvalitete u znanosti i visokom obrazovanju (NN 45/09), na temelju kojeg je donesen, prestao važiti?
```

Setting 3a:

```text
Nemoj pretraživati web.

Za odgovor na pitanje na raspolaganju su ti sljedeći podaci iz baze pravnih propisa.

PODACI IZ BAZE

1. Pravilnik o uvjetima za izdavanje dopusnice za obavljanje znanstvene djelatnosti, uvjetima za reakreditaciju znanstvenih organizacija i sadržaju dopusnice (NN 83/2010) objavljen je 5. srpnja 2010. i stupio je na snagu 13. srpnja 2010. Donesen je na temelju članka 26. stavka 2. Zakona o osiguravanju kvalitete u znanosti i visokom obrazovanju (NN 45/2009).

2. Zakon o osiguravanju kvalitete u visokom obrazovanju i znanosti (NN 151/2022) objavljen je 22. prosinca 2022. i stupio je na snagu 30. prosinca 2022. Toga je dana prestao važiti Zakon NN 45/2009 (članak 51. Zakona NN 151/2022).

3. Kraj pravne snage Pravilnika nije zabilježen kao datum. Ovisi o uvjetu iz članka 50. Zakona NN 151/2022, koji glasi: „Pravilnik o sadržaju dopusnice te uvjetima za izdavanje dopusnice za obavljanje djelatnosti visokog obrazovanja, izvođenje studijskog programa i reakreditaciju visokih učilišta (»Narodne novine«, br. 24/10.) i Pravilnik o uvjetima za izdavanje dopusnice za obavljanje znanstvene djelatnosti, uvjetima za reakreditaciju znanstvenih organizacija i sadržaju dopusnice (»Narodne novine«, br. 83/10.), doneseni na temelju Zakona o osiguravanju kvalitete u znanosti i visokom obrazovanju (»Narodne novine«, br. 45/09.), ostaju na snazi do donošenja Standarda kvalitete iz članka 8. stavka 3. ovoga Zakona te odluke o obliku i detaljnijem sadržaju dopusnice iz članka 21. stavka 3. ovoga Zakona.“

4. Taj se uvjet odnosi na pravnu snagu Pravilnika i proizvodi učinak od 30. prosinca 2022. Ispunjen je tek kada su ispunjena oba zahtjeva: (a) donošenje Standarda kvalitete iz članka 8. stavka 3. Zakona NN 151/2022 i (b) donošenje odluke o obliku i detaljnijem sadržaju dopusnice iz članka 21. stavka 3. Zakona NN 151/2022.

5. U bazi nije zabilježeno donošenje nijednog od tih dvaju akata, niti je zabilježena provjera izvora o njihovu donošenju.

6. Kraj primjenjivosti Pravilnika nije utvrđen: izvori u bazi ne navode primjenjuje li se Pravilnik nakon prestanka njegove pravne snage.

PITANJE

Je li Pravilnik o uvjetima za izdavanje dopusnice za obavljanje znanstvene djelatnosti, uvjetima za reakreditaciju znanstvenih organizacija i sadržaju dopusnice (NN 83/10) danas na snazi, s obzirom na to da je Zakon o osiguravanju kvalitete u znanosti i visokom obrazovanju (NN 45/09), na temelju kojeg je donesen, prestao važiti?
```

Setting 3b: the same as setting 3a, except that point 5 reads:

```text
5. U bazi nije zabilježeno donošenje nijednog od tih dvaju akata, niti je zabilježena provjera izvora o njihovu donošenju. To ne znači da akti nisu doneseni, nego samo da njihovo donošenje u bazi nije provjereno.
```

English translation of the prompt of settings 3a and 3b:

```text
Do not search the web.

To answer the question, you have the following data from a database of legal regulations.

DATA FROM THE DATABASE

1. The Regulation on the Conditions for Issuing a Licence for Scientific Activity, the Conditions for the Re-accreditation of Scientific Organisations and the Content of the Licence (OG 83/2010) was published on 5 July 2010 and entered into force on 13 July 2010. It was adopted under Article 26(2) of the Act on Quality Assurance in Science and Higher Education (OG 45/2009).

2. The Act on Quality Assurance in Higher Education and Science (OG 151/2022) was published on 22 December 2022 and entered into force on 30 December 2022. On that day Act OG 45/2009 ceased to be in force (Article 51 of Act OG 151/2022).

3. The end of the legal force of the Regulation is not recorded as a date. It depends on the condition in Article 50 of Act OG 151/2022, which reads: "The Regulation on the content of the licence and the conditions for issuing a licence for higher education activity, for delivering a study programme and for the re-accreditation of higher education institutions (Official Gazette No. 24/10) and the Regulation on the conditions for issuing a licence for scientific activity, the conditions for the re-accreditation of scientific organisations and the content of the licence (Official Gazette No. 83/10), adopted under the Act on Quality Assurance in Science and Higher Education (Official Gazette No. 45/09), shall remain in force until the adoption of the Quality Standards referred to in Article 8(3) of this Act and of the decision on the form and detailed content of the licence referred to in Article 21(3) of this Act."

4. This condition concerns the legal force of the Regulation and takes effect from 30 December 2022. It is fulfilled only when both requirements are met: (a) the adoption of the Quality Standards under Article 8(3) of Act OG 151/2022 and (b) the adoption of the decision on the form and detailed content of the licence under Article 21(3) of Act OG 151/2022.

5. The database records the adoption of neither of these two acts, nor does it record a check of the sources for their adoption.
   \[Setting 3b adds:] This does not mean that the acts have not been adopted, only that their adoption has not been checked in the database.

6. The end of the applicability of the Regulation has not been established: the sources in the database do not state whether the Regulation is applied after its legal force ends.

QUESTION

Is the Regulation on the Conditions for Issuing a Licence for Scientific Activity, the Conditions for the Re-accreditation of Scientific Organisations and the Content of the Licence (Official Gazette No. 83/10) still in force today, given that the Act on Quality Assurance in Science and Higher Education (Official Gazette No. 45/09), under which it was adopted, has ceased to be in force?
```

### Responses (in Croatian, unedited)

Setting 1:

```text
Ne može se zaključiti da je Pravilnik NN 83/10 automatski prestao važiti samo zato što je prestao važiti Zakon NN 45/09 na temelju kojeg je donesen.
U pravilu, podzakonski propisi doneseni na temelju starog zakona mogu ostati na snazi i nakon prestanka tog zakona ako novi zakon to dopušta prijelaznim odredbama, najčešće formulacijom tipa: „propisi doneseni na temelju ranijeg zakona ostaju na snazi do donošenja novih propisa, ako nisu u suprotnosti s novim zakonom”.
Dakle, odgovor bi bio:
Da, Pravilnik NN 83/10 može se i dalje smatrati važećim, ali samo u mjeri u kojoj nije u suprotnosti s novim Zakonom NN 151/22 i dok nije donesen novi pravilnik koji ga zamjenjuje. Sama činjenica da je prestao važiti Zakon NN 45/09 nije dovoljna da bi Pravilnik automatski prestao važiti.
Za konačnu formulaciju u pravnom tekstu napisala bih oprezno:
Pravilnik NN 83/10 nije prestao važiti samim prestankom važenja Zakona NN 45/09 na temelju kojega je donesen. Kao podzakonski propis može se nastaviti primjenjivati na temelju prijelaznih odredaba novog Zakona NN 151/22, do donošenja novog provedbenog propisa, i to u dijelu u kojem nije u suprotnosti s novim zakonskim uređenjem.
```

Setting 2:

```text
Ne prestaje automatski važiti samo zato što je prestao važiti zakon NN 45/09 na temelju kojeg je donesen.
U priloženom Zakonu NN 151/22 postoji izričita prijelazna odredba: članak 50. navodi upravo taj Pravilnik NN 83/10 i kaže da on ostaje na snazi do donošenja Standarda kvalitete iz čl. 8. st. 3. i odluke o obliku i detaljnijem sadržaju dopusnice iz čl. 21. st. 3. Tek nakon toga prestaje važiti po toj prijelaznoj osnovi.
Dakle, pravni zaključak je:
Pravilnik NN 83/10 nije prestao važiti samim prestankom važenja Zakona NN 45/09. Novi zakon ga je izričito zadržao na snazi prijelazno. Za odgovor je li baš danas još na snazi treba provjeriti jesu li u međuvremenu donesena oba akta iz članka 50. Bez web-pretraživanja i samo iz ova dva PDF-a mogu sigurno reći: prestao bi vrijediti tek ako su ti novi akti doneseni; inače je i dalje prijelazno na snazi.
```

Setting 3a:

```text
Da, prema podacima koje navodi baza, Pravilnik NN 83/10 treba smatrati i dalje na snazi, odnosno njegov prestanak pravne snage nije utvrđen.
Ključ je u tome da prestanak važenja Zakona NN 45/09 nije sam po sebi ukinuo Pravilnik. Novi Zakon NN 151/2022 u članku 50. izričito propisuje da taj Pravilnik ostaje na snazi do donošenja dvaju novih akata:
1. Standarda kvalitete iz članka 8. stavka 3. Zakona NN 151/2022, i
2. odluke o obliku i detaljnijem sadržaju dopusnice iz članka 21. stavka 3. istog Zakona.
Budući da u dostavljenim podacima nije zabilježeno donošenje nijednog od tih dvaju akata, a kraj pravne snage Pravilnika nije evidentiran datumom, iz raspoloživih podataka proizlazi da Pravilnik NN 83/10 nije prestao važiti.
Precizna formulacija bila bi:
Pravilnik NN 83/10 nije prestao važiti samo zato što je prestao važiti Zakon NN 45/09 na temelju kojeg je donesen. Na temelju prijelazne odredbe članka 50. Zakona NN 151/2022, Pravilnik ostaje na snazi do donošenja propisanih novih akata. Kako u bazi nije evidentirano da su ti akti doneseni, prema dostupnim podacima Pravilnik je i dalje na snazi.
```

Setting 3b:

```text
Prema podacima koje si dala: prestanak važenja Zakona NN 45/09 sam po sebi nije ugasio Pravilnik NN 83/10.
Razlog je prijelazna odredba članka 50. Zakona NN 151/2022, koja izričito propisuje da taj Pravilnik, iako je donesen na temelju starog Zakona NN 45/09, ostaje na snazi do ispunjenja posebnog uvjeta: donošenja oba nova akta:
1. Standarda kvalitete iz članka 8. stavka 3. Zakona NN 151/2022
2. odluke o obliku i detaljnijem sadržaju dopusnice iz članka 21. stavka 3. Zakona NN 151/2022
Dakle, pravni zaključak je:
Pravilnik NN 83/10 nije prestao važiti automatski 30. prosinca 2022. kada je prestao važiti Zakon NN 45/09. Ostaje na snazi na temelju članka 50. Zakona NN 151/2022, sve dok ne budu donesena oba navedena nova akta.
Međutim, iz same baze koju navodiš ne može se konačno potvrditi je li Pravilnik danas još uvijek na snazi, jer baza ne sadrži provjeru jesu li ta dva nova akta u međuvremenu donesena. Zato bi najprecizniji odgovor bio:
Na temelju raspoloživih podataka nema osnove zaključiti da je Pravilnik NN 83/10 prestao važiti; naprotiv, prema članku 50. Zakona NN 151/2022 on ostaje na snazi dok se ne donesu oba propisana nova akta. Za konačnu potvrdu današnjeg statusa treba provjeriti jesu li ti akti doneseni.
```

Setting 3a, earlier run without the instruction not to search the web:

```text
Da, prema podacima koje navodiš, Pravilnik NN 83/10 treba smatrati i dalje na snazi, barem dok se ne utvrdi da su donesena oba akta iz članka 50. Zakona NN 151/2022.
Ključ je u tome da prestanak važenja Zakona NN 45/09 nije automatski ukinuo pravilnike donesene na njegovoj osnovi. Novi Zakon NN 151/2022 u članku 50. izričito propisuje da Pravilnik NN 83/10 ostaje na snazi do donošenja:
1. Standarda kvalitete iz članka 8. stavka 3. Zakona NN 151/2022, i
2. odluke o obliku i detaljnijem sadržaju dopusnice iz članka 21. stavka 3. istog Zakona.
Budući da baza ne bilježi donošenje tih akata niti datum prestanka pravne snage Pravilnika, iz danih podataka proizlazi zaključak:
Pravilnik NN 83/10 nije prestao važiti samim prestankom Zakona NN 45/09, nego je prijelaznom odredbom novog Zakona NN 151/2022 zadržan na snazi do donošenja novih provedbenih akata. Kako donošenje tih akata nije zabilježeno, prema raspoloživim podacima Pravilnik se i dalje smatra važećim.
Precizna formulacija za pravnu analizu:
Pravilnik NN 83/10 ostao je na snazi na temelju članka 50. Zakona NN 151/2022. Prestanak važenja Zakona NN 45/09 ne dovodi sam po sebi do prestanka važenja Pravilnika, jer je novi zakon izričito uredio njegov prijelazni status. Njegov prestanak ovisi o kumulativnom ispunjenju uvjeta iz članka 50., odnosno donošenju Standarda kvalitete i odluke o obliku i detaljnijem sadržaju dopusnice.
```

### English translations

Setting 1:

```text
It cannot be concluded that Regulation OG 83/10 automatically ceased to be in force merely because Act OG 45/09, under which it was adopted, ceased to be in force.
As a rule, subordinate regulations adopted under an old act may remain in force after that act has ceased to be in force if the new act allows this through transitional provisions, most often with a formulation such as: "regulations adopted under the former act remain in force until new regulations are adopted, unless they are contrary to the new act".
So the answer would be:
Yes, Regulation OG 83/10 can still be considered valid, but only to the extent that it is not contrary to the new Act OG 151/22 and until a new regulation replacing it is adopted. The mere fact that Act OG 45/09 ceased to be in force is not sufficient for the Regulation to cease to be in force automatically.
For the final wording in a legal text, I would write cautiously:
Regulation OG 83/10 did not cease to be in force merely because Act OG 45/09, under which it was adopted, ceased to be in force. As a subordinate regulation, it may continue to be applied on the basis of the transitional provisions of the new Act OG 151/22, until a new implementing regulation is adopted, and to the extent that it is not contrary to the new statutory framework.
```

Setting 2:

```text
It does not automatically cease to be in force merely because Act OG 45/09, under which it was adopted, ceased to be in force.
The attached Act OG 151/22 contains an express transitional provision: Article 50 names this very Regulation OG 83/10 and states that it remains in force until the adoption of the Quality Standards under Article 8(3) and of the decision on the form and detailed content of the licence under Article 21(3). Only after that does it cease to be in force on this transitional basis.
So the legal conclusion is:
Regulation OG 83/10 did not cease to be in force merely because Act OG 45/09 ceased to be in force. The new act expressly kept it in force on a transitional basis. To answer whether it is still in force today, one has to check whether both acts under Article 50 have been adopted in the meantime. Without web search and from these two PDFs alone, I can say with certainty: it would cease to be in force only if these new acts have been adopted; otherwise it remains in force on a transitional basis.
```

Setting 3a:

```text
Yes, according to the data provided by the database, Regulation OG 83/10 should be considered still in force, that is, the end of its legal force has not been established.
The key point is that the cessation of Act OG 45/09 did not by itself repeal the Regulation. The new Act OG 151/2022 expressly provides in Article 50 that this Regulation remains in force until the adoption of two new acts:
1. the Quality Standards under Article 8(3) of Act OG 151/2022, and
2. the decision on the form and detailed content of the licence under Article 21(3) of the same Act.
Since the data supplied record the adoption of neither of these two acts, and the end of the legal force of the Regulation is not recorded as a date, it follows from the available data that Regulation OG 83/10 has not ceased to be in force.
A precise wording would be:
Regulation OG 83/10 did not cease to be in force merely because Act OG 45/09, under which it was adopted, ceased to be in force. On the basis of the transitional provision of Article 50 of Act OG 151/2022, the Regulation remains in force until the prescribed new acts are adopted. Since the database does not record that these acts have been adopted, according to the available data the Regulation is still in force.
```

Setting 3b:

```text
According to the data you have given: the cessation of Act OG 45/09 did not by itself extinguish Regulation OG 83/10.
The reason is the transitional provision of Article 50 of Act OG 151/2022, which expressly provides that this Regulation, although adopted under the old Act OG 45/09, remains in force until a special condition is met: the adoption of both new acts:
1. the Quality Standards under Article 8(3) of Act OG 151/2022
2. the decision on the form and detailed content of the licence under Article 21(3) of Act OG 151/2022
So the legal conclusion is:
Regulation OG 83/10 did not cease to be in force automatically on 30 December 2022, when Act OG 45/09 ceased to be in force. It remains in force on the basis of Article 50 of Act OG 151/2022 until both of these new acts are adopted.
However, the database you refer to does not by itself allow a definite confirmation of whether the Regulation is still in force today, because the database does not contain a check of whether these two new acts have been adopted in the meantime. The most precise answer would therefore be:
On the basis of the available data, there are no grounds to conclude that Regulation OG 83/10 has ceased to be in force; on the contrary, under Article 50 of Act OG 151/2022 it remains in force until both prescribed new acts are adopted. For a definite confirmation of its current status, it has to be checked whether these acts have been adopted.
```

Setting 3a, earlier run without the instruction not to search the web:

```text
Yes, according to the data you provide, Regulation OG 83/10 should be considered still in force, at least until it is established that both acts under Article 50 of Act OG 151/2022 have been adopted.
The key point is that the cessation of Act OG 45/09 did not automatically repeal the regulations adopted under it. The new Act OG 151/2022 expressly provides in Article 50 that Regulation OG 83/10 remains in force until the adoption of:
1. the Quality Standards under Article 8(3) of Act OG 151/2022, and
2. the decision on the form and detailed content of the licence under Article 21(3) of the same Act.
Since the database records neither the adoption of these acts nor the date on which the legal force of the Regulation ended, the following conclusion follows from the data given:
Regulation OG 83/10 did not cease to be in force merely because Act OG 45/09 ceased to be in force, but was kept in force by the transitional provision of the new Act OG 151/2022 until the new implementing acts are adopted. Since the adoption of these acts is not recorded, according to the available data the Regulation is still considered valid.
A precise wording for a legal analysis:
Regulation OG 83/10 remained in force on the basis of Article 50 of Act OG 151/2022. The cessation of Act OG 45/09 does not by itself lead to the cessation of the Regulation, because the new act expressly regulated its transitional status. Its cessation depends on the cumulative fulfilment of the conditions of Article 50, that is, the adoption of the Quality Standards and of the decision on the form and detailed content of the licence.
```

## License

CC BY 4.0 (https://creativecommons.org/licenses/by/4.0/). Excerpts of legal provisions are reproduced from the Croatian Official Gazette (Narodne novine) and identified by their ELI.

## Citation

Please cite the paper and this resource: https://github.com/amestrovic/TeLex-KG.

