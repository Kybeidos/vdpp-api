# How to use the VDPP REST API

## Preface
The Versatile Document Processing Platform (VDPP) offers NLP services to analyze documents of various formats.
The VDPP REST API is designed around the aspect of documents, (NLP) services and requests to use a service on a subset of documents.

## Table of Contents
1. [Structure](#1-structure)
    1. [Documents](#11-documents)
    2. [Services](#12-services)
    3. [Requests](#13-requests)
2. [Example Scenario](#2-example-scenario)
3. [Links](#3-links)

## 1. Structure
An application that wants to use the VDPP REST API mainly interacts with three domain objects:
1. **Documents**:   A document object represents an arbitrary file including various metadata.
                    It is identified by its unique `path`.
2. **Services**:    A service object represents a service that ingests and processes documents.
                    The service may also create various results (i.e. summaries).
                    It is identified by its unique `code`.
3. **Requests**:    A request object represents a user-request. It also tracks the processing status
                    and provides the processing results once available.
                    It is identified by its unique `id`.
                    
### 1.1. Documents
Documents act as the data source for VDPP's services. Currently, the platform cannot directly process
documents located on the user's system. All relevant files must be uploaded instead.

There are two ways to upload documents to the VDPP service:
* Specifying an external URL that VDPP can download the document from.
Send a `POST` request to the endpoint `/documents` with the following minimal JSON body:
```
{
    "path": "/my/unique/path/file.txt",
    "url": "http://direct.link.com"
}
```
* Directly uploading the document via the VDPP API. Send a `POST` request to the endpoint `/documents` of format
`multipart/form-data`:

Using cURL
```
curl --request POST 'http://host:port/documents' \
--header 'Content-Type: multipart/form-data' \
--form 'path=/my/unique/path/' \
--form 'doc01.txt=@/user/me/doc01.txt' \
--form 'doc02.txt=@/user/me/doc02.txt'
```

Either way results in the document uploaded to be available for processing with one of VDPP's services.

### 1.2. Services
The list of available services can be retrieved by sending a `GET /services` request.
Each service description contains information about the usage (i.e. parametrization) of the service.

### 1.3. Requests
Requests combine documents and a service to trigger processing in VDPP.
To create a request, users must `POST` a JSON body to the endpoint `/requests` of the following format:
```
{
    "serviceCode": "CODE",
    "parameters": [{
    	"name": "PARAM1",
    	"values": ["value of PARAM1"]
    }, 
    {
    	"name": "PARAM2",
    	"values": ["value1", "value2"]
    }],
    "documentIds": [
            "/my/unique/path/doc01.txt",
            "/my/unique/path/doc02.txt"
    ]
}
```
Sending the request schedules a job for the given service which will be initialized with the given parameters and documents.

## 2. Example Scenario
Bob wants to automatically summarize a document to get a quick overview of the document's contents.

(1)
First, he uploads his document using cURL:
```
curl --request POST 'http://host:port/documents' \
--header 'Content-Type: multipart/form-data' \
--form 'path=/bobsdocuments/' \
--form 'doc01.txt=@/user/me/doc01.txt'
```
If everything went fine, he should receive the following answer with status code `200 OK`:
```
[
    {
        "path": "/bobsdocuments/doc01.txt",
        "creator": null,
        "editor": null,
        "dateCreated": "2020-01-13T13:11:10.236+0000",
        "dateModified": "2020-01-13T13:11:10.236+0000",
        "encodingFormat": "text/plain",
        "geoData": null,
        "url": null
    }
]
```

(2)
Bob forgot the exact name for the summarization service, so he sends following request using cURL:
```
curl --request GET 'http://host:port/services'
``` 
In the list of services that he then receives, he discovers
```
[
    {
        "id": "35bf401f-bb52-478c-bd12-2ad8ad6d6db7",
        "name": "LSA Summarization",
        "code": "SUMMARIZATLSA",
        "description": "Summarizes one or more docs using LSA algorithm",
        "serviceType": "Summarization",
        "supportedMimeTypes": [
            "*.docx",
            "*.xls",
            "*.xlsx",
            "*.pptx",
            "*.odt",
            "*.pdf",
            "*.txt",
            "*.csv",
            "*.html",
            "*.log"
        ],
        "parameters": [
            {
                "name": "ORIGINATOR",
                "description": "The owner/customer of the document",
                "type": "String",
                "constraints": [
                    "UTF-8"
                ],
                "displayInUI": false
            },
            {
                "name": "LANGUAGE",
                "description": "Document language (affects summarization!)",
                "type": "enum",
                "constraints": [
                    "en",
                    "de"
                ],
                "displayInUI": true
            },
            {
                "name": "SUMMLENGTH",
                "description": "Required summary length in sentences",
                "type": "Integer",
                "constraints": [
                    ">0",
                    "< #sentences of document"
                ],
                "displayInUI": true
            }
        ],
        "displayInUI": true
    }
]
```
The service code is `SUMMARIZATLSA`.
The description says that text files are supported.
There are three parameters, of which two are relevant: `LANGUAGE` and `SUMMLENGTH`.

(`ORIGINATOR` is irrelevant/hidden because of the value `"displayInUI": false`)

(3) His text is in English and he wants a summary length of 10 sentences.
Therefore, Bob sends the following request using cURL:
```
curl --request POST 'http://host:port/requests' \
--header 'Content-Type: application/json' \
--data-raw '{
    "serviceCode": "SUMMARIZATLSA",
    "parameters": [{
    	"name": "LANGUAGE",
    	"values": ["en"]
    }, 
    {
    	"name": "SUMMLENGTH",
    	"values": ["10"]
    }],
    "documentIds": ["/bobsdocuments/doc01.txt"]
}'
``` 
Again, with status code `200 OK`, he receives the following answer:
```
{
    "id": "fcfd6604-8720-4ba1-9ceb-6de8a0189a7a",
    "serviceCode": "SUMMARIZATLSA",
    "parameters": [{
    	"name": "LANGUAGE",
    	"values": ["en"]
    }, 
    {
    	"name": "SUMMLENGTH",
    	"values": ["10"]
    }],
    "documentIds": ["/bobsdocuments/doc01.txt"]
    "status": {
        "dateCreated": "2020-01-13T13:28:07.520+0000",
        "dateModified": "2020-01-13T13:28:07.525+0000",
        "statusCode": "SCHEDULED",
        "statusMessage": "The request has been scheduled and will be processed shortly."
    },
    "results": []
}
```
He sees that his request is now scheduled. The id of his request is `fcfd6604-8720-4ba1-9ceb-6de8a0189a7a`.

(4) Now he can poll the request status until it indicates that the result is available:
```
curl --request GET 'http://host:port/requests/fcfd6604-8720-4ba1-9ceb-6de8a0189a7a/status'
```
```
{
    "dateCreated": "2020-01-13T13:28:07.520+0000",
    "dateModified": "2020-01-13T13:28:21.560+0000",
    "statusCode": "SUCCEEDED",
    "statusMessage": "The request has been successfully processed and the results can now be retrieved."
}
```

(5) He can then get the results:
```
curl --request GET 'http://host:port/requests/fcfd6604-8720-4ba1-9ceb-6de8a0189a7a/result'
```
```
[
    {
        "dateCreated": "2020-01-13T13:28:21.555+0000",
        "results": [
            {
                "name": "summary.txt",
                "description": "The summary of doc01.txt",
                "encodingFormat": "text/plain",
                "url": "http://hw02.kybeidos.de:50070/webhdfs/v1/apps/VDPP/NlpResults/fcfd6604-8720-4ba1-9ceb-6de8a0189a7a/summary.txt?op=OPEN",
                "type": "user_file"
            }
        ]
    }
]
```
The `url` field describes a download link which Bob can use to download the result file.

## 3. Links
Current API in Swagger format: http://hw05.kybeidos.de:8090
