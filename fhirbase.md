## fhirbase: FHIR persistence in PostgreSQL

FHIR is specification of semantic resources and API for working with healthcare data.
Please address the [official specification](http://hl7-fhir.github.io/) for more details.

To implement FHIR server we have to persist & query data in application internal
format or in FHIR format. This article describes how to store FHIR resources
in relational database (PostgreSQL) and about opensource FHIR storage implementation - [fhirbase](https://github.com/fhirbase/fhirbase).


## Overview

FHIR describes ~ 100 [resources](http://hl7-fhir.github.io/resourcelist.html)
as base StructureDefinitions, which by themselves are resources in FHIR terms.

Fhirbase stores each resource in two tables - one for current version and second for previous versions of resources.

For example resources of type Patient are stored in patient and patient_history tables:

```psql
fhirbase=# \d patient
                       Table "public.patient"
    Column     |           Type           |        Modifiers
---------------+--------------------------+-------------------------
 version_id    | text                     |
 logical_id    | text                     | not null
 resource_type | text                     | default 'Patient'::text
 updated       | timestamp with time zone | not null default now()
 published     | timestamp with time zone | not null default now()
 category      | jsonb                    |
 content       | jsonb                    | not null
Inherits: resource

fhirbase=# \d patient_history

                   Table "public.patient_history"
    Column     |           Type           |        Modifiers
---------------+--------------------------+-------------------------
 version_id    | text                     | not null
 logical_id    | text                     |
 resource_type | text                     | default 'Patient'::text
 updated       | timestamp with time zone | not null default now()
 published     | timestamp with time zone | not null default now()
 category      | jsonb                    |
 content       | jsonb                    | not null
Inherits: resource_history


```


Minimal installation of fhirbase consists of only
few tables for "meta" resources provided by FHIR specification:

* StructureDefinition
* OperationDefinition
* SearchParameter
* ValueSet

This tables are filled from FHIR distribution
with core resources, ie base profiles & search params resources,
defined by FHIR valuesets and operations.

You can generate tables for specific resources using generate_tables() procedure:

```sql
SELECT fhir.generate_tables('{Patient, Organization, Encounter}')
```

This will generate tables to store patient,
organization and encounter resources.

If you call generate_tables() without parameters, fhirbase will generate tables for all resources. Function return amount of generated tables.

```sql
SELECT fhir.generate_tables();
```

When concrete resource type tables are generated,
column installed in profile table for this resource is set to true.

Functions representing public API of fhirbase are all located in fhir schema.
The first group of functions implements CRUD operations on resources:

```sql

SELECT fhir.create('{"resourceType":"Patient", name: [{"given": ["John"]}}')
-- => {id:<generatedId>, meta: {versionId: <generatedId>}, resourceType":"Patient", name: ....}

SELECT fhir.update(updated_resource_with_versionId)
SELECT fhir.read('Patient', <generatedId>)
SELECT fhir.vread('Patient', <generatedId>)

SELECT fhir.history('Patient', <generatedId>)
-- return resource history

```
