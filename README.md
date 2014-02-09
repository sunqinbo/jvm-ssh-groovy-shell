# What is this?

This is an integration between the [Mina SSH server](http://mina.apache.org/sshd-project/) and the [Groovy Shell](http://groovy.codehaus.org/Groovy+Shell). Using this integration, you can quickly add an SSH server to your app and be able to remotely access a Groovy shell that lets you interactively run code inside your running JVM. In other words, it's a remotely accessible REPL that's inside your JVM, thereby allowing you to inspect what's happening inside your app without having to write JMX or other ways of externally exposing management functionality.

# Demo

There's a demo app included. In your clone of the repo, run:
```
./gradlew :demo-app:run
```

If you want to run the demo, you'll need Java 8 since it uses the shiny new JSR-310 classes. (The rest of the library doesn't require Java 8.)

This will start up a simple HTTP service on `localhost:8080` that exposes a `/nowUtc` endpoint that spits out a timestamp in ISO-8601. Here it is in action along with the `date` command to show that it's doing what it should:

```
% TZ=UTC date --iso-8601=seconds && curl http://localhost:8080/nowUtc
2014-02-09T22:47:43+0000
2014-02-09T22:47:43.059Z
```

To show what you might do with the Groovy shell, the time is generated by the `TimeSource` class that has a `setSecondsOffset(int seconds)` method. Unsurprisingly, this shoves the timestamps forwards or backwards relative to the correct time. Let's make time be wrong by 30 seconds (using Groovy property syntax, because we can). The demo app's groovy shell is preconfigured to have a `timeSource` binding to the relevant `TimeSource` instance:

```
% ssh -p 10222 localhost
groovy:000> timeSource
===> com.palominolabs.ssh.groovy.demo.TimeSource@6013754e
groovy:000> timeSource.secondsOffset = 30
===> 30
```

(If you try this yourself, you'll see the line wraps aren't working right yet -- it's a known issue.)

You should see a log line:
```
2014-02-09 14:50:55,712 [pool-1-thread-1] INFO  MDC[] c.p.ssh.groovy.demo.TimeSource - Setting offset to 30
```

And now, to check that it worked:

```
% TZ=UTC date --iso-8601=seconds && curl http://localhost:8080/nowUtc
2014-02-09T22:51:14+0000
2014-02-09T22:51:44.300Z
```

Sure enough, the second timestamp is 30 seconds fast.

Obviously this is a contrived example, but it should be clear how useful it is to be able to interactively execute arbitrary code inside a running JVM.

# Security

In the demo above, you didn't have to provide a password because the demo app defaults to allowing all connections. However, that's not realistic for real-world use, so there's also a `PublickeyAuthenticator` implementation provided ([`AuthorizedKeysPublickeyAuthenticator`](https://github.com/palominolabs/ssh-groovy-shell/blob/master/authorized-keys-ssh-authenticator/src/main/java/com/palominolabs/ssh/auth/publickey/AuthorizedKeysPublickeyAuthenticator.java)) that uses ssh's `authorized_keys` format to define the public keys to allow. Currently, it supports RSA and DSA keys.

The demo app supports public key authentication; you simply need to tell it what authorized_keys file to use. If you already have ssh keys set up, you can point the demo app at your current public key to quickly test it.

```
./gradlew -DDEMO_SSH_AUTHORIZED_KEYS=$HOME/.ssh/id_rsa.pub :demo-app:run
```

In a real deployment, you'd probably use a file containing many such public keys, like the `authorized_keys` file you might already have on your servers. For now ,
using your own public key will of course allow your corresponding private key to work.

Check the source of [`DemoMain`](https://github.com/palominolabs/ssh-groovy-shell/blob/master/demo-app/src/main/java/com/palominolabs/ssh/groovy/demo/DemoMain.java) to see how to hook up this style of authentication.
