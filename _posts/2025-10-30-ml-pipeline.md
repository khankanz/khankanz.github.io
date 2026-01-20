---
layout: post
title: "Building a ML Pipeline for Microcalcification Classification in OMOP"
date: 2025-10-30
excerpt: "OMOP's non-auto-incrementing PKs taught me more about database design than any tutorial."
tags: [ml-pipeline, omop, fhir, database-integration, nlp, radiology, sqlalchemy, pydantic, deberta]
---

This post is a detailed system design walkthrough for an NLP pipeline built to classify microcalcification status from radiology reports and persist the findings into an OMOP Common Data Model. Starting with no existing infrastructure, I'll break down the key technical challenges and implementation details. We'll cover everything from establishing a robust, transactional database foundation to solving for non-auto-incrementing primary keys in a concurrent environment, dynamically routing NLP results to the correct OMOP domain tables, and using the `fact_relationship` table to ensure data linkage and discoverability in tools like Atlas.

---

## Introduction
Today, I'm going to walk you through the system design and implementation of an ML pipeline I built for classifying microcalcification status from radiology reports. This was one of my initial tasks, starting from zero plumbing. We read radiology reports from an Aidbox database, where they're stored as observations linked to diagnostic reports of the FHIR resource. We then use a trained DeBERTa microcalcifications model to classify the status, and save the results into an OMOP database. This involved solving multiple challenges at each step—I'll focus on the key investigations and implementations I handled. Let's dive into the pipeline end-to-end.

## Database Connection and Setup
First, we connect to a Postgres database. I created environment files that the script reads initially—these contain database connection variables. This allows the same code to run on testing or production systems.

We have helper functions to test the connection and ensure it's successful. There's also setup for logging, saving all actions to a log file. We're using SQLAlchemy as a Pythonic wrapper around database controls. Logging is done at the database connection level, so every read-write operation gets logged—any transactions on the database are captured.

Next, we use another library called FastAI, which provides a simplifying wrapper around the database. We set `DBM = DB.meta` to get the metadata of the database. This lets us do things like `DBM.tables` to get all table names, and reflect the structure. Reflection means creating Pythonic objects (using Pydantic or SQLAlchemy) that represent the database structure. For example, a person is represented as `P = DBM.person`.

We define a context manager via `contextlib`. There's also a helper function for Year, Month, Day, which is handy for dealing with Aidbox. This gives us a data model representing valid columns, like in the persons table.

We create a `dbTable` object, which includes the connection to the actual table, the database object, and the data model (e.g., for person). The context manager allows transactions on tables: we try the transaction, and if there's an exception, we roll back to avoid corrupt data. This lets us query like "Find me a person where person source value or pid is equal to xyz" and retrieve the person.

## Handling Primary Keys (No Auto-Increment)
There's a weird situation with OMOP tables where the next primary key isn't auto-incremented. I investigated and wrote a helper function to get the next primary key for a given table. It grabs the first primary key column, runs a SQL query to check the maximum value in that column. If nothing is returned or it's 0, next ID is 1; otherwise, it's the max plus 1.

But there was a bug: with batch calls and multiple processes, we might get the same max ID. My workaround was to create `get next primary key range`—same process, but it gets a range based on how many you need to create. You use that range to populate IDs when creating observations.

## Adding Persons and Basic Records
The first functionality I built was to add a person. Given person data and the persons table, it checks if there's an existing patient by matching on `personSourceValue`. If there's a match, return it; otherwise, insert the data.

For notes, there are helper functions for parsing date time, which is a big pain—FHIR date time needs conversion to an OMOP-acceptable format.

There's a general `fetch records` function: given a database table, column name, and where clause, it fetches the record. This generalizes to persons table—for example, fetch by `person source value` (the FHIR ID). I used Python partials to make it cooler: set the persons table and `person source value` as the column, then just provide the ID to create `fetch patients`.

## Creating Notes
Creating a note starts with prepping the note object. You have the source value (a string), effective start date, patient ID, and note table. Parse the effective start date from the diagnostic report resource into date and time.

Return a note object: note ID from `get next primary key`, person ID as PID, note date as whatever. Key fields: `note type concept ID` is `type concept ID.EHR`; `Note Class Concept ID` is `Note Class Concept ID.diagnostic study`; `Note Text` is the source value. We agreed `Note Text` contains the FHIR Diagnostic Report ID.

Once prepped, add the note using a wrapper helper. Roll this into `create and add the note`—it's an iterative buildup of functions.

## Concept Table Lookups and Caching
For the concept table, we need a function to take a term and look up details. Connect to the concept table as `concepts` (a DB table object). Search by concept name (e.g., "micro calcifications") to get SNOMED code or concept ID.

I put a cache around this lookup function because scanning the table takes time, and you won't look up the same concept many times—caching keeps it in memory to save computation.

The `getConceptDetails` function lets you specify vocabulary as `vocabulary type SNOMED` and concept class (none or something else). It searches for concept name like the term, where standard concept is 's' and vocabulary ID is SNOMED. Returns the first match as a dictionary (e.g., for "microcalcification": concept ID 4132707, name "microcalcification"; for "negative": concept ID 9189, name "negative"). If no match, raise ValueError.

`processNlpFinding` takes a term (string) and NLP result, returns a dictionary in question-answer format. Map using the concept table: question is from `getConceptDetails` on the term; answer is the NLP result. Example: `processNlpFinding('microcalcification unknown')` gives a dict with question (concept ID, name) and answer (concept ID, name).

## Creating Note NLP Entries
OMOP is person-centric: person must exist to create a note (pointer back), and note must exist for note NLP (pointer back). Create sequentially.

Connect to note_nlp table as `notes_nlp`—understands structure for validation. `FetchNotes` is a partial of `fetch records` for notes. Fetch a note to get its object (note ID, person ID, date/time, etc.).

Helper: given a note object, extract date (handy for note NLP). To prep note NLP entry: use note date for alignment. Note NLP ID from `get next primary key`; note ID points back to note table (one-to-many: one note can have multiple note NLP). For lexical variant (string): process NLP finding, save as JSON (question-answer format). NLP date/time same as note. NLP system is model metadata: "Radbert V1" (manufacturer "EMR TEL Lab", device name "Radbert Microcalcifications", version "1.0.0").

I used Pydantic to extend objects representing tables (e.g., classes for `vocabulary type` like SNOMED, `concept ID`, `LOINC`, `no class concept ID`, `type concept ID`, language concept like note base class). Pydantic sets rules for valid/optional fields, validates on object creation before database insert.

Before insert, check if note NLP exists for the note (though this might not make sense later—skipping for now). There are batch functions and `create note NLP` building on this.

## Storing in Domain Tables
Note NLP breaks from OMOP-centric view; to make discoverable (e.g., via Atlas), store in domain tables. Fetch note NLP: get lexical variant (JSON question-answer), look at question concept ID (e.g., for microcalcification: 41327). Fetch concept from concepts table, check domain ID—it tells which OMOP CDM table (e.g., observations for microcalcifications).

Helper: given note NLP ID, return domain type (wraps this logic). Connect to observations table as `observations`. Get next primary key. Given note ID, get person ID from associated note.

To prep observation: start with note NLP ID, fetch entry, get findings from lexical variant. Observation ID: next primary key; person ID: from note; observation concept ID: findings question concept ID; date/time: same as note NLP/note; observation type concept ID: NLP (type concept); value as concept: findings answer concept ID; observation source value: "table = note NLP; ID = [note NLP ID]" (for tracking/linkage back).

## Handling Observation Periods
OMOP constraint: patients should be pre-populated with observation period (range of earliest/latest entries). When adding a note, check if note date falls within the patient's observation period; if not, update start/end date.

I created a class in OMOP CDM 5-4 classes called `ObservationPeriod`. Helper: given person ID, retrieve from observation period table, check if note date exceeds range (returns tuple: true/false for too far past/future). Update if needed—otherwise, no validation error, but the entry won't be discoverable in tools like Atlas (note -> note NLP -> domain table like observation).

## Using FACT_RELATIONSHIP for Linkages
More recently, rather than free text/JSON in observation source value pointing back to note NLP, we use the FACT_RELATIONSHIP table. It allows pointing note NLP to domain table entry (and vice versa) with two concepts: "Has associated procedure" and "Associate procedure of". For each pointer, add two entries in FACT_RELATIONSHIP for two-way linkage.

## Wrapping Up
Zooming out: We take a FHIR diagnostic report ID, get text, run through the model (we'll discuss that), process results into note and note NLP, then propagate to domain tables. This pipeline involved iterative helper functions, caching, transactions, and OMOP linkages I investigated deeply.

Questions? I'd love feedback on what to expand.
