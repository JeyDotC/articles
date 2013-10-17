Comunicating Jax-ws client with WCF service with server sertificate
=====================================================

This article is about comunicating a jax-ws client with a WCF service using certificates having these conditions:

* The WCF binding is *wsHttpBinding*.
* WCF security is at *Message*.
* The library used in client side is *jax-ws*.
* The client have been generated with *WSIMPORT* ([Wsimport tutorial](http://www.mkyong.com/webservices/jax-ws/jax-ws-wsimport-tool-example/), [Wsimport in netbeans](https://netbeans.org/kb/docs/websvc/client.html)).

**NOTE:** In my case I used netbeans as my IDE, but it is not required, if anyone can reproduce these steps in eclipse and have a post about it, I'd like to add a link to that post.

## The current situation

One day your boss tells you "Johny, I need a WCF service and a java client" you creates them and everything goes good until he says, "I need certificates!" I gess this image illustrates the situation:

![Java and .NET don't want to talk :(](https://raw.github.com/JeyDotC/articles/master/image/interoperabilidad.png)

Once certificates come in play, a myriad of errors pops up in front of you, one of the worst ones is having your client indefinitely trying to talk to the service, no exceptions, zero explosions, nothing that can give you a clue on what is going on.

## The long road to the solution

### Step 1: Configure youre keystore and add it to your client

The first thing is to have your certificates in your project's keystore, there are several ways to do this, that depends on your certificates set. In this case I had a CA signed private certificate (.pfx) from which one can export a public certificate (.cer) the latter is the one we're interested in, to import it one can use this command:

```
keytool -keystore "path/to/mykeystore.jks" -importcert -alias some_alias -trustcacerts -file "path/to/myCertificate.cer" -keypass yourcertificate_password -storepass mykeystore_password
```

Once you have your keystore ready, you will need to add a reference to it in your webservice client. Netbeans users may refer to [this tutorial](https://metro.java.net/nonav/1.2/guide/Configuring_Keystores_and_Truststores.html#Configuring_the_Keystore_and_Truststore).

### Step 2: Install METRO, the latest version! 

Configuring my keystore wasn't enough, errors appeared here and there, if my memory doesn't fail, in that moment the client tried to send the messages ignoring any server side security configuration, also, those were the horrible moments of the client trying to talk to the server and sticking forever in it's futile attempt.

After a lot of research (mostly in stackOverflow), a promising solution appeared, the [Metro](https://metro.java.net/) library which is intended to mend faces between Java and .Net, one just need to reference it and it was supposed to do the rest. 

**Note:** Remember to remove any previous reference to jax-ws as that library is included with Metro.

 But then, a meaningless `NullPointerException` appeared, those were two or three days cursing certificates and viewing this stackOverflow [question](http://stackoverflow.com/questions/13849451/nullpointerexception-java-webservice-client-on-top-of-wcf-using-ws-security) unanswered. Then I realized that the library version (automatically added by netbeans)  was 2.0, updating to the 2.3 version changed the situation, a meaningful exception were thrown and then I could go on, never felt so happy to seeing an exception before!

### Step 3: Install BouncyCastle

The exception thrown was:

> Exception: "algorithm is not supported for key encryption java.security.NoSuchAlgorithmException: Cannot find any provider supporting RSA/ECB/OAEPPadding"

The answer were found [here](http://stackoverflow.com/questions/17207491/after-update-to-java7u25-from-java7u21-jax-ws-client-of-my-program-throws-cannot). Apparently, the standard Java installation was missing some encryption algorithms, so it was necessary to install them.

All I had to do was to download this library, add it to the project and add this line of code before calling the service:

```java
Security.addProvider(new BouncyCastleProvider());
```

### Step 4 (hopely the last): Install the "Java Cryptography Extension (JCE) Unlimited Strength Jurisdiction Policy Files"

Once the encryption algorithm obstacle were passed, the next problem appeared: 

> Java Security: Illegal key size or default parameters 

Man! after a week of problems one feels like Sisyphus!. Fortunately the solution were found by someone else and is [here](http://stackoverflow.com/questions/6481627/java-security-illegal-key-size-or-default-parameters).

Apparently the *Java Cryptography Extension (JCE) Jurisdiction Policy Files* that comes with the JRE have some kind of limit for something (I don't know what or where and I don't care), it was necessary to replace them with the *Unlimited Strength* version which can be downloaded [here](http://www.oracle.com/technetwork/java/javase/downloads/jce-6-download-429243.html) for Java 6 and [here](http://www.oracle.com/technetwork/java/javase/downloads/jce-7-download-432124.html) for Java 7, make sure to download the correct version for your JRE. 

To install them, go to your JRE installation folder and go to the '/lib/security' folder, backup the files inside it and replace them with the ones you just downloaded. Detailed instructions can be found in the README.txt file that comes with the download.

Praise the Lord! Java and .NET finally talked to each other after 8 or 9 days of agony :smile_cat:

## Summary

In order to comunicate Java and WCF one must:

. Configure the keystore and add it to the client.
. Install METRO and be sure to install the latest version.
. Install BouncyCastle and add the provider by calling `Security.addProvider`
. Install the "Java Cryptography Extension (JCE) Unlimited Strength Jurisdiction Policy Files" 

## TODO in this article

[ ] Reference a good article about keystore configuration.
[ ] Paste the WCF's web.config file to illustrate the wsBinding used in this case.
