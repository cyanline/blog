## Extracting Deleted Messages From An iPhone

date: March 20, 2014


### Introduction
During a recent investigation, CyanLine faced the task of recovering deleted iMessages from an iPhone 4. To make the task even more difficult than it already was, the iMessages had been deleted two months prior. Apple claims to not have access to these messages transmitted using their encrypted communication service, iMessage.  Therefore a subpoena won't be of any help here. 

CyanLine found traces of deleted messages that were in the SMS.db of the iPhone. The SMS.db is a SQLlite database file used by the iPhone to store incoming and outgoing SMS and iMessage messages.
Acquiring The Raw Bytes
We were provided with the device's pass code, which allowed us to view the devices storage in an unencrypted state, including the iMessages. In order to perform the raw acquisition, we needed to gain unrestricted access to the device's storage. To do this, the device had to be jailbroken. Next, we performed a raw acquisition of the device's storage. 
Using Cydia [1], we installed OpenSSH to allow for remote root access to the device. Next we logged into the device remotely using SSH. From here we requested and were granted super-user capabilities. Finally we used GNU's cat to read the raw bytes from '/dev/rdisk1d2'. The output was sent over the network via SSH, and finally written to an image file using GNU's dd. 

    cat /dev/rdisk1d2 | ssh 192.168.2.105 dd of=iphone-4.dd

The raw acquisition format obtains a forensically sound copy of all bytes on the storage medium. Note, this includes deleted files or file fragments, and data stored in slack space.

### The Search
Fortunately for this particular investigation, after the messages were deleted, searches in Spotlight search returned cached message content from the deleted messages. To make things easier screen shots were taken of various deleted messages.

The screen shots provided us the opportunity perform keyword searches for message text. In order to do so, we needed to extract the SMS.db, extract the readable strings from the SMS.db, and perform a search for keywords. The iPhone 4, iOS version 6.1.3 stores messages in a file called SMS.db. 

We extracted the SMS.db from the raw acquisition file. We then generated a list of strings in the SMS.db using GNU's strings.  Finally, we performed a search on the strings in the database using our extracted SMS.db strings text and GNU's grep. 

Search results revealed that message fragments were still in the SMS.db file.

### The Analysis
Sms.db is a sqlite3 database.  There are two tables of interest to us in the database; message and handle.  The message table stores, among other things, the contents of the message, the handle id, the date read, the date sent, and the service (SMS or iMessage).  The handle table stores information about the other party of the message.

The goal is to identify the structure of the data and extract deleted message components from the raw data.

### Date and Time Formatting
The sms.db stores dates using an integer.  The date integer is the number of seconds since 01/01/2001 00:00:00 which is known as Mac Absolute time.  The difference between Mac absolute time and Unix epoch time is 978307200 seconds. Adding 978307200 to the Mac Absolute time value gives a much easier number to work with. 

### Methodology
Loading the sms.db up in a sqlite viewer makes it easy to inspect data that has not been deleted, but to extract data that has been deleted we must inspect the raw data.  As an example for this post we confirmed that the text "thats a great idea lets meet today I want to see you" was actively present in the database.  A search through the raw data showed that it existed in the sms.db file.  The sms.db contains the database schema (Figure 3).  Using GNU's xxd we were able to view the raw data contained in the file.  The following command was used to extract the raw data:

    xxd sms.db | grep --after-context=20 --before-context=10 "I want to"

In Figure 1.1 below, the highlighted portion is the data that corresponds to the account\_guid column of the message table.   Immediately previous to that is the account phone number and service that was used to transmit the message.  The phone number corresponds to the number used by the examined iPhone.  As can be seen in the sms.db schema (Figure 3) the date column follows the account\_guid column (Figure 1.2).

The 4 nibbles[2] highlighted in Figure 1.2 represent the date column.  Converted from hexidecimal to decimal the value of the date column is 397537001.  As noted earlier this is in Mac Absolute Time, therefore adding 978307200 to that value results in 1375844201 which is formatted in Unix Epoch time.  Formatted in a more human readable form 1375844201 results in Tue, Aug 6 22:56:41 EDT 2013.  The 8 nibbles following the date represent the date\_read and date\_delivered columns respectively.  The 3 dates all appear to be the same.  And seem to only be independently set when the service is iMessage. 

The sms.db contains a table called handles which has a relationship with the message table. The handle table contains information for the second party in the conversation.  This includes phone number and service and the phone number uses for communication (SMS or iMessage).

The handle id corresponding to the message in question is highlighted in Figure 1.3.  Converting the handle id to decimal results in 3.  The handle id 3, corresponds to the number +1 (555) 765-4321 using the iMessage service.  1 (555) 765-4321 is stored in the contact database under the name John Smith.
The following is an example of a non-deleted message which is currently stored in the sms.db file.

##### Figure 1 - Raw Data (Output from the command above)

    017e520: 6f64 6179 2069 2077 616e 7420 746f 2073 oday I want to s
    017e530: 6565 2079 6f75 2103 040b 7374 7265 616d ee you!...stream
    017e540: 7479 7065 6481 e803 8401 4084 8484 194e typed.....@....N
    017e550: 534d 7574 6162 6c65 4174 7472 6962 7574 SMutableAttribut
    017e560: 6564 5374 7269 6e67 0084 8412 4e53 4174 edString....NSAt
    017e570: 7472 6962 7574 6564 5374 7269 6e67 0084 tributedString..
    017e580: 8408 4e53 4f62 6a65 6374 0085 9284 8484 ..NSObject......
    017e590: 0f4e 534d 7574 6162 6c65 5374 7269 6e67 .NSMutableString
    017e5a0: 0184 8408 4e53 5374 7269 6e67 0195 8401 ....NSString....
    017e5b0: 2b35 7468 6174 7320 6120 6772 6561 7420 +5thats a great 
    017e5c0: 6964 6561 206c 6574 7320 6d65 6574 2074 Idea lets meet t
    017e5d0: 6f64 6179 2069 2077 616e 7420 746f 2073 oday I want to s
    017e5e0: 6565 2079 6f75 2186 8402 6949 0135 9284 ee you!...iI.5..
    017e5f0: 8484 0c4e 5344 6963 7469 6f6e 6172 7900 ...NSDictionary.
    017e600: 9584 0169 0192 8498 981d 5f5f 6b49 4d4d ...i......__kIMM
    017e610: 6573 7361 6765 5061 7274 4174 7472 6962 essagePartAttrib
    017e620: 7574 654e 616d 6586 9284 8484 084e 534e uteName......NSN
    017e630: 756d 6265 7200 8484 074e 5356 616c 7565 umber....NSValue
    017e640: 0095 8401 2a84 9b9b 0086 8686 0a69 4d65 ....*........iMe
    017e650: 7373 6167 6570 3a2b 3135 3535 3132 3334 ssagep:+15551234
    017e660: 3536 3741 4638 3443 3832 362d 3739 4338 567AF4C826-79C8
    017e670: 2d34 4646 342d 4235 4531 2d39 3537 3630 -4FF4-B5E1-95760
    017e680: 3541 3932 3536 3517 b1ee e917 b1ee e917 5A92565.........

###### Figure 1.1

![Figure 1.1](./images/articles/imessage_figure1_1.png)

###### Figure 1.2

![Figure 1.2](./images/articles/imessage_figure1_2.png)

###### Figure 1.3

![Figure 1.3](./images/articles/imessage_figure1_3.png)

###### Figure 1.4

![Figure 1.4](./images/articles/imessage_figure1_4.png)

Figure 2 is an example of a deleted message fragment.  Using the method above we can identify the content of the message (Fiogure 2.1) and the handle id of the second party (Figure 2.2). 

##### Figure 2 - Sample Deleted Message

    01651c0: 6572 6520 6172 6529 796f 752c 2077 6520 ere are you, we 
    01651d0: 6e65 6564 2074 6f20 7461 6c6b 2e3b 040b need to talk.;..
    01651e0: 7374 7265 616d 7479 7065 6481 e803 8401 streamtyped.....
    01651f0: 4084 8484 124e 5341 7474 7269 6275 7465 @....NSAttribute
    0165200: 6453 7472 696e 6700 8484 084e 534f 626a dString....NSObj
    0165210: 6563 7400 8592 8484 8408 4e53 5374 7269 ect.......NSStri
    0165220: 6e67 0194 8401 2b1f 7768 6572 6520 6172 ng....+.where ar
    0165230: 6520 796f 752c 2077 6520 6e65 6564 2074 e you, we need t
    0165240: 6f20 7461 6c6b 2e86 8402 6949 011f 9284 o talk....iI....
    0165250: 8484 0c4e 5344 6963 7469 6f6e 6172 7900 ...NSDictionary.
    0165260: 9484 0169 0192 8496 961d 5f5f 8232 935d ...i......__.2.]
    0165270: 2800 5511 0800 0100 0082 6e01 0813 1155 (.U.......n....U
    0165280: 0804 0408 0909 0808 0808 0808 0908 0808 ................

###### Figure 2.1

![Figure 2.1](./images/articles/imessage_figure2_1.png)

###### Figure 2.2

![Figure 2.2](./images/articles/message_figure2_2.png)

##### Figure 3 - Database schema
<table>
<tr><td>SQL Column</td><td>Value</td></tr>
<tr><td>ROWID</td> <td>INTEGER PRIMARY KEY AUTOINCREMENT</td></tr>
<tr><td>guid</td><td>TEXT UNIQUE NOT NULL</td></tr><tr>
<tr><td>text</td><td> TEXT</td></tr>
<tr><td>replace</td><td> INTEGER DEFAULT 0</td></tr>
<tr><td>service_center</td><td> TEXT</td></tr>
<tr><td>handle_id</td><td> INTEGER DEFAULT 0</td></tr>
<tr><td>subject</td><td> TEXT</td></tr>
<tr><td>country</td><td> TEXT</td></tr>
<tr><td>attributedBody</td><td> BLOB</td></tr>
<tr><td>version</td><td> INTEGER DEFAULT 0</td></tr>
<tr><td>type</td><td> INTEGER DEFAULT 0</td></tr>
<tr><td>service</td><td> TEXT</td></tr>
<tr><td>account</td><td> TEXT</td></tr>
<tr><td>account_guid</td><td> TEXT</td></tr>
<tr><td>error</td><td> INTEGER DEFAULT 0</td></tr>
<tr><td>date</td><td> INTEGER</td></tr>
<tr><td>date_read</td><td> INTEGER</td></tr>
<tr><td>date_delivered</td><td> INTEGER</td></tr>
<tr><td>is_delivered</td><td> INTEGER DEFAULT 0</td></tr>
<tr><td>is_finished</td><td> INTEGER DEFAULT 0</td></tr>
<tr><td>is_emote</td><td> INTEGER DEFAULT 0</td></tr>
<tr><td>is_from_me</td><td> INTEGER DEFAULT 0</td></tr>
<tr><td>is_empty</td><td> INTEGER DEFAULT 0</td></tr>
<tr><td>is_delayed</td><td> INTEGER DEFAULT 0</td></tr>
<tr><td>is_auto_reply</td><td> INTEGER DEFAULT 0</td></tr>
<tr><td>is_prepared</td><td> INTEGER DEFAULT 0</td></tr>
<tr><td>is_read</td><td> INTEGER DEFAULT 0</td></tr>
<tr><td>is_system_message</td><td> INTEGER DEFAULT 0</td></tr>
<tr><td>is_sent</td><td> INTEGER DEFAULT 0</td></tr>
<tr><td>has_dd_results</td><td> INTEGER DEFAULT 0</td></tr>
<tr><td>is_service_message</td><td> INTEGER DEFAULT 0</td></tr>
<tr><td>is_forward</td><td> INTEGER DEFAULT 0</td></tr>
<tr><td>was_downgraded</td><td> INTEGER DEFAULT 0</td></tr>
<tr><td>is_archive</td><td> INTEGER DEFAULT 0</td></tr>
<tr><td>cache_has_attachments</td><td> INTEGER DEFAULT 0</td></tr>
<tr><td>cache_roomnames</td><td> TEXT</td></tr>
<tr><td>was_data_detected</td><td> INTEGER DEFAULT 0</td></tr>
<tr><td>was_deduplicated</td><td> INTEGER DEFAULT 0</td></tr>
</table>

### Conclusion
We were able to recover date/time information for some of these messages, as well as message text, the other parties db handle id. We were not able to recover the database row which ties the handle id to a phone number, full dates and times for some messages, and wee unable to determine whether the message we recovered was sent or received by our client. 

[1] Cydia is an APT front end for the iPhone develop by Jay Freeman (http://www.saurik.com/id/1)

[2] In computing, a nibble is a four-bit aggregation, or half an octet.
