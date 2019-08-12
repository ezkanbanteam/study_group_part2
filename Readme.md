# Richardson Maturity Model 

## REST = Resource Representational State Transfer

RESTful API is a API that uses HTTP request to POST, GET, PUT, DELETE data. 

## HTTP verbs in CRUD services
HTTP verb | CRUD service   
--------- | --------------
POST      | Create operation
GET       | Read operation
PUT       | Update operation
DELETE    | Delete operation

## HTTP Status code 
Status code | Message   
----------- | --------------
200         | OK
201         | Created
404         | Not found
409         | Conflict

## Overview

![overview](https://raw.githubusercontent.com/ezkanbanteam/study_group_part2/master/overview.png)

## Level 0: The Swamp of POX
The starting point for the model is using HTTP as a transport system for remote interactions, but without using any of the mechanisms of the web.

### Scenario : I want to book an appointment with my doctor

![level0](https://raw.githubusercontent.com/ezkanbanteam/study_group_part2/master/level0.png)

#### First, appointment software makes a request of the hospital appointment system to obtain information.

```xml
POST /appointmentService HTTP/1.1
[various other headers]

<openSlotRequest date = "2019-08-13" doctor = "Adam"/>
```

The server then will return a document giving me this information.

```xml
HTTP/1.1 200 OK
[various headers]

<openSlotList>
  <slot start = "1400" end = "1500">
    <doctor id = "Adam"/>
  </slot>
  <slot start = "1800" end = "1900">
    <doctor id = "Adam"/>
  </slot>
</openSlotList>
```

#### Next step is to book an appointment, which I can again do by posting a document to the endpoint.

```xml
POST /appointmentService HTTP/1.1
[various other headers]

<appointmentRequest>
  <slot doctor = "Adam" start = "1400" end = "1500"/>
  <patient id = "Kevin"/>
</appointmentRequest>

```

If all is well I get a response saying my appointment is booked.

```xml
HTTP/1.1 200 OK
[various headers]

<appointment>
  <slot doctor = "Adam" start = "1400" end = "1500"/>
  <patient id = "Kevin"/>
</appointment>
```

If there is a problem, say someone else got in before me, then I'll get some kind of error message in the reply body.

```xml
HTTP/1.1 200 OK
[various headers]

<appointmentRequestFailure>
  <slot doctor = "Adam" start = "1400" end = "1500"/>
  <patient id = "Kevin"/>
  <reason>Slot not available</reason>
</appointmentRequestFailure>
```

## Level 1: Resources

Now rather than making all our requests to a singular service endpoint, we now start talking to individual resources.

![level1](https://raw.githubusercontent.com/ezkanbanteam/study_group_part2/master/level1.png)

### With our initial query, we might have a resource for given doctor.

```xml
POST /doctors/mjones HTTP/1.1
[various other headers]

<openSlotRequest date = "2019-08-13"/>
```
The reply carries the same basic information, but each slot is now a resource that can be addressed individually.

```xml
HTTP/1.1 200 OK
[various headers]

<openSlotList>
  <slot id = "1234" doctor = "mjones" start = "1400" end = "1450"/>
  <slot id = "5678" doctor = "mjones" start = "1600" end = "1650"/>
</openSlotList>
```

With specific resources booking an appointment means posting to a particular slot.

```xml
POST /slots/1234 HTTP/1.1
[various other headers]

<appointmentRequest>
  <patient id = "jsmith"/>
</appointmentRequest>
```

If all goes well I get a similar reply to before.

```xml
HTTP/1.1 200 OK
[various headers]

<appointment>
  <slot id = "1234" doctor = "mjones" start = "1400" end = "1450"/>
  <patient id = "jsmith"/>
</appointment>
```

The difference now is that if anyone needs to do anything about the appointment, like book some tests, they first get hold of the appointment resource, which might have a URI like http://royalhope.nhs.uk/slots/1234/appointment, and post to that resource.

## Level 2 - HTTP Verbs
It use the HTTP verbs as closely as possible to how they are used in HTTP itself.

![level2](https://raw.githubusercontent.com/ezkanbanteam/study_group_part2/master/level2.png)

For our the list of slots, this means we want to use GET.

```xml
GET /doctors/mjones/slots?date=20190813&status=open HTTP/1.1
Host: royalhope.nhs.uk
```

The reply is the same as it would have been with the POST

```xml
HTTP/1.1 200 OK
[various headers]

<openSlotList>
  <slot id = "1234" doctor = "mjones" start = "1400" end = "1450"/>
  <slot id = "5678" doctor = "mjones" start = "1600" end = "1650"/>
</openSlotList>
```

To book an appointment we need an HTTP verb that does change state, a POST or a PUT. I'll use the same POST that I did earlier.

```xml
POST /slots/1234 HTTP/1.1
[various other headers]

<appointmentRequest>
  <patient id = "jsmith"/>
</appointmentRequest>
```

Even if I use the same post as level 1, there's another significant difference in how the remote service responds. If all goes well, the service replies with a response code of <font color=green>201</font> to indicate that there's a new resource in the world.

```xml
HTTP/1.1 201 Created
Location: slots/1234/appointment
[various headers]

<appointment>
  <slot id = "1234" doctor = "mjones" start = "1400" end = "1450"/>
  <patient id = "jsmith"/>
</appointment>
```

There is another difference if something goes wrong, such as someone else booking the session.

```xml
HTTP/1.1 409 Conflict
[various headers]

<openSlotList>
  <slot id = "5678" doctor = "mjones" start = "1600" end = "1650"/>
</openSlotList>
```

## Level 3 - Hypermedia Controls
It addresses the question of how to get from a list open slots to knowing what to do to book an appointment.

![level3](https://raw.githubusercontent.com/ezkanbanteam/study_group_part2/master/level3.png)

### We begin with the same initial GET that we sent in level 2

```xml
GET /doctors/mjones/slots?date=20190813&status=open HTTP/1.1
Host: royalhope.nhs.uk
```
But the response has a new element
Each slot now has a link element which contains a URI to tell us how to book an appointment.

<font color="purple">rel</font> describe the kind of relationship
<font color="purple">uri</font> for the target URI

```xml
HTTP/1.1 200 OK
[various headers]

<openSlotList>
  <slot id = "1234" doctor = "mjones" start = "1400" end = "1450">
     <link rel = "/linkrels/slot/book" 
           uri = "/slots/1234"/>
  </slot>
  <slot id = "5678" doctor = "mjones" start = "1600" end = "1650">
     <link rel = "/linkrels/slot/book" 
           uri = "/slots/5678"/>
  </slot>
</openSlotList>
```

The POST would again copy that of level 2

```xml
POST /slots/1234 HTTP/1.1
[various other headers]

<appointmentRequest>
  <patient id = "jsmith"/>
</appointmentRequest>
```

And the reply contains a number of hypermedia controls for different things to do next.

```xml
HTTP/1.1 201 Created
Location: http://royalhope.nhs.uk/slots/1234/appointment
[various headers]

<appointment>
  <slot id = "1234" doctor = "mjones" start = "1400" end = "1450"/>
  <patient id = "jsmith"/>
  <link rel = "/linkrels/appointment/cancel"
        uri = "/slots/1234/appointment"/>
  <link rel = "/linkrels/appointment/addTest"
        uri = "/slots/1234/appointment/tests"/>
  <link rel = "self"
        uri = "/slots/1234/appointment"/>
  <link rel = "/linkrels/appointment/changeTime"
        uri = "/doctors/mjones/slots?date=20100104&status=open"/>
  <link rel = "/linkrels/appointment/updateContactInfo"
        uri = "/patients/jsmith/contactInfo"/>
  <link rel = "/linkrels/help"
        uri = "/help/appointment"/>
</appointment>
```
### In kanabn project
#### Level 0
``` json
POST /board/editBoard HTTP/1.1
Host: ssl-ezscrum.csie.ntut.edu.tw

{
    "boardId"  : "880a9cfa-bf71-4d67-9f10-839da381eb10",
    "title" : "ezKanban"
}
```

```json
HTTP/1.1 200 OK
{
    outputMeaasge : "Title is changed."
}
```

#### Level 1
```json 
POST /board/880a9cfa-bf71-4d67-9f10-839da381eb10/edit HTTP/1.1
Host: ssl-ezscrum.csie.ntut.edu.tw
{
    "title" : "ezKanban"
}
```

```json
HTTP/1.1 200 OK
{
    outputMeaasge : "Title is changed."
}
```

#### Level 2
```json 
PUT /board/880a9cfa-bf71-4d67-9f10-839da381eb10 HTTP/1.1
Host: ssl-ezscrum.csie.ntut.edu.tw
{
    "title" : "ezKanban"
}
```

```json
HTTP/1.1 200 OK
{
    outputMeaasge : "Title is changed."
}
```

#### Level 3
```json 
POST /board/fj49f34f-jd49-95fk-f93k-fk93kfpw0fjr HTTP/1.1
Host: ssl-ezscrum.csie.ntut.edu.tw
{
    "title" : "ezScrum"
}
```

```json
HTTP/1.1 201 OK
{
    outputMeaasge : "Title is changed.",
    [
        {
            rel: "edit",
            uri: "/edit/880a9cfa-bf71-4d67-9f10-839da381eb10"
        },
        {
            rel: "delete",
            uri: "/delete/880a9cfa-bf71-4d67-9f10-839da381eb10"
        }
    ]
    
}
```