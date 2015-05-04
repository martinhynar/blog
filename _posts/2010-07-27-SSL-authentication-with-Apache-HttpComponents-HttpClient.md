---
layout: post
title: "SSL authentication with Apache HttpComponents HttpClient"
abstract: "Setting up secure SSL based communication in Java using Apache HttpCommons"
tags: clojure hiccup
---

This post shows how to set up secure communication in an application that makes use of [Apache HttpComponents](http://hc.apache.org/) library.
The goal is to have a client application that shall communicate with server requiring certificate based authentication
and validate that the server is the one it shall be (we authenticate it as well). This is called mutual authentication.



## The big picture

Both parties have the following items available

+ own private key (self-generated) that is used to encrypt the payload sent to the other party
+ public key of the other party

### Prepare key and trust stores

First of all, we need to prepare our "stores". There are two. The first one, trust store, is the file in which
certificates we trusting to are stored. This store is used to verify validity of the counterpart's certificate. The
second one, the key store is used to store own keys that are used to encrypt the payload.

**Note:** In this post, I am going to use separate trust and key stores besides those that are used by default. The
usage of the default stores is well described around the Internet.

First of all, we want to generate self-signed key pair. For this we will use keytool application that gets shipped
together with your JDK. To generate the keys, run this command:

`keytool -genkey -alias server -keyalg RSA –keystore keystore –storetype JKS -storepass secret`

You will be asked few simple questions that I omit here. This generates keystore file with one entry. You can list the
contents of the keystore using following command:

`keytool -list -keystore keystore -storepass secret`

Now, it is necessary to extract the public part of the pair and hand it over to the other party. To make the extraction,
run the following command:

`keytool -export -alias server -rfc -file servercert.pub -keystore keystore -storepass secret`

This file then needs to be imported by the other party into its trust store so that it is trusted party. To import the
file, run the following command:

`keytool -import -alias server -file servercert.pub -storetype jks -keystore truststore -storepass secret`

Now the public key is imported into trust store. You can list the contents using similar command as used before to list
key store contents.

The same procedure needs to be run for the other party (lets call it client). The final state after the key stores setup
is the following

**Client:**

+ keystore - contains own private key (self-generated)
+ truststore - contains imported public key of the server

**Server**

+ keystore - contains own private key (self-generated)
+ truststore - contains imported public key of the server


###What the parameters of keytool application mean

You might ask what the parameters of the command line mean, let me explain that:

+ `-genkey` - instructs keytool to generate keypair
+ `-keyalg` - simply, which algorithm shall be used
+ `-storetype` - type of the store file (JKS or PKCS12)
+ `-list` - list the contents of the store
+ `-keystore` - the name of the file that is considered a key store (without path it takes actual location)
+ `-storepass` - password for the keystore
+ `-import` - switch that tells to the tool that we are going to import something
+ `-export` - switch that tells to the tool that we are going to export something
+ `-rfc` - tells to use RFC style of the exported certificate (it is then "readable")
+ `-file` - this tells the tool from/to where we are importing/exporting
+ `-alias` - identifies a nick name of the certificate

Now we have both trust and key stores prepared. The next step is to incorporate them into the application. The following
method snippet creates DefaultHttpClient instance. The DefaultHttpClient is the class from Apache's HttpClient library
that wraps communication with the counterpart. On this instance you are running all the request, etc.

<script src="https://gist.github.com/martinhynar/06ec843655df89c9d852.js"></script>
