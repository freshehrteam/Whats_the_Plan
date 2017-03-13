## What's my Plan Technical guide

This document describes the series of [Operon Ehrscape API](https://ehrscape.code4health.org/api-explorer.html) calls required for inegrating the WhatsMyPlan app with the openEHR Ehrscape API and assumes a base level of understanding of that API and the use of openEHR - further details can be found at [Overview of openEHR and Ehrscape](/docs/training/openehr_intro.md).

The steps covered are...  

* Open an Ehrscape Session

* Retrieve Patient's `ehrId` from Ehrscape, based on their `subjectId` (NHS Number)

* Run an AQL (Archetype Query Language) query which returns a 'flat list' of key datapoints from the patient's Adverse Reaction list.

* Close the Ehrscape Session

The baseURL for all Ehrscape API calls is https://ehrscape.code-4-health.org

### API parameter conventions

In the descriptions below we use the convention

`{{Password}}`

to indicate that a local variable or parameter should be substituted

The baseURL for all Ehrscape API calls is https://test.operon.systems.org

i.e. the call below, described as
`POST /rest/v1/session?username={{userName}}&password={{password}}`

where
```
login= mylogin
password = mypassword
```
should be resolved to
```
https://test.operon.systems.org/rest/v1/session?username=mylogin&password=mypassword
```

### A. Open the Ehrscape session

The first step in working with Ehrscape is to open a Session and retrieve the ``sessionId`` token. This allows subsequent API calls to be made without needing to login on each occasion.  
The session should be formally closed when you are finished.

##### Call: Create a new openEHR session:
 ````http
 POST /rest/v1/session?username={{userName}}&password={{password}}
 ````
##### Returns:
````json
{
"sessionId": "fc234d24-7b59-49f5-a724-8d37072e832b"
}

````

### B. Retrieve Patient’s `ehrId` from Ehrscape based on their subjectId

We now need to retrieve the patient’s internal `ehrID` associated with their subjectId. The ehrID is a unique string which, for security reasons, cannot be associated with the patient, if for instance their openEHR records were leaked.

##### Call: Returns the EHR for the specified subject ID and namespace.
````
GET /rest/v1/ehr/?subjectId={{subjectId}}&subjectNamespace={{subjectNamespace}}
Headers:
 Ehr-Session: {{sessionId}} //The value of the sessionId
````
##### Return:
````json
{
  "ehrId": "0da489ee-c0ae-4653-9074-57b7f63c9f16"
}
````

### C. Retrieve the Patient’s list of allergies / adverse reactions

Now that we have the patient’s `ehrId` we can use it to locate their existing records.
We use an Archetype Query Language (AQL) call to retrieve a list of their allergies ``composition`` records. Compositions are document-level records which act as the container for all openEHR patient data.

AQL is a database-neutral query language which is understood by openEHR back-end servers.

##### AQL statement

````sql
select
	a/uid/value as compositionId,
    a/context/start_time/value as dateRecorded,
    b_a/data[at0001]/items[at0002]/value/value as Causative_agent,
    b_a/data[at0001]/items[at0009]/items[at0011]/value/value as Manifestation
from EHR e [a/ehr_id/value = '{{ehrId}}']
contains COMPOSITION a[openEHR-EHR-COMPOSITION.adverse_reaction_list.v1]
contains EVALUATION b_a[openEHR-EHR-EVALUATION.adverse_reaction_risk.v1]
where a/name/value='Adverse reaction list'
````

The query API call returns a `Resultset` which is a nested set of name/value pairs whose format is determined by the AQL query.

The `compositionId` element in the response is the unique identifier for the composition and `start_time` is the time that the document was authored.


##### Call: Run AQL query and return a Resultset
```
GET /rest/v1/query?aql=select 	a/uid/value as compositionId,     a/context/start_time/value as dateRecorded,     b_a/data[at0001]/items[at0002]/value/value as Causative_agent,     b_a/data[at0001]/items[at0009]/items[at0011]/value/value as Manifestation from EHR e [a/ehr_id/value = 'dabcbf61-94bb-45df-a472-9c7a489a200d'] contains COMPOSITION a[openEHR-EHR-COMPOSITION.adverse_reaction_list.v1] contains EVALUATION b_a[openEHR-EHR-EVALUATION.adverse_reaction_risk.v1] where a/name/value='Adverse reaction list'
Headers:
 Ehr-Session: {{sessionId}} //The value of the sessionId
```

##### Return:
````json
{
  "meta": {
    "href": "http://test.operon.systems/rest/v1/query/?aql=select%20%09a/uid/value%20as%20compositionId,%20%20%20%20%20a/context/start_time/value%20as%20dateRecorded,%20%20%20%20%20b_a/data%5Bat0001%5D/items%5Bat0002%5D/value/value%20as%20Causative_agent,%20%20%20%20%20b_a/data%5Bat0001%5D/items%5Bat0009%5D/items%5Bat0011%5D/value/value%20as%20Manifestation%20from%20EHR%20e%20%5Ba/ehr_id/value%20%3D%20'dabcbf61-94bb-45df-a472-9c7a489a200d'%5D%20contains%20COMPOSITION%20a%5BopenEHR-EHR-COMPOSITION.adverse_reaction_list.v1%5D%20contains%20EVALUATION%20b_a%5BopenEHR-EHR-EVALUATION.adverse_reaction_risk.v1%5D%20where%20a/name/value%3D'Adverse%20reaction%20list'"
  },
  "aql": "select \ta/uid/value as compositionId,     a/context/start_time/value as dateRecorded,     b_a/data[at0001]/items[at0002]/value/value as Causative_agent,     b_a/data[at0001]/items[at0009]/items[at0011]/value/value as Manifestation from EHR e [a/ehr_id/value = 'dabcbf61-94bb-45df-a472-9c7a489a200d'] contains COMPOSITION a[openEHR-EHR-COMPOSITION.adverse_reaction_list.v1] contains EVALUATION b_a[openEHR-EHR-EVALUATION.adverse_reaction_risk.v1] where a/name/value='Adverse reaction list'",
  "executedAql": "select \ta/uid/value as compositionId,     a/context/start_time/value as dateRecorded,     b_a/data[at0001]/items[at0002]/value/value as Causative_agent,     b_a/data[at0001]/items[at0009]/items[at0011]/value/value as Manifestation from EHR e [a/ehr_id/value = 'dabcbf61-94bb-45df-a472-9c7a489a200d'] contains COMPOSITION a[openEHR-EHR-COMPOSITION.adverse_reaction_list.v1] contains EVALUATION b_a[openEHR-EHR-EVALUATION.adverse_reaction_risk.v1] where a/name/value='Adverse reaction list'",
  "resultSet": [
    {
      "dateRecorded": "2016-12-20T00:11:02.518+02:00",
      "Manifestation": "eruption due to drug",
      "compositionId": "9809c9bd-daca-4a6d-8a7f-e78fc947af8c::hcbox.oprn.ehrscape.com::1",
      "Causative_agent": "allergy to penicillin"
    },
    {
      "dateRecorded": "2016-12-20T00:11:02.518+02:00",
      "Manifestation": "allergic angio-oedema due to ingested food",
      "compositionId": "9809c9bd-daca-4a6d-8a7f-e78fc947af8c::hcbox.oprn.ehrscape.com::1",
      "Causative_agent": "allergy to seafood"
    }
  ]
}
````

### D. Commit a Patient Note to Ehrscape

To be done

### E. Retrieve a whole Patient Note from Ehrscape

To be done

### F. Retrieve parts of a Patient Note from Ehrscape via AQL

To be done

### G. Close the Ehrscape session

The last step in working with Ehrscape is to close the session.

##### Call: Close the openEHR session:
 ````
 DELETE /rest/v1/session?sessionId={{sessionId}}
 ````
##### Returns:
````json
{
  "sessionId": "2dcd6528-0471-4950-82fa-a018272f1339"
}
````
