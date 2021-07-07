---
title: Sending App Data Over SMS on Android
date: 2013-12-22
tags: [SMS]
---

For some time now I've been working on [an Android app](https://github.com/jasonrogena/ngombe_planner-android) that is going to be used by small scale farmers in East Africa. We expect that most of the farmers will not have an internet connection. Hell, most of them won't have android phones. But that's a story for another day. So, for a proof of concept, the app I'm making should roll back to sending data using SMS when no HTTP connection is available. Here some notes on how to do it:


### Architecture

You'd want to follow an architectural styl, such as [REpresentational State Transfer](http://www.ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm), that explicitly defines the types and structures of messages to be sent between Client and Server. This is very important because you want to keep the intereactions as clean as possible.

In this instance, I did not go the REST way. However, I made up a simple 'Protocol' defining intereactions between the Client (Android App) and the Server. Here it goes: *All messages sent between the Server and the Client are either json strings or integers. The integers represent predefined acknowledgement and error codes*.


### Make it work well over HTTP

As much as I would have wanted to start implementing the SMS functionality from the get go I had to first get the requirements right first. Blame that on the client. So I built, we reviewed. Then I built some more and we reviewed again until all the features were working (although all interactions with the server were through a http connection).

One thing, however, that I had to do was to make sure the app was as modular as possible. Each important function (eg handling notifications or interaction with the server) was handled by one class. For instance all interactions with the server are handled by [DataHandler.java](https://github.com/jasonrogena/ngombe_planner-android/blob/master/NgombePlanner/src/main/java/org/cgiar/ilri/np/farmer/backend/DataHandler.java).


### The SMS Server

When everything was working fine over HTTP and I was sure it was time to implement message transmission over SMS, we had to choose Server Side Software for handling SMSs. My boss chose [Kannel](http://www.kannel.org). I wouldn't have chosen anything else. Kannel is free and open. Honestly, I would not recommend Ozeki to anyone. Kannel is a bit hard to configure but once you get things running it is bliss.

Here's [another post](https://jasonrogena.github.io/2014/01/18/kannel-and-the-huawei-e160.html) I wrote on configuring Kannel.

We have two servers in the office, one has all the PHP scripts that interact with the Android app and the other runs Kannel. To make things less complicated I'll call the server running the PHP scripts Azizi and the one running Kannel KServer. KServer has a Huawei E160 HSPA modem plugged into it. Not the best modem but it works. The KServer acts like a proxy, re-routing SMSs from the Andriod app to Azizi and routing responses from Azizi to the Android app via SMS.


### Rolling back to SMS on the Android App

Sending SMSs from the Android app is pretty simple. When DataHandler is called, it checks whether it can connect to Azizi over HTTP. It sends the data using SMS if it cannot directly connect to Azizi. Code snippet below shows the method in DataHandler responsible for determining if data should be sent using HTTP or SMS.

```java
public static String sendDataToServer(Context context, String jsonString, String appendedURL, booleanwaitForResponse) {
    String response;
    if(checkNetworkConnection(context)){
        response = sendDataUsingHttpConnection(jsonString, appendedURL);
    }
    else{
        response = sendDataUsingSMS(context, jsonString, appendedURL, waitForResponse);
    }
    return response;
}
```


The sendDataUsingSMS method is pretty simple. I first created a new [SmsManager](http://developer.android.com/reference/android/telephony/gsm/SmsManager.html) Object. Note from the code snippet below that I intend to send multipart SMSs instead of regular SMSs. This is because the text being sent to the server might be greater than 150 characters (limit for one SMS).

```java
SmsManager smsManager = SmsManager.getDefault();
String message = appendedURL+SMS_DELIMITER+jsonString;
ArrayList<String> multipartMessage = smsManager.divideMessage(message);
int noOfParts = multipartMessage.size();
```

Then I initialize three [BroadcastReceivers](http://developer.android.com/reference/android/content/BroadcastReceiver.html). Basically, what Broadcast receivers do is listen for broadcasted Intents. For instance when an SMS is sent, Android broadcasts the Intent. All BroadcastReceivers that registered to listen for that particular intent will get the broadcast message from Android.

```java
MistroSMSSentReceiver mistroSMSSentReceiver = new MistroSMSSentReceiver(message, noOfParts);    
context.registerReceiver(mistroSMSSentReceiver, new IntentFilter(ACTION_SMS_SENT));
MistroSMSDeliveredReceiver mistroSMSDeliveredReceiver = new MistroSMSDeliveredReceiver(message, noOfParts);
context.registerReceiver(mistroSMSDeliveredReceiver, new IntentFilter(ACTION_SMS_DELIVERED));
if(waitForResponse){//method will be waiting for a response sms from the server
    MistroSMSReceiver mistroSMSReceiver = new MistroSMSReceiver();
    IntentFilter smsReceivedIntentFilter = new IntentFilter(ACTION_SMS_RECEIVED);
    smsReceivedIntentFilter.addAction("android.provider.Telephony.SMS_RECEIVED");
    context.registerReceiver(mistroSMSReceiver, smsReceivedIntentFilter);
}
```

As I was saying, I initialize three BroadcastReceivers:

 1. MistroSMSSentReceiver - That listens for when an SMS is sent
 2. MistroSMSDeliveredReceiver - That listens for when an SMS is delivered
 3. MistroSMSReceiver - That listens for when an SMS is received

All three BroadcastReceivers are inner classes in DataHandler.java that extend BroadcastReceiver.

You might notice from the code snippet above that; MistroSMSSent Receiver will be listening for an intent with the *ACTION_SMS_SENT* message, MistroSMSDeliveredReceiver will be listening for an intent with the *ACTION_SMS_DELIVERED* message, and MistroSMSReceiver will be listening for an intent with the message *ACTION_SMS_RECEIVED*.

Then I register the MistroSMSSentReceiver and MistroSMSDeliveredReceiver for each of the SMS fragments to be sent.

```java
ArrayList<PendingIntent> sentPendingIntents = new ArrayList<PendingIntent>();
ArrayList<PendingIntent> deliveredPendingIntents = new ArrayList<PendingIntent>();
for(int i = 0; i<noOfParts; i++){
    PendingIntent newSentPE = PendingIntent.getBroadcast(context, 0, new Intent(ACTION_SMS_SENT), 0);
    sentPendingIntents.add(newSentPE);
    PendingIntent newDeliveredPE = PendingIntent.getBroadcast(context, 0, new Intent(ACTION_SMS_DELIVERED), 0);
    deliveredPendingIntents.add(newDeliveredPE);
}
```

Finally I send the SMS and wait for KServer to send back the response. It's that simple!

```java
smsManager.sendMultipartTextMessage(SMS_SERVER_ADDRESS, null, multipartMessage, sentPendingIntents, deliveredPendingIntents);
 
long startTime = System.currentTimeMillis();
if(waitForResponse){
    while(true){
        long currTime = System.currentTimeMillis();
        long timeDiff = currTime - startTime;
        if(getSharedPreference(context, SP_KEY_SMS_RESPONSE,"").length()>0){
            return getSharedPreference(context, SP_KEY_SMS_RESPONSE,"");
        }
        else if(timeDiff>SMS_RESPONSE_TIMEOUT){
            Log.w(TAG, "SMS response timeout exceeded");
            return null;
        }
    }
}
else{
    return ACKNOWLEDGE_OK;
}    smsManager.sendMultipartTextMessage(SMS_SERVER_ADDRESS, null, multipartMessage, sentPendingIntents, deliveredPendingIntents);
 
long startTime = System.currentTimeMillis();
if(waitForResponse){
    while(true){
        long currTime = System.currentTimeMillis();
        long timeDiff = currTime - startTime;
        if(getSharedPreference(context, SP_KEY_SMS_RESPONSE,"").length()>0){
            return getSharedPreference(context, SP_KEY_SMS_RESPONSE,"");
        }
        else if(timeDiff>SMS_RESPONSE_TIMEOUT){
            Log.w(TAG, "SMS response timeout exceeded");
            return null;
        }
    }
}
else{
    return ACKNOWLEDGE_OK;
}
```

As you can see from the code above, waiting for KServer's response is an endless loop that terminates only when the ENTIRE response is received (put inside the SharedPreference *SP_KEY_SMS_RESPONSE*) or when it timesout.

I say ENTIRE because the server might respond with a large message that will be split into more than one SMSs. What MistroSMSReceiver does to handle this is:

 - It listens for when an SMS is received.
 - If an SMS is received and the SMS is from KServer, it first gets the value of the SharedPreference *SP_KEY_SMS_CACHE* (which has a default value of an empty string), then it appends the SMS to the value of *SP_KEY_SMS_CACHE* then:
 - IT checks for whether the resultant message is a valid json string or if it one of the predefined flags. 
 - If so the message is saved in the SharedPreference *SP_KEY_SMS_RESPONSE*.
 - Otherwise the message is cached in the SharedPreference *SP_KEY_SMS_CACHE* while MistroSMSReceiver waits for the rest of the SMS fragments.


### What's next

There are still some things that I need to figure out:

 1. How to ensure the order of SMSs (making up a split message) is maintained when reconstructing a message.
 2. How to compress messages before sending them over SMS. Here's a [nice post](http://www.davidhampgonsalves.com/Compress-JSON.js/) how to do it.
