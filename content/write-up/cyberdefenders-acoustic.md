---
title: "Cyberdefenders · Digital Forensic · Acoustic"
date: 2023-09-03T22:08:20+02:00
draft: false
---

The scenario:

> This challenge takes you into the world of voice communications on the internet. VoIP is becoming the de-facto standard for voice communication. As this technology becomes more common, malicious parties have more opportunities and stronger motives to control these systems to conduct nefarious activities. This challenge was designed to examine and explore some of the attributes of the SIP and RTP protocols.

## Abstract

The article delves into the realm of voice communications on the internet, focusing on Voice over Internet Protocol (VoIP) technology. VoIP has become the standard for voice communication, but with its widespread adoption comes the risk of malicious exploitation. This challenge explores the Session Initiation Protocol (SIP) and Real-Time Transport Protocol (RTP) attributes to understand and counteract potential threats.

A series of questions guide the exploration, covering topics such as identifying the transport protocol, suites of scanning tools, User-Agent of the victim system, extensions requiring authentication, scanned extensions, real SIP client traces, dialed phone numbers, default credentials, RTP codec usage, sampling time, and uncovering hidden messages. 

![SIPVicious OSS logo; SIP-based VoIP systems security testing toolset.](/img/write-up/acoustic/sipvicious.png)

<figcaption>SIPVicious OSS logo; SIP-based VoIP systems security testing toolset.</figcaption>

---

## Q1. What is the transport protocol being used?

The first thing I like to do with Wireshark is to have an overview. Statistics > Protocol Hierarchy provides general information about the capture.

![Capture overview.](/img/write-up/acoustic/statistics.png)

<figcaption>Capture overview.</figcaption>

We can see two unusual protocols here: `Session Initiation Protocol` and `Real-Time Transport Protocol`. According to the [SIP's Wikipedia page](https://en.wikipedia.org/wiki/Session_Initiation_Protocol):

> SIP is designed to be independent of the underlying transport layer protocol and can be used with the User Datagram Protocol (UDP), the Transmission Control Protocol (TCP), and the Stream Control Transmission Protocol (SCTP).

In the statistics, we can see that `SIP` is under `UDP`.

## Q2. The attacker used a bunch of scanning tools that belong to the same suite. Provide the name of the suite.

I used the log file:

```
OPTIONS sip:100@honey.pot.IP.removed SIP/2.0
Via: SIP/2.0/UDP 127.0.0.1:5061;branch=z9hG4bK-2159139916;rport
Content-Length: 0
From: "sipvicious"<sip:100@1.1.1.1>; tag=X_removed
Accept: application/sdp
User-Agent: friendly-scanner
To: "sipvicious"<sip:100@1.1.1.1>
Contact: sip:100@127.0.0.1:5061
CSeq: 1 OPTIONS
Call-ID: 845752980453913316694142
Max-Forwards: 70
```

The user-agent is a good starting point, as I don't now anything in VoIP. Googling `friendly-scanner` will give some informations and especially this one:

> Also known as friendly-scanner, it is freely available to help pentesters, security teams and developers quickly test their SIP systems.

From: [SIPVicious OSS - open-source tools for testing VoIP security](https://www.enablesecurity.com/sipvicious/oss/).

## Q3. What is the User-Agent of the victim system?

Thanks to the hit, I have to check SIP packets. Filter this protocol by typing 'sip' in the Wireshark's search bar. Then, follow a conversation (right-click > follow > UDP stream) and you will see this kind of datas:

```
OPTIONS sip:100@172.25.105.40 SIP/2.0
Via: SIP/2.0/UDP 127.0.1.1:5060;branch=z9hG4bK-1453809699;rport
Content-Length: 0
From: "sipvicious"<sip:100@1.1.1.1>; tag=6163313936393238313363340131323031353530343335
Accept: application/sdp
User-Agent: UNfriendly-scanner - for demo purposes
To: "sipvicious"<sip:100@1.1.1.1>
Contact: sip:100@127.0.1.1:5060
CSeq: 1 OPTIONS
Call-ID: 61127078793469957194131
Max-Forwards: 70

SIP/2.0 200 OK
Via: SIP/2.0/UDP 127.0.1.1:5060;branch=z9hG4bK-1453809699;received=172.25.105.43;rport=5060
From: "sipvicious"<sip:100@1.1.1.1>; tag=6163313936393238313363340131323031353530343335
To: "sipvicious"<sip:100@1.1.1.1>;tag=as18cdb0c9
Call-ID: 61127078793469957194131
CSeq: 1 OPTIONS
User-Agent: Asterisk PBX 1.6.0.10-FONCORE-r40
Allow: INVITE, ACK, CANCEL, OPTIONS, BYE, REFER, SUBSCRIBE, NOTIFY
Supported: replaces, timer
Contact: <sip:172.25.105.40>
Accept: application/sdp
Content-Length: 0
```

## Q4. Which tool was only used against the following extensions: 100,101,102,103, and 111?

SIP Vicious contains few tools:

- svmap

- svwar

- svcrack

- svreport

- svcrash

I used hints and gone to source code of each tools. You have to read each tool code and in svcrack there is an interesting function:

```python
from sipvicious.libs.svhelper import ...makeRequest
```

This function is interesting because its used to forge the request and send it to the victim, you can find usefull information inside it:

```python
def makeRequest(method, fromaddr, toaddr, dsthost, port, callid, srchost='', branchunique=None, cseq=1,
    auth=None, localtag=None, compact=False, contact='sip:123@1.1.1.1', accept='application/sdp', contentlength=None,
    localport=5060, extension=None, contenttype=None, body='', useragent='friendly-scanner', requesturi=None):
```

Because of the `useragent` variable, which match logged user-agent inside `log.txt`, we can answer `svcrack.py`.

## Q5. Which extension on the honeypot does NOT require authentication?

I searched on `log.txt`, in this file you can find some `Authorization` header:

```
Authorization: Digest username="101",realm="localhost",nonce="3711809134",uri="sip:honey.pot.IP.removed",response="MD5_hash_removedXXXXXXXXXXXXXXXX",algorithm=MD5
```

It looks like an authentication, and, in this case, for user 101. I've looked for the same header for user 100, 101, 102 and 103, and the only one that doesn't have this header is 100.

## Q6. How many extensions were scanned in total?

I wasn't able to find the answer, I found something close to the answer but I haven't been able to find the answer. Please refer to another write-up for an explaination.

## Q7. There is a trace for a real SIP client. What is the corresponding user-agent? (two words, once space in between)

Use a grep regex to find all the User-Agent:

```bash
grep -Ei 'User-Agent:' log.txt|uniq
```

## Q8. Multiple real-world phone numbers were dialed. Provide the first 11 digits of the number dialed from extension 101?

As I don't know anything about SIP, I asked to ChatGPT how to find a dialed phone number. It answered me that you have to check `INVITE` request type. This command search for extension 101 with INVITE type.

```grep
grep -Ei 'From: "Unknown"<sip:101' log.txt -B8|grep INVITE
```

It returns 3 matchs:

```
INVITE sip:900114382089XXXX@honey.pot.IP.removed;transport=UDP SIP/2.0
INVITE sip:00112322228XXXX@honey.pot.IP.removed;transport=UDP SIP/2.0
INVITE sip:00112524021XXXX@honey.pot.IP.removed;transport=UDP SIP/2.0
```

We can forget the first one because it's not 11 digit long. I don't know why it's the last one, both of them have an INVITE request for extension 101. Maybe I forget something?

## Q9. What are the default credentials used in the attempted basic authentication? (format is username:password)

Search for 'password' in Wireshark: Edit > Find a packet (don't forget to search for a **string**). In the HTTP paquets, you will find the credentials.

## Q11. Which codec does the RTP stream use? (3 words, 2 spaces in between)

Filter the capture on `RTP` packet type and open one of them. There is no codec field but there is `payload type`, searching on "rtp codec" I found this page: [RTP payload formats - Wikipedia](https://en.wikipedia.org/wiki/RTP_payload_formats). In fact, the RTP payload format is related to the codec used:

> ITU-T G.711 PCM A-Law audio 64 kbit/s

It looks like a codec, we found it!

## Q12. How long is the sampling time (in milliseconds)?

Not really a DFIR question^^.

Go to [G.711 - Wikipedia](https://en.wikipedia.org/wiki/G.711), you will find that the sampling frequency is **8kHz**, 8000 times per second. To get the time between two samples, just calculate 1/8000 = 0.000125. As we want millisecond, multiply it by 1000 = **0.125**.

## Q13. What was the password for the account with username 555?

The capture contains HTTP packets which seems related to the VoIP system interface. I extracted the HTTP objects but without find the answer.

The hint 'take a look at sip_custom.conf' helped me a lot. I found packet number 1287 which contains many informations such as passwords:

```html
<form  name="section_form" method="post" action="?file=sip_custom.conf">

            <input type=hidden name="themd5" value="ee6207b44f73896fd799ad41c9e20947">

            <input type=hidden name="updateSection" value="sip_custom.conf">

            <h2>Edit: sip_custom.conf</h2>

            <textarea name="section_text" cols=100 rows="30" wrap="off">[555]
type=friend
username=555
secret=1234
```

## Q14. Which RTP packet header field can be used to reorder out of sync RTP packets in the correct sequence?

I found two header field:

- Sequence number

- Timestamp

I tried both, the answer is Timestamp, maybe Sequence number can be used for this feature too.

## Q15. The trace includes a secret hidden message. Can you hear it?

Wireshark has the tool, go to : `Telephony > RTP > RTP Player`.
