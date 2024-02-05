# Koodoo Document Checker API Spec

This document outlines Koodoo's Document Checker REST APIs and helps you integrate them seamlessly into your applications!

**Version**: 1.0.0
**Updated on**: 17/01/2024

## System Design

![Document Checker API System Design](https://i.imgur.com/sFsWPFH.png "Koodoo Document Checker API System Design")

## High-level Flowchart

![Document Checker API Flowchart](https://i.imgur.com/5ERfHXp.png "Koodoo Document Checker API Flowchart")

## Usage Information

Across this document, if a field is not explicitly REQUIRED, it can be considered OPTIONAL.

We use the following basic types along with these formats:

| Type    | Format            | Supported comparisons                 |
|---------|-------------------|---------------------------------------|
| String  | string            | Equality                              |
| String  | date (YYYY-MM-DD) | Less than, equality, and greater than |
| Number  | Integer           | Less than, equality, and greater than |
| Boolean | -                 | Equality                              |
| Array   | -                 | -                                     |
| Object  | -                 | -                                     |
| Enum    | string            | -                                     |

The date format support less-than (`$lt`) and greater-than (`$gt`) inequality comparisons apart from the regular equality (`$eq`) comparison. For this reason, we accept an object for keys that use the date format like so:

Date format equality comparison:

```javascript
{
    "dob": {
        "$eq": "2024-01-01"
    }
}
```

Date format inequality comparison:

```javascript
{
    "dob": {
        "$lt": "2024-01-01",
        "$gt": "1990-01-01"
    }
}
```

This comparison format for dates is used during [rule application](#business_rules).

These are the HTTP response codes that we use across our API:

| Code | Description                                  |
|------|----------------------------------------------|
| 200  | Successful operation                         |
| 400  | Bad request                                  |
| 500  | Internal server error - Something went wrong |

## Authentication

All methods in this document require bearer authentication using the `Authorization` header.

| Field name    | Type   | Description                                                    |
|---------------|--------|----------------------------------------------------------------|
| Authorization | String | Bearer token is used as “Bearer: <API key>” for authorization. |

## API Base URL

The API base URL is: `https://api.koodoo.io/documents/v1.0.0/`. All methods use this base URL.

## Document Management

### POST /submission

**Content type**: multipart/form-data

This method ingests a set of documents. The type of data will be intuited from the magic bytes in the document if "document_type" is not passed.

> If the request succeeds, we immediately respond with a submission reference (i.e. submission_id).
>
> A submission is a collection of documents. Along with this submission_id, we respond with an array of document_ids (uuids) pertaining to the documents that were successfully received.
>
> You may use submission_id to immediately query our [results method](#result).


##### <a name="ingest_rf"></a>Request fields:

| Field name    | Type   | Description                                                                                                                                              |
|---------------|--------|----------------------------------------------------------------------------------------------------------------------------------------------------------|
| documents     | Array  | **REQUIRED** Array of document objects. Even a single document object may be passed. The next 3 fields represent each document object.                        |
| document_uuid | String | **REQUIRED** Client side document reference name. We will return this in the response.                                                                       |
| file_content  | String | **REQUIRED** Base64 encoding of file content                                                                                                                 |
| document_type | Enum   | File format of the document being sent. Currently supports one of ["pdf", "jpg", "png", "jpeg"]. If no document_type is sent, we will infer from magic bytes. |


Example request:

```json
{
  "documents": [
    {
      "document_uuid": "c064e4cd8cd7bc11d6861c2fb9fa1875",
      "document_content": "<<file contents>>",
      "document_type": "pdf"
    },
    {
      "document_uuid": "5849a89e383f05a7cd62fa76db548816",
      "document_content": "<<file contents>>",
      "document_type": "pdf"
    }
  ]
}
```

Example response:

```json
{
  "ok": true,
  "submission_id": "153f047f8c277954a5d2179ae3fe5959",
  "documents": ["c064e4cd8cd7bc11d6861c2fb9fa1875", "5849a89e383f05a7cd62fa76db548816"]
}
```


### PATCH /submissions/{submission_id}/documents

**Content type**: multipart/form-data

This method allows consumers to add documents to a submission.

The request fields are the same as the ingestion method - [link](#ingest_rf). However, we expect the following path parameter:

| Field name    | Type   | Description                     |
|---------------|--------|---------------------------------|
| submission_id | String | **REQUIRED** Submission identifier. |


Example request to add new document to submission:

```
PATCH /submissions/153f047f8c277954a5d2179ae3fe5959/documents
```

```json
{
  "documents": [
    {
      "document_uuid": "25240ad8324e9dfb0cec29c629d3dbe2",
      "document_content": "<<file contents>>",
      "document_type": "pdf"
    }
  ]
}
```

Example response:

```json
{
  "ok": true,
  "submission_id": "153f047f8c277954a5d2179ae3fe5959",
  "documents": ["c064e4cd8cd7bc11d6861c2fb9fa1875", "5849a89e383f05a7cd62fa76db548816", "25240ad8324e9dfb0cec29c629d3dbe2"]
}
```

### DELETE /submissions/{submission_id}/documents/{document_id}

This method allows consumers to delete documents from a submission.

##### Path parameters:

| Field name    | Type   | Description                     |
|---------------|--------|---------------------------------|
| submission_id | String | **REQUIRED** Submission identifier. |
| document_id   | String | **REQUIRED** Document identifier.   |

Example request to delete documents from submission:

```
DELETE /submissions/153f047f8c277954a5d2179ae3fe5959/documents/c064e4cd8cd7bc11d6861c2fb9fa1875
```

Example response:

```json
{
  "ok": true,
  "submission_id": "153f047f8c277954a5d2179ae3fe5959",
  "documents": ["5849a89e383f05a7cd62fa76db548816", "25240ad8324e9dfb0cec29c629d3dbe2"]
}
```

### DELETE /submissions/{submission_id}

This method allows consumers to delete an entire submission.

##### Path parameters:

| Field name | Type  | Description                                               |
|------------|-------|-----------------------------------------------------------|
| submission_id  | String | **REQUIRED** Submission Identifier |

Example request to delete an entire submission:

```
DELETE /submissions/153f047f8c277954a5d2179ae3fe5959
```

Example response:

```json
{
  "ok": true
}
```

## Check Submission results
### POST /submissions/{submission_id}/results

This method is used to pass business rules and query the result of a submission.

> **Note**: Given that there is significant processing that happens on our end, results will only be returned once available. To this end, our response contains an `is_ready` flag at two levels. One, at the parent level and others at different child levels. If the parent level `is_ready` flag is set to true, that means that all items from the request are ready to be consumed. If any of the child-level objects contain an `is_ready` flag set to false, that item is not yet ready.

##### Path parameters:

| Field name  | Type   | Description                                      |
|-------------|--------|--------------------------------------------------|
| submission_id | String | **REQUIRED** Submission ID for which a result is being queried. |


##### <a name="business_rules">Business Rules

This method supports two types of business rules:

1. Rules that apply to each (and every) document individually (ex: first name, last name, etc)
2. Rules that apply across all documents in a submission (ex: verify continuous dates)

For rules that apply to each document (1. above), we currently support the following checks:

| Rule key      | Description                                                       |
|---------------|-------------------------------------------------------------------|
| first_name    | Checks if extracted first name matches the supplied value.        |
| last_name     | Checks if extracted last name matches the supplied value.         |
| dob           | Checks if extracted date of birth matches the supplied value.     |
| document_date | Checks if extracted document date matches the supplied condition. |

For rules that apply across all documents in a submission (2. above), we currently support the following checks:

| Rule key                        | Description                                                                                                           |
|---------------------------------|-----------------------------------------------------------------------------------------------------------------------|
| verify_continuous_dates         | Checks if extracted dates across all documents in a submission are continous.                                               |
| continuous_date_range_tolerance | Accepts a tolerance value (in days) for the "verify_continuous_dates" check.                                          |
| date_range                      | Checks if extracted dates across all documents in a submission are within a date range specified by the supplied condition. |


Example submission results request:

```
POST /submissions/153f047f8c277954a5d2179ae3fe5959/results
```

Body:
```json
{
  "rules": {
    "each_document": {
      "first_name": "John",
      "last_name": "Doe",
      "dob": {
        "lt": "2023-01-01"
      },
      "document_date": {
        "gt": "1990-01-01",
        "lt": "2010-01-01"
      }
    },
    "across_documents": {
      "verify_continuous_date_range": true,
      "continuous_date_range_tolerance_days": 7,
      "date_range" : {
        "gt": "1990-01-01",
        "lt": "2010-01-01"
      }
    }
  }
}
```

There are three response possibilities when querying submission results:

1. No part of the submission results are ready
2. Some parts of the submission results are ready
3. All of the submission results are ready

Example response when results for no part of the submission results are ready:

```json
{
  "ok": true,
  "results": {
    "submission_id": "153f047f8c277954a5d2179ae3fe5959",
    "documents": [
      {
        "document_uuid": "5849a89e383f05a7cd62fa76db548816",
        "is_ready": false
      },
      {
        "document_uuid": "25240ad8324e9dfb0cec29c629d3dbe2",
        "is_ready": false
      }
    ]
  }
}
```

Example response when results for some parts of the submission results are ready:

```json
{
  "ok": true,
  "results": {
    "submission_id": "153f047f8c277954a5d2179ae3fe5959",
    "each_document": {
        "documents": [
            {
                "document_uuid": "5849a89e383f05a7cd62fa76db548816",
                "inferred_document_type": "payslip",
                "first_name": true,
                "last_name": true,
                "dob": false
            },
            {
                "document_uuid": "25240ad8324e9dfb0cec29c629d3dbe2",
                "is_ready": false
            }
        ]
    },
    "across_documents": {
        "is_ready": false
    }
  }
}
```

Example response when all of the submission results are ready:

```json
{
  "ok": true,
  "results": {
    "is_ready": true,
    "each_document": {
        "documents": [
            {
                "document_uuid": "5849a89e383f05a7cd62fa76db548816",
                "inferred_document_type": "payslip",
                "first_name": true,
                "last_name": true,
                "dob": false
            },
            {
                "document_reference_id": "25240ad8324e9dfb0cec29c629d3dbe2",
                "inferred_document_type": "payslip",
                "first_name": false,
                "last_name": false,
                "dob": false
            }
        ],
    },
    "across_documents": {
      "verify_continuous_date_range": false,
      "continuous_date_range_tolerance_days": true,
      "date_range" : true
    }
  }
}
```