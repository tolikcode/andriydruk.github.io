+++
date = "2015-09-13T22:51:10+03:00"
description = ""
draft = false
image = "/img/material-005.jpg"
title = "Wrap Annoying APIs With RxJava"

+++

[Reactive Extensions (also known as ReactiveX or Rx)](http://reactivex.io/intro.html) is a library, that brings [FRP](https://en.wikipedia.org/wiki/Functional_reactive_programming) programming paradigm to different platforms. And it's a breath of fresh air for android developers in Java 6 desert. A lot of articles about RxJava concept and common usage have already been written. That's why I would like to publish my example of RxJava usage in Android just to show you how you can wrap Java APIs. In this article I will describe a process of Java API wrapping for Apple’s DNSSD library (which in my opinion is quite an annoying API). 

DNSSD library's public API is a Singleton that provides static methods implementing basic operations such as searching or advertising. All operations work asynchronously and their results are returned to you in callbacks from a background thread. It's a usual Java API. What's wrong with it? Let's imagine a basic usage scenario of services search in a network: 

* Browse service
* Resolve service
* Query records
* Update UI

That means that you should start 'browse' operation with a BrowseCallback. In this callback you start 'resolve' with a ResolveCallback, and than inside ResolveCallback you start 'query records' with a QueryRecordsCallback. After that you send a message to the MainThread, where you update your UI. This turns to a 'callback hell'. Real problems arise when a user leaves your app before you've got any results from this chain of callbacks, and you have to unsubscribe all listeners and to cancel all operations. 

Ok, let's wrap this API with RxJava.

<!--more-->

**Intro**
------------- 

If you aren't familiar with RxJava, I highly recommend you to start from the official documentation. In this article I'm not going to explain RxJava operators, only example of usage.

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

Every method returns a DNSSDService object, that is actually an analog for [Subscription](http://reactivex.io/RxJava/javadoc/rx/Subscription.html) object in RxJava.

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

I've presented every operation (static method from DNSSD) as RxJava Observable: 

~~~java
interface class RxDNSSDService {
     Observable<T> getObservable(Context context);
}
~~~

Actually implementation of getObservable method can be common for all operations. Let's transform this interface to an abstract class:

~~~java
abstract static class RxDNSSDService<T> {

    DNSSDService mService;

    protected abstract DNSSDService getService(Subscriber<? super T> subscriber) 
        throws DNSSDException;

    protected Observable<T> getObservable(final Context context){
        return Observable.create(new Observable.OnSubscribe<T>() {

            @Override
            public void call(Subscriber<? super T> subscriber) {
                if (!subscriber.isUnsubscribed()) {
                    context.getSystemService(Context.NSD_SERVICE);
                    try {
                        mService = getService(subscriber);
                    } catch (DNSSDException e) {
                        e.printStackTrace();
                        subscriber.onError(e);
                    }
                }
            }
        }).doOnUnsubscribe(() -> {
            if (mService != null) {
                mService.stop();
                mService = null;
            }
        });
    }
}
~~~

Now method getObservable returns a new Observable. Inside OnSubsribe callback I will check if a subscriber is actually subscribed at that moment and call the original DNSSD API.  I've also added [doOnUnsubscribe](http://reactivex.io/documentation/operators/do.html) operator where I stop Apple's library DNSSDService.

Before implementing classes extending RxDNSSDService I've created a BonjourService POJO class. The main idea is to create a chain of observables that will operate with this class:

~~~java
public class BonjourService {

    public final int flags;
    public final int ifIndex;
    public final String serviceName;
    public final String regType;
    public final String domain;

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
Ok, the first operation I've implemented is 'browse':

~~~java
public static Observable<BonjourService> browse(final String regType, final String domain) {
    return new RxDNSSDService<BonjourService> (){
        @Override
        protected DNSSDService getService(Subscriber<? super BonjourService> subscriber) 
                throws DNSSDException {
            return DNSSD.browse(0, DNSSD.ALL_INTERFACES, regType, domain,
                    new RxDNSSD.BrowseListener(subscriber));
        }
    }.getObservable(INSTANCE.mContext);
}

private static class BrowseListener implements com.apple.dnssd.BrowseListener{
    private Subscriber<? super BonjourService> mSubscriber;

    private BrowseListener(Subscriber<? super BonjourService> subscriber){
        mSubscriber = subscriber;
    }

    @Override
    public void serviceFound(DNSSDService browser, int flags, int ifIndex, 
            String serviceName, String regType, String domain) {
        if (mSubscriber.isUnsubscribed()){
            return;
        }
        BonjourService service = new BonjourService(flags, ifIndex, serviceName, 
            regType, domain);
        mSubscriber.onNext(service);
    }

    @Override
    public void serviceLost(DNSSDService browser, int flags, int ifIndex, 
            String serviceName, String regType, String domain) {
        //Not implemented yet
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

'browse' is an endless operation. That's why it never calls onComplete method on a Subscriber. Every time when DNSSD library calls a serviceFound callback I emit a new BonjourService object. At the moment I haven't yet implemented serviceLost callback. In RxJava Observable represents a series of events that happen at some time sequence. That's why I have to create some additional class (for example BrowseAnswer) that contains BonjourService and a boolean flag. But it will destroy our idea about chain of observable with one type - BonjourService. Actually BonjourService already has a flag and I will use it to mark BonjourService as deleted. According to DNSSD documentation Apple uses next masks for flags:

~~~java
/**	Flag indicates to a {@link BrowseListener} that another result is
	queued.  Applications should not update their UI to display browse
	results if the MORE_COMING flag is set; they will be called at least once
	more with the flag clear.
*/
public static final int		MORE_COMING = ( 1 << 0 );

/**	If flag is set in a {@link DomainListener} callback, indicates that the 
	result is the default domain. 
*/
public static final int		DEFAULT = ( 1 << 2 );

/**	If flag is set, a name conflict will trigger an exception when registering 
	non-shared records.<P> A name must be explicitly specified when 
	registering a service if this bit is set (i.e. the default name may 
	not not be used).
 */
public static final int		NO_AUTO_RENAME = ( 1 << 3 );

/**	If flag is set, allow multiple records with this name on 
	the network (e.g. PTR records)
	when registering individual records on a {@link DNSSDRegistration}.
*/
public static final int		SHARED = ( 1 << 4 );

/**	If flag is set, records with this name must be unique on 
	the network (e.g. SRV records). 
*/
public static final int		UNIQUE = ( 1 << 5 );

/**	Set flag when calling enumerateDomains() to restrict results to domains 
	recommended for browsing. 
*/
public static final int		BROWSE_DOMAINS = ( 1 << 6 );

/**	Set flag when calling enumerateDomains() to restrict results to domains 
	recommended for registration. 
*/
public static final int		REGISTRATION_DOMAINS = ( 1 << 7 );
~~~

Despite the fact that not all of these flags are used with 'browse' operation, we will create a new mask (1 << 8) for our DELETED flag. If you aren't familiar with bit shifts, this operator makes  a shift for a binary number to N position. In the example above this number is 256. After adding a new mask 'serviceLost' method looks like:

~~~java
@Override
public void serviceLost(DNSSDService browser, int flags, int ifIndex, String serviceName, 
        String regType, String domain) {
    if (mSubscriber.isUnsubscribed()){
        return;
    }
    BonjourService service = new BonjourService(flags | BonjourService.DELETED, ifIndex, 
        serviceName, regType, domain);
    mSubscriber.onNext(service);
}
~~~

Now look at the usage of my browse method:

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
    });
~~~

I don't use subscribeOn, because DNSSD will create its own thread and it's not necessary to call all DNSSD API methods from a background thread. However, I always use observeOn, because all callbacks from DNSSD API will be called from a background thread.

**Resolve** 
-------------

Next operation is DNSSD.resolve(). I use this operation to resolve a service name to a target host, port number, and a txt record. 
Unlike browse() operation this one will return Trasnformation, that transforms an Observable of BonjourService to an Observable of resolved BonourService. 

First of all I've applied a [flatMap](http://reactivex.io/documentation/operators/flatmap.html) to income Observalbe. Then in map's function I check if BonjourService contains DELETED flag. After that I emit this Observable (maybe the information about this BonjourService will be usefull for a Subsriber). 
In another case I emit Observable from RxDNSSDService that will query record from DNSSD API.

~~~java
public static Observable.Transformer<BonjourService, BonjourService> resolve() {
    return observable -> observable.flatMap(bs -> {
        if ((bs.flags & BonjourService.DELETED) == BonjourService.DELETED) {
            return Observable.just(bs);
        }
        return new RxDNSSDService<BonjourService>() {
            @Override
            protected DNSSDService getService(Subscriber<? super BonjourService> subscriber) 
                    throws DNSSDException {
                return DNSSD.resolve(bs.flags, bs.ifIndex, bs.serviceName, bs.regType, 
                    bs.domain, new ResolveListener(subscriber, bs));
            }
        }.getObservable(INSTANCE.mContext);
    });
}
~~~

If the operation finishes successfully, I will put resolved address to BonjourService's map and send it to a Subscriber. Unlike browse() operation this one is not endless, that's why after emitting Bonjour Service I call onComplete. In the case of error, I forward this error to a Subscriber.

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
        mBonjourService.timestamp = System.currentTimeMillis();
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
mResolveSubscription = Observable.just(mService)
        .compose(RxDNSSD.resolve())
        .observeOn(AndroidSchedulers.mainThread())
        .subscribe(bonjourService -> {
            if ((bonjourService.flags & BonjourService.DELETED) == BonjourService.DELETED) {
                return;
            }
            updateUI(bonjourService, true);
        }, throwable -> {
            Log.e("DNSSD", "Error: ", throwable);
        });
~~~

**Query Records** 
------------- 
We use DNSSD.queryRecords() to query for an arbitrary DNS record. This operation is very similar to the previous one. 

~~~java
public static Observable.Transformer<BonjourService, BonjourService> queryRecords() {
    return observable -> observable.flatMap(bs -> {
        if ((bs.flags & BonjourService.DELETED) == BonjourService.DELETED) {
            return Observable.just(bs);
        }
        return new RxDNSSDService<BonjourService>() {
            @Override
            protected DNSSDService getService(Subscriber<? super BonjourService> subscriber) 
                    throws DNSSDException {
                return DNSSD.queryRecord(0, bs.ifIndex, bs.hostname, 1 , 1,
                        new QueryListener(subscriber, bs));
            }
        }.getObservable(INSTANCE.mContext);
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

If the operation finishes sucessfully, I put a resolved address to BonjourService map and emit this object. 

Example of usage:

~~~java
mResolveSubscription = Observable.just(mService)
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
        });
~~~

Creating of Rx chains is very usefull because you don't need to care about canceling all operations and removing listeners. The only thing you have to do inside your onPause||onStop||onDestroy methods is to call mSubscription.cancel().
You can find a usage af the described wrapper (I called it RxDNSSD) in my project [Bonjour Browser](https://github.com/andriydruk/BonjourBrowser). 

That's all. Happy wrapping annoying APIs!