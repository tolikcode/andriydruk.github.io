+++
date = "2015-09-13T22:51:10+03:00"
description = ""
draft = false
title = "Wrap Annoying APIs With RxJava"
onmain = false

+++

[Reactive Extensions (also known as ReactiveX or Rx)](http://reactivex.io/intro.html) is a library, that brings [FRP](https://en.wikipedia.org/wiki/Functional_reactive_programming) programming paradigm to different platforms. And it's a breath of fresh air for android developers in Java 6 desert. A lot of articles about RxJava concept and common usage have already been written and I don't want to explain it again. That's why I would like to publish my example of RxJava usage in Android just to show you how you can wrap Java APIs. In this article I will describe a process of Java API wrapping for Apple’s DNSSD library (which in my opinion is quite an annoying API). 

<!--more-->

DNSSD library's public API is a Singleton that provides static methods implementing basic operations such as searching or advertising. All operations work asynchronously and their results are returned to you in callbacks from a background thread. It's a usual Java API. What's wrong with it? Let's imagine a basic usage scenario of services search in a network: 

* Browse service
* Resolve service
* Query records
* Update UI

That means that you should start `browse` operation with a `BrowseCallback`. In this callback you start `resolve` with a `ResolveCallback`, and than inside `ResolveCallback` you start `queryRecords` with a `QueryRecordsCallback`. After that you send a message to the main thread, where you update your UI. This turns into 'callback hell'. Real problems arise when a user leaves your app before you've got any results from this chain of callbacks, and you have to unsubscribe all listeners and to cancel all operations. Another big headache is exceptions handling from all operations and forwarding corresponding messages to the UI.

Ok, let's wrap this API with RxJava.

**Intro**
------------- 

If you aren't familiar with RxJava, I highly recommend you to start from the [official documentation](http://reactivex.io/intro.html). In this article I'm not going to explain RxJava operators, only examples of usage.

Ok, firstly, let's take a look at DNSSD Apple API:

~~~java
abstract public class DNSSD{
    public static DNSSDService  browse( int flags, int ifIndex, String regType, 
        String domain, BrowseListener listener) throws DNSSDException;

    public static DNSSDService  resolve( int flags, int ifIndex, String serviceName, 
        String regType, String domain, ResolveListener listener) throws DNSSDException;

    public static DNSSDService  queryRecord( int flags, int ifIndex, String serviceName, 
        int rrtype, int rrclass, QueryListener listener) throws DNSSDException;
}

public interface BrowseListener extends BaseListener
{
    /** Called to report discovered services.<P> */
    void    serviceFound(DNSSDService browser, int flags, int ifIndex,
                         String serviceName, String regType, String domain);

    /** Called to report services which have been deregistered.<P> */
    void    serviceLost(DNSSDService browser, int flags, int ifIndex,
                        String serviceName, String regType, String domain);


}

public interface ResolveListener extends BaseListener
{
    /** Called when a service has been resolved.<P> */
    void    serviceResolved(DNSSDService resolver, int flags, int ifIndex, String fullName,
                            String hostName, int port, TXTRecord txtRecord);
}

public interface QueryListener extends BaseListener
{
    /** Called when a record query has been completed. Inspect flags 
        parameter to determine nature of query event.<P> */
    void    queryAnswered(DNSSDService query, int flags, int ifIndex, String fullName,
                          int rrtype, int rrclass, byte[] rdata, int ttl);
}

public interface BaseListener
{
    /** Called to report DNSSD operation failures.<P> */
    void    operationFailed(DNSSDService service, int errorCode);
}
~~~

Every method returns a `DNSSDService` object, that contains only one method to stop this operation.

~~~java
public interface DNSSDService
{
    /**
    Halt the active operation and free resources associated with the DNSSDService.<P>

    Any services or records registered with this DNSSDService will be deregistered. Any
    Browse, Resolve, or Query operations associated with this reference will be terminated.<P>
    */
    void stop();
} 
~~~

I've written an interface for `DNSSDService`s creation  and a general method for `Observable`s creation.

~~~java
public interface DNSSDServiceCreator<T>{
    DNSSDService getService(Subscriber<? super T> subscriber) throws DNSSDException;
}

private <T> Observable<T> createObservable(DNSSDServiceCreator<T> mCreator){
    final DNSSDService[] mService = new DNSSDService[1];
    return Observable.create(new OnSubscribe<T>() {
        @Override
        public void call(Subscriber<? super T> subscriber) {
            if (!subscriber.isUnsubscribed()) {
                mContext.getSystemService(Context.NSD_SERVICE);
                try {
                    mService[0] = mCreator.getService(subscriber);
                } catch (DNSSDException e) {
                    e.printStackTrace();
                    subscriber.onError(e);
                }
            }
        }
    }).doOnUnsubscribe(() -> {
        if (mService[0] != null) {
            mService[0].stop();
            mService[0] = null;
        }
    });
}
~~~

I use a trick with `final DNSSDService[] mService = new DNSSDService[1]` to save `DNSSDService` instance from [`OnSubscribe`](http://reactivex.io/RxJava/javadoc/rx/Observable.OnSubscribe.html) callback and to stop it inside [`doOnUnsubscribe`](http://reactivex.io/documentation/operators/do.html) operator.

Before implementing basic operations I've created a `BonjourService` class. The main idea is to create a chain of observables that will operate with this class:

~~~java
public class BonjourService {

    public final int flags;
    public final int ifIndex;
    public final String serviceName;
    public final String regType;
    public final String domain;

    public Map<String, String> dnsRecords = new ArrayMap<>();
    public String hostname;
    public int port;

    public BonjourService(int flags, int ifIndex, String serviceName, String regType, 
    	String domain) {
        this.flags = flags;
        this.ifIndex = ifIndex;
        this.serviceName = serviceName;
        this.regType = regType;
        this.domain = domain;
    }
}
~~~

**Browse** 
------------- 
Ok, the first operation I've implemented is `browse`:

~~~java
public static Observable<BonjourService> browse(final String regType, final String domain) {
    return INSTANCE.createObservable(subscriber -> DNSSD.browse(0, DNSSD.ALL_INTERFACES, 
        regType, domain, new BrowseListener(subscriber)));
}

private static class BrowseListener implements com.apple.dnssd.BrowseListener{
    private Subscriber<? super BonjourService> mSubscriber;

    private BrowseListener(Subscriber<? super BonjourService> subscriber){
        mSubscriber = subscriber;
    }

    @Override
    public void serviceFound(DNSSDService browser, int flags, int ifIndex, String serviceName, 
            String regType, String domain) {
        if (mSubscriber.isUnsubscribed()){
            return;
        }
        mSubscriber.onNext(new BonjourService(flags, ifIndex, serviceName, egType, domain));
    }

    @Override
    public void serviceLost(DNSSDService browser, int flags, int ifIndex, String serviceName, 
            String regType, String domain) {
        if (mSubscriber.isUnsubscribed()){
            return;
        }
        mSubscriber.onNext(new BonjourService(flags | BonjourService.DELETED, ifIndex, 
            serviceName, regType, domain));
    }

    @Override
    public void operationFailed(DNSSDService service, int errorCode) {
        if (mSubscriber.isUnsubscribed()){
            return;
        }
        mSubscriber.onError(new RuntimeException("DNSSD browse error: " + errorCode));
    }
}
~~~

`browse` is an endless operation. That's why it never calls `onComplete` method on a `Subscriber`. Every time when DNSSD library calls a `serviceFound` callback I emit a `new BonjourService` object. Inside a `serviceLost` callback I do the same, but with additional flag `BonjourService.DELETED`. That's why `Subsriber` should always check this flag before updating of the UI.

Now look at the usage of my `browse` method:

~~~java
mSubscription = RxDNSSD.browse(mReqType, mDomain)
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe(bonjourService -> {
        if ((bonjourService.flags & BonjourService.DELETED) != BonjourService.DELETED) {
            mAdapter.add(bonjourService);
        } else {
            mAdapter.remove(bonjourService);
        }
        mAdapter.notifyDataSetChanged();
    }, throwable -> {
        Log.e("DNSSD", "Error: ", throwable);
        showErrorMessage(throwable);
    });
~~~

I don't use `subscribeOn`, because DNSSD will create its own thread and it's not necessary to call all DNSSD API methods from a background thread. However, I always use `observeOn`, because all callbacks from DNSSD API will be called from a background thread.

**Resolve** 
-------------

Next operation is `resolve`. I use this operation to resolve target host service name, port number, and a txt record. 
Unlike `browse` operation this one will return [`Transformer`](https://github.com/ReactiveX/RxJava/wiki/Implementing-Your-Own-Operators), that transforms an `Observable` of `BonjourService` into an `Observable` of resolved `BonjourService`. 

First of all I've applied a [`flatMap`](http://reactivex.io/documentation/operators/flatmap.html) to income `Observalbe`. Then in map's function I check if `BonjourService` contains `DELETED` flag then I return `Observable.just(bs)` (maybe the information about this `BonjourService` will be usefull for a `Subsriber`). 
In another case I return new `Observable` that will query record from DNSSD API.

~~~java
public static Observable.Transformer<BonjourService, BonjourService> resolve() {
    return observable -> observable.flatMap(bs -> {
        if ((bs.flags & BonjourService.DELETED) == BonjourService.DELETED) {
            return Observable.just(bs);
        }
        return INSTANCE.createObservable(subscriber -> DNSSD.resolve(bs.flags, bs.ifIndex, 
            bs.serviceName, bs.regType, bs.domain, new ResolveListener(subscriber, bs)));
    });
}
~~~

If the operation finishes successfully, I will put a target host, port number, and a txt record to `BonjourService` object and emit it to a `Subscriber`. Unlike `browse` operation this one is not endless, that's why after emitting I call `onComplete`. In the case of error, I forward this error to a `Subscriber`.

~~~java
private static class ResolveListener implements com.apple.dnssd.ResolveListener{
    private Subscriber<? super BonjourService> mSubscriber;
    private BonjourService mBonjourService;

    private ResolveListener(Subscriber<? super BonjourService> subscriber, 
            BonjourService service){
        mSubscriber = subscriber;
        mBonjourService = service;
    }

    @Override
    public void serviceResolved(DNSSDService resolver, int flags, int ifIndex, 
            String fullName, String hostName, int port, TXTRecord txtRecord) {
        if (mSubscriber.isUnsubscribed()){
            return;
        }
        mBonjourService.port = port;
        mBonjourService.hostname = hostName;
        mBonjourService.dnsRecords.clear();
        mBonjourService.dnsRecords.putAll(parseTXTRecords(txtRecord));
        mSubscriber.onNext(mBonjourService);
        mSubscriber.onCompleted();
        resolver.stop();
    }

    @Override
    public void operationFailed(DNSSDService service, int errorCode) {
        if (mSubscriber.isUnsubscribed()){
            return;
        }
        mSubscriber.onError(new RuntimeException("DNSSD resolve error: " + errorCode));
    }
}
~~~

Example of usage:

~~~java
mSubscription = RxDNSSD.browse(mReqType, mDomain)
        .compose(RxDNSSD.resolve())
        .observeOn(AndroidSchedulers.mainThread())
        .subscribe(bonjourService -> {
            if ((bonjourService.flags & BonjourService.DELETED) == BonjourService.DELETED) {
                return;
            }
            updateUI(bonjourService, true);
        }, throwable -> {
            Log.e("DNSSD", "Error: ", throwable);
            showErrorMessage(throwable);
        });
~~~

**Query Records** 
------------- 
We use `queryRecords` to query for an arbitrary DNS record. This operation is very similar to the previous one.

~~~java
public static Observable.Transformer<BonjourService, BonjourService> queryRecords() {
    return observable -> observable.flatMap(bs -> {
        if ((bs.flags & BonjourService.DELETED) == BonjourService.DELETED) {
            return Observable.just(bs);
        }
        return INSTANCE.createObservable(subscriber -> DNSSD.queryRecord(0, bs.ifIndex, 
            bs.hostname, 1, 1, new QueryListener(subscriber, bs)));
    });
}

private static class QueryListener implements com.apple.dnssd.QueryListener{
    private Subscriber<? super BonjourService> mSubscriber;
    private BonjourService mBonjourService;

    private QueryListener(Subscriber<? super BonjourService> subscriber, 
            BonjourService bonjourService){
        mSubscriber = subscriber;
        mBonjourService = bonjourService;
    }

    @Override
    public void queryAnswered(DNSSDService query, int flags, int ifIndex, 
            String fullName, int rrtype, int rrclass, byte[] rdata, int ttl) {
        if (mSubscriber.isUnsubscribed()){
            return;
        }
        try {
            InetAddress address = InetAddress.getByAddress(rdata);
            mBonjourService.dnsRecords.put(BonjourService.DNS_RECORD_KEY_ADDRESS, 
                address.getHostAddress());
            mSubscriber.onNext(mBonjourService);
            mSubscriber.onCompleted();
        } catch (Exception e) {
            mSubscriber.onError(e);
        } finally {
            query.stop();
        }
    }
    @Override
    public void operationFailed(DNSSDService service, int errorCode) {
        if (mSubscriber.isUnsubscribed()){
            return;
        }
        mSubscriber.onError(new RuntimeException("DNSSD queryRecord error: " + errorCode));
    }
}
~~~

If the operation finishes sucessfully, I put a resolved address to `BonjourService` and emit this object. 

Example of usage:

~~~java
mSubscription = RxDNSSD.browse(mReqType, mDomain)
        .compose(RxDNSSD.resolve())
        .compose(RxDNSSD.queryRecords())
        .observeOn(AndroidSchedulers.mainThread())
        .subscribe(bonjourService -> {
            if ((bonjourService.flags & BonjourService.DELETED) == BonjourService.DELETED) {
                return;
            }
            updateUI(bonjourService, true);
        }, throwable -> {
            Log.e("DNSSD", "Error: ", throwable);
            showErrorMessage(throwable);
        });
~~~

Creating of Rx chains is very usefull because you don't need to care about canceling of all operations and listeners removal. The only thing you have to do inside your `onPause`||`onStop`||`onDestroy` methods is to call `mSubscription.cancel()`. Another big advantage is the ability to handle all exceptions from a chain in one place.

You can find a usage of the described wrapper (I've named it [`RxDNSSD`](https://github.com/andriydruk/BonjourBrowser/blob/master/app/src/main/java/com/druk/bonjour/browser/dnssd/RxDNSSD.java)) in my project [Bonjour Browser](https://github.com/andriydruk/BonjourBrowser). 

That's all. Happy wrapping annoying APIs!