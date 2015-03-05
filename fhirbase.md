## fhirbase: FHIR persistence in PostgreSQL

FHIR is specification of semantic resources and API for working with healthcare data.
Please address the [official specification](http://hl7-fhir.github.io/) for more details.

To implement FHIR server we have to persist & query data in application internal
format or in FHIR format. This article describes how to store FHIR resources
in relational database (PostgreSQL) and about open source FHIR
storage implementation - [fhirbase](https://github.com/fhirbase/fhirbase).


## Overview

*fhirbase* is built on top of PostgreSQL and require version higher then 9.4
(i.e. [jsonb](http://www.postgresql.org/docs/9.4/static/datatype-json.html) support).

FHIR describes ~ 100 [resources](http://hl7-fhir.github.io/resourcelist.html)
as base StructureDefinitions, which by themselves are resources in FHIR terms.

Fhirbase stores each resource in two tables - one for current version
and second for previous versions of resources. By convention tables are named
in downcase after resource types: Patient => patient, StructureDefinition => structuredefinition.

For example *Patient* resources are stored
in *patient* and *patient_history* tables:

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

All resource tables have similar structure and are inherited from *resource* table,
to allow cross-table queries (for more information see [PostgreSQL inheritance](http://www.postgresql.org/docs/9.4/static/tutorial-inheritance.html)).

Minimal installation of fhirbase consists of only
few tables for "meta" resources:

* StructureDefinition
* OperationDefinition
* SearchParameter
* ValueSet
* ConceptMap

This tables are filled with resources provided by FHIR distribution.

Most of API for fhirbase are represented as functions in *fhir* schema,
other schemas are used as code modules.

First helpful function is `fhir.generate_tables(resources text[])`, which generates tables
for specific resources passed as array.
For example to generate tables for patient, organization and encounter:

```sql
psql fhirbase

fhirbase=# select fhir.generate_tables('{Patient, Organization, Encounter}');

--  generate_tables
-----------------
--  3
-- (1 row)
```

If you call generate_tables() without any parameters,
then tables for all resources described in StructureDefinition
will be generated:

```sql

fhirbase=# select fhir.generate_tables();
-- generate_tables
-----------------
-- 93
--(1 row)
```

When concrete resource type tables are generated,
column installed in profile table for this resource is set to true.

Functions representing public API of fhirbase are all located in fhir schema.
The first group of functions implements CRUD operations on resources:

* create(resource json)
* read(resource_type, logical_id)
* update(resource json)
* vread(resource_type, version_id)
* delete(resource_type, logical_id)
* history(resource_type, logical_id)
* is_exists(resource_type, logical_id)
* is_deleted(resource_type, logical_id)


```sql

SELECT fhir.create('{"resourceType":"Patient", "name": [{"given": ["John"]}]}')
-- {
--  "id": "c6f20b3a...",
--  "meta": {
--    "versionId": "c6f20b3a...",
--    "lastUpdated": "2015-03-05T15:53:47.213016+00:00"
--  },
--  "name": [{"given": ["John"]}],
--  "resourceType": "Patient"
-- }
--(1 row)

-- create - insert new row into patient table
SELECT resource_type, logical_id, version_id,* from patient;

-- resource_type |  logical_id  |  version_id | content
---------------+---------------------------------------------------
-- Patient       | c6f20b3ab... | c6f20b....  | {"resourceType".....}
--(1 row)



SELECT fhir.read('Patient', 'c6f20b3a...');
-- {
--  "id": "c6f20b3a...",
--  "meta": {
--    "versionId": "c6f20b3a...",
--    "lastUpdated": "2015-03-05T15:53:47.213016+00:00"
--  },
--  "name": [{"given": ["John"]}],
--  "resourceType": "Patient"
-- }
--(1 row)

SELECT fhir.update(
   jsonbext.merge(
     fhir.read('Patient', 'c6f20b3a...'),
     '{"name":[{"given":"Bruno"}]}'
   )
);
-- returns update version

SELECT count() FROM patient; => 1
SELECT count() FROM patient_history; => 1

-- read previous version of resource
SELECT fhir.vread('Patient', /*old_version_id*/ 'c6f20b3a...');


SELECT fhir.history('Patient', 'c6f20b3a...');

-- {
--     "type": "history",
--     "entry": [{
--         "resource": {
--             "id": "c6f20b3a...",
--             "meta": {
--                 "versionId": "a11dba...",
--                 "lastUpdated": "2015-03-05T16:00:12.484542+00:00"
--             },
--             "name": [{
--                 "given": "Bruno"
--             }],
--             "resourceType": "Patient"
--         }
--     }, {
--         "resource": {
--             "id": "c6f20b3a...",
--             "meta": {
--                 "versionId": "c6f20b3a...",
--                 "lastUpdated": "2015-03-05T15:53:47.213016+00:00"
--             },
--             "name": [{
--                 "given": ["John"]
--             }],
--             "resourceType": "Patient"
--         }
--     }],
--     "resourceType": "Bundle"
-- }

SELECT fhir.is_exists('Patient', 'c6f20b3a...'); => true
SELECT fhir.is_deleted('Patient', 'c6f20b3a...'); => false

SELECT fhir.delete('Patient', 'c6f20b3a...');
-- return last version

SELECT fhir.is_exists('Patient', 'c6f20b3a...'); => false
SELECT fhir.is_deleted('Patient', 'c6f20b3a...'); => true
```
When resource is created *logical_id* and *version_id* is generated as sha1(resource::jsonb::text).
On every update resource content updated in *patient* table and old version of resource is copied
into *patient_history* table.


## search & indexing
