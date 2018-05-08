---
layout: post
title:  "Introduction of Google API discovery"
lang: en
date: 2018-03-26
type: tech
---

### Concepts

API discovery is a service that can be able to build tools working with APIs of google services. the tools can be client libraries, IDE plugins, other tools plugins, providers that interact with google APIs. it provides the metadata of APIs in a machine-readable way. it returns schema, resources, and other basic information via a specific API.

it includes two levels of descriptions:

*  A directory of all APIs.
*  A detailed information for each of APIs

### Schema of API discovery

#### APIs directory

The APIs directory is a service which published on [https://www.googleapis.com/discovery/v1/apis](https://www.googleapis.com/discovery/v1/apis), it will return all APIs information of it supported as following style:

```shell
"kind":"discovery#directoryList",
"discoveryVersion":"v1",
"items":[]
```

```items``` is a array of item which represents an API, the detailed information looks like

```shell
{
"kind":"discovery#directoryItem",
"id":"compute:v1",
"name":"compute",
"version":"v1",
"title":"Compute Engine API",
"description":"Creates and runs virtual machines on Google Cloud Platform.",
"discoveryRestUrl":"https://www.googleapis.com/discovery/v1/apis/compute/v1/rest",
"discoveryLink":"./apis/compute/v1/rest",
"icons":{
"x16":"https://www.google.com/images/icons/product/compute_engine-16.png",
"x32":"https://www.google.com/images/icons/product/compute_engine-32.png"
},
"documentationLink":"https://developers.google.com/compute/docs/reference/latest/",
"preferred":true
}
```

The most important part is the link of API information, it can be found via ```discoveryRestUrl``` parameter, e.g. the link of ```compute service```  is

```shell
https://www.googleapis.com/discovery/v1/apis/compute/v1/rest
```

#### Structure of API Schema

As above mentioned, the link of each API can be found in APIs directory. by which can be able to get the detailed information of this API service. the structure is

```
{
"kind":"discovery#restDescription",
"etag":"\"-iA1DTNe4s-I6JZXPt1t1Ypy8IU/QdrSoC3sccj4V1AevKB02CZS13o\"",
"discoveryVersion":"v1",
"id":"compute:v1",
"name":"compute",
"version":"v1",
"revision":"20180314",
"title":"Compute Engine API",
"description":"Creates and runs virtual machines on Google Cloud Platform.",
"ownerDomain":"google.com",
"ownerName":"Google",
"icons":{},
"documentationLink":"https://developers.google.com/compute/docs/reference/latest/",
"protocol":"rest",
"baseUrl":"https://www.googleapis.com/compute/v1/projects/",
"basePath":"/compute/v1/projects/",
"rootUrl":"https://www.googleapis.com/",
"servicePath":"compute/v1/projects/",
"batchPath":"batch/compute/v1",
"parameters":{},
"auth":{},
"schemas":{},
"resources":{}
}
```

There are three additional parties except the basic information which are important to this structure. 

* parameters
* schemas
* resources

***```parameters```***  are any parameters to apply to this query. which will be appended after the URL. the pattern is

```
https://www.googleapis.com/discovery/v1/apis/api/version/rest?parameters
```

If we take a look at the whole url in request, it would be as following format:    
***```Absolute URI +  BasePath + ResourcePath + Parameters```***

let us take the instances of compute service as an example, if we are going to create an instance, the following request url would be taken:

```
https://www.googleapis.com/compute/v1/projects/{project}/zones/{zone}/instances?orderBy=creationTimestamp
```

this value of each parts refer to the pattern would be:

* Absolute URI :  www.googleapis.com        
* BasePath: compute/v1/projects        
* ResourcePath: {project}/zones/{zone}/instances       
* Parameters: orderBy=creationTimestamp        

In general the parameters will be used to pass some additional information for the request. e.g. filter the result set by keyword.

***```schema```*** is the definition of resource body. it can be either the body of request or the body of response. it will be used when resource request happen. the structure of schema will let client know how the body looks like.  let's also take the instance as an example. the schema structure would be:

```
"Instance":{
"id":"Instance",
"type":"object",
"description":"An Instance resource.",
"properties":{}
}
```

It contains the basic information and the properties of it. properties consists in a number of key-value pairs that represents the style which can be taken to pass. e.g.

```
"canIpForward":{
"type":"boolean",
"description":"Allows this instance to send and receive packets with non-matching destination or source IPs."
},
"cpuPlatform":{
"type":"string",
"description":"[Output Only] The CPU platform used by this instance."
},
...
```

***```resources```***  represents the collection of operations on one object via API, it also define what the whole request/response looks like. includes the body, the HTTP method such GET, POST, DELETE... the url which has been mentioned before. the format is

```json
"instances":{
"methods":{
"get":{},
"insert":{},
"list":{
"id":"compute.instances.list",
"path":"{project}/zones/{zone}/instances",
"httpMethod":"GET",
"description":"Retrieves the list of instances contained within the specified zone.",
"parameters":{
"filter":{
"type":"string",
"location":"query"
},
"maxResults":{
"type":"integer",
"default":"500",
"format":"uint32",
"minimum":"0",
"location":"query"
},
"orderBy":{
"type":"string",
"location":"query"
},
"pageToken":{
"type":"string",
"description":"Specifies a page token to use. Set pageToken to the nextPageToken returned by a previous list request to get the next page of results.",
"location":"query"
},
"project":{
"type":"string",
"description":"Project ID for this request.",
"required":true,
"pattern":"(?:(?:[-a-z0-9]{1,63}\\.)*(?:[a-z](?:[-a-z0-9]{0,61}[a-z0-9])?):)?(?:[0-9]{1,19}|(?:[a-z0-9](?:[-a-z0-9]{0,61}[a-z0-9])?))",
"location":"path"
},
"zone":{
"type":"string",
"description":"The name of the zone for this request.",
"required":true,
"pattern":"[a-z](?:[-a-z0-9]{0,61}[a-z0-9])?",
"location":"path"
}
},
"parameterOrder":[
"project",
"zone"
],
"response":{
"$ref":"InstanceList"
},
"scopes":[
"https://www.googleapis.com/auth/cloud-platform",
"https://www.googleapis.com/auth/compute",
"https://www.googleapis.com/auth/compute.readonly"
]
},
...
```

By now, we have went through all of the API discovery definitions. all in one word, API discovery provides the metadata service for all google apis. by which the client tools can be design in an extensible way. 

