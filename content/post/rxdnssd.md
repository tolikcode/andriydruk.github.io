+++
date = "2015-09-13T22:51:10+03:00"
description = ""
draft = true
image = "/img/about-bg.jpg"
tags = ["androiddev"]
title = "RxJava with DNSSD"

+++

Today I wanna show how you can use RxJava in your application on example of using Apple DNSSD Java API. This API based on one Singelton DNSSD and numbers of static method with callback such as browse, queryRecord or resolve. Every method returns DNSSDService object with only one method 'stop'. Basic sceneary of searching services in network are browse service -> query its records -> resolve address. And it turns to 'callback hell'. And real problem starts when user leave your app before you get result from this chain and you should cancel all this operation immedeatly.

Firstly, we will create domain class BonjourService:

~~~java
public class BonjourService {

    public final int flags;
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

Every operation can be presented like a RxJava Observable: 

~~~java
interface class RxDNSSDService {
     Observable<BonjourService> getObservable(Context context);
}
~~~

Actually implementation of getObservable method can be a common for all operation. Let's transform inteface to abstract class:

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

Now method getObservable returns new Observalbe with OnSubscribe and OnUnsubscribe. On OnSubsribe callback firstly we check if subsriber is subsribrated at this moment. Than I start NSD_SERVICE to avoid Exception I describred in my previous post. And next I create DNSSDService. On OnSubscribe I check if service was started, and that stop it.

Ok, first operation I created is browse:

~~~java
public static Observable<BonjourService> browse(final String regType, final String domain) {
return new RxDNSSDService<BonjourService> ()
{
    @Override
    protected DNSSDService getService(Subscriber<? super BonjourService> subscriber) 
        throws DNSSDException {

        return DNSSD.browse(0, DNSSD.ALL_INTERFACES, regType, domain, new BrowseListener() {
                 
            @Override
            public void serviceFound(DNSSDService browser, int flags, int ifIndex, 
        	    String serviceName, String regType, String domain) 
            {
              	if (!subscriber.isUnsubscribed()) {
                    BonjourService service = new BonjourService(flags, ifIndex, 
                    	serviceName, regType, domain);
                    subscriber.onNext(service);
                }
            }

            @Override
            public void serviceLost(DNSSDService browser, int flags, int ifIndex, 
            	String serviceName, String regType, String domain) {
                 	//no implementation(
            }

            @Override
            public void operationFailed(DNSSDService service, int errorCode) {
                if (!subscriber.isUnsubscribed()) {
                    subscriber.onError(new RuntimeException("DNSSD browse 
                    	error: " + errorCode));
                }
            }
        });
    }
}.getObservable(INSTANCE.mContext);
}
~~~

Browse is endless operation that's why it never calls onComplete method of Subscriber. Every time DNSSD library calls serviceFound callback I push new BonjourService object to Subscriber by onNext. What about 'serviceLost'? Concept of observable is a line of events that happend on some time sequance. That's why we should create additional class for BrowseAnswer that contains BonjourService and some boolean flag. But it will destoy our idea about chain of observable with one type BonjourBrowser. Can we avoid this problem? Yes, BonjourService already has flag option and we will use it for mark BonjourService as deleted. From DNSSD documentation you can find that Apple use next mask for flag:

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

Despite of fact that not all of this flag used with browse opearion, we will create new mask (1 << 8) for our DELETED flag. If you not familiar with bit's shifts, this operator make shift for binary number to N position. In case above it's number 256. After adding new mask 'serviceLost' method:

~~~java
@Override
public void serviceLost(DNSSDService browser, int flags, int ifIndex, String serviceName, 
	String regType, String domain) {
    if (!subscriber.isUnsubscribed()) {
        BonjourService service = new BonjourService(flags | BonjourService.DELETED, 
            ifIndex, serviceName, regType, domain); 	
        subscriber.onNext(service);	
   }
}
~~~

Now look to usages of rxBrowse method:
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

We don't use subscribeOn because DNSSD create own thread and it's not a neccesary to call all DNSSD API method from working thread. But observeOn we use always, because all callbacks from DNSSD API will returned in not main thread.

Move to next method:  queryRecords. Unlike browse method this one will get Observable as a argument, which is mean that you should connect this method to Rx chain or create Observable.just().

~~~java
public static Observable<BonjourService> queryRecords(Observable<BonjourService> observable) {
return observable.flatMap(bs -> {
    if ((bs.flags & BonjourService.DELETED) == BonjourService.DELETED) {
        return Observable.just(bs);
    }
    return new RxDNSSDService() {

        @Override
        protected DNSSDService getService(Subscriber<? super BonjourService> subscriber) 
        	throws DNSSDException {
        return DNSSD.queryRecord(0, bs.ifIndex, bs.hostname, 1, 1, new QueryListener() {

            @Override
            public void queryAnswered(DNSSDService query, int flags, int ifIndex, 
            	String fullName, int rrtype, int rrclass, byte[] rdata, int ttl) {
                try {
                    InetAddress address = InetAddress.getByAddress(rdata);
                    bs.dnsRecords.put(BonjourService.DNS_RECORD_KEY_ADDRESS, 
                    	address.getHostAddress() + ":" + bs.port);
                    subscriber.onNext(bs);
                    subscriber.onCompleted();
                } catch (Exception e) {
                    subscriber.onError(e);
                } finally {
                    query.stop();
                }
            }

            @Override
            public void operationFailed(DNSSDService service, int errorCode) {
                subscriber.onError(new RuntimeException("DNSSD queryRecord error: " 
                	+ errorCode));
            }
        });
        }
    }.getObservable(INSTANCE.mContext);
});
}
~~~

Firstly we transform Observable from arguments to flatMap. Than in map function we will check if BonjourService contains DELETED flag, we will ignore it and return Observable.just(bs) (maybe information about this BonjourService will be usefull for Subsriber). In other case we return new RxDNSSDService that will queryRecord from DNSSD API with anonymus callback. If operation is sucessfull, we will put resolved address to BonjourService map and send it to Subscriber. Unlike browse opearion this operation has end, that's why after sending BonjourService we will call onComplete. In case of cathing error, we will send  onError message to Subscriber.

Example of usage:
~~~java
mSubscription = RxDNSSD.queryRecords(RxDNSSD.resolve(RxDNSSD.browse(mReqType, mDomain)))
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe(bonjourService -> {
        if ((bonjourService.flags & BonjourService.DELETED) == BonjourService.DELETED) {
            return;
        }
        updateUI(bonjourService);
    }, throwable -> {
    	Log.e("DNSSD", "Error: ", throwable);
    });
~~~

Sample of using RxDNSSD you can find in my project Bonjour Browser.
