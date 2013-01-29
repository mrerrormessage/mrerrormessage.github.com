---
layout: post
title: "Making Core Data Thread Safe"
description: "A look at using Core Data in a threaded environment"
comments: true
categories: [objective-c, coredata]
---

For iOS and OSX, Apple provides the amazing Core Data framework to manage object storage, loading, and relationships. 
Unfortunately, Core Data is not thread safe. 
You can get yourself into big trouble if you run Core Data on multiple threads and are not _extremely_ careful. 
In iOS 5, Apple released a new feature - [nested managed object contexts](http://developer.apple.com/library/ios/#releasenotes/DataManagement/RN-CoreData/_index.html) to deal with the complexity of thread safety and concurrency. 
Unfortunately (especially because numerous blog posts have been written about how to use these features to make Core Data work across threads) it appears that these features do not fix the problem they were supposed to fix (making background Core Data tasks easier), even worse it seems that they can cause crashes. 
See [wbyoung's blog post](http://wbyoung.tumblr.com/post/27851725562/core-data-growing-pains) for more details. 

Fortunately, there are ways to use Core Data in a thread-safe way without nesting managed object contexts. 
Apple has a published [Core Data Concurrency Guide](http://developer.apple.com/library/ios/#documentation/cocoa/conceptual/CoreData/Articles/cdConcurrency.html) which lists cautions and fixes for working with Core Data in a threaded environment. 
Unfortunately, it is quite vague, offers no sample code, and makes recommendations with little context. 
Last week my team and I tackled crashes involving Core Data concurrency and I thought it might be helpful to share how we managed to use Core Data safely in a background thread.

Our application's architecture has a singleton class from which all other classes can obtain an `NSManagedObjectContext` when needed. This approach is part of the reason it was possible for us to fix this problem in our application. This class looks something like:

    @interface CoreDataManager
    @property(nonatomic, retain)NSManagedObjectContext *managedObjectContext;
    @property(nonatomic, retain)NSPersistentStoreCoordinator *persistentStoreCoordinator;
    
    + (CoreDataManager *)sharedCoreDataManager;
    @end

You will notice that the `persistentStoreCoordinator` is nonatomic. 
This is because we do not work directly with `persistentStoreCoordinator` and threading issues between multiple managed object contexts and a single persistent store coordinator are taken care of by the framework.
Now, a naive implementation of making this thread-safe would be to make `managedObjectContext` atomic. 
This actually doesn't produce desired result because __any__ change made to __any__ `NSManagedObject` is not thread safe. 
Due to Core Data faulting (lazy loading), even __reads__ are not thread safe. 

In our project, we were already using Core Data on a background thread and starting to see some very frequent crashes from sharing a managed object context across thread boundaries. 
Apple's recommendation is to have multiple managed object contexts share a single persistent store coordinator, so we did that. It looked something like this:

    @interface CoreDataManager
    
    - (NSManagedObjectContext *)managedObjectContextForDetachedThread;
    @end
    
    @implementation CoreDataManager
    - (NSManagedObjectContext *)managedObjectContextForDetachedThread
    {
        NSManagedObjectContext *context = [[[NSManagedObjectContext alloc] init] autorelease];
        [context setPersistentStoreCoordinator:self.persistentStoreCoordinator];
        return context;
    }
    @end

This is good enough to stop the crashing. 
However, it only gets us halfway to where we want to be.
Any time we update in a background thread, the results aren't added to the main-thread managed object context. 
In order to receive those, we have to register for change notifications. 
Apple recommends that [`NSManagedObjectContext`](https://developer.apple.com/library/ios/#documentation/Cocoa/Reference/CoreDataFramework/Classes/NSManagedObjectContext_Class/NSManagedObjectContext.html) register for change notification on any managed object contexts it wants to observe. 
Apple warns that several system frameworks emit these too, so they can be "difficult to handle." 
We have the added challenge that we want to pass along notifications to our UI components, 
but we don't want to register them for notifications to [`NSManagedObjectContextObjectsDidChangeNotification`](https://developer.apple.com/library/ios/documentation/Cocoa/Reference/CoreDataFramework/Classes/NSManagedObjectContext_Class/NSManagedObjectContext.html#//apple_ref/c/data/NSManagedObjectContextObjectsDidChangeNotification)
because they might receive the notification before the changes have been merged in to their managed object context. 

With all this in mind, we can make the following changes:

    extern NSString *const CDMManagedObjectContextDidChange = @"com.example.Example:CDMManagedObjectContextDidChange";

    @implementation CoreDataManager
    - (NSManagedObjectContext *)managedObjectContextForDetachedThread
    {
        NSManagedObjectContext *context = [[[NSManagedObjectContext alloc] init] autorelease];
        [context setPersistentStoreCoordinator:self.persistentStoreCoordinator];
        [[NSNotificationCenter defaultCenter] addObserver:self
                                              selector:@selector(detachedManagedObjectContextDidChange:)
                                              name:NSManagedObjectContextDidSaveNotification                                     
                                              object:context];
        return context;
    }
    
    - (void) detachedManagedObjectContextDidChange:(NSNotification*)notification
    {
        [self.managedObjectContext mergeChangesFromContextDidSaveNotification:notification];
        NSNotificationCenter *center = [[NSNotificationCenter] defaultCenter];
        NSNotification *notification = [NSNotification notificationWithName:CDMManagedObjectContextDidChange object:self.managedObjectContext];
        [center performSelectorOnMainThread:@selector(postNotification:) withObject:notification waitUntilDone:NO];
    }
    
    - (void) dealloc
    {
        [[NSNotificationCenter defaultCenter] removeObserver:self];
    }
    @end

The important things here to notice are that we aren't depending on the default Core-Data-generated notification to update objects on the main thread, and when we pass along our notification to the objects that the main thread managed object context has changed we are doing it on the main thread, but not waiting. 
Both of these points are important.
There are two pitfalls here.
The first is that we send the notification on a background thread.
What this will do is run code registered to the `CDMManagedObjectContextDidChange` notification in a background thread.
This is a _bad_ idea because objects listening for this notification are, by definition, in the main thread and will be accessing their main-thread managed object context from a background thread (which is what we want to avoid in the first place). 
The second thing that is important is that we __do not__ wait for this notification to finish. 
If we wait for it to finish we will lose any speedup from running Core Data in the background because our background thread block waiting for the notification to finish before continuing.
If you are super-paranoid about safety (or just don't care how long background tasks take to run) you could wait until done, but I fail to see any benefit.

So the only thing left is to give some examples about how we integrate this pattern with the rest of the application. Our foreground classes might look a bit like this:

    @implementation SomeForegroundCoreDataClient
    - (id) init
    {
      ...
        self.managedObjectContext = [[CoreDataManager sharedCoreDataManager] managedObjectContext];
        [[NSNotificationCenter defaultCenter] addObserver:self
                                              selector:@selector(dataChanged:)
                                              name:CDMManagedObjectContextDidChange
                                              object:nil];
        self.someObjectForDisplay = someManagedObject;
      ...
    }
    
    - (void)dataChanged:(NSNotification*)notification
    {
        self.someObjectForDisplay = [self.managedObjectContext existingObjectWithID:self.someObjectForDisplay.objectID
                                                                              error:nil];
    }
    @end

What we have done here is just reloaded the object using `existingObjectWithID:error:`.
In my testing, the other two "get this object by its objectID" methods, namely `objectWithID:` and `objectRegisteredForID:` seem to only query the cache and not the persistent store to reload the object.

Readers familiar with Core Data will also likely consider using `refreshObject:mergeChanges:`. 
This may work, but Apple's documentation indicates it will use the cache (exactly the thing we don't want) rather than the persistent store;
even worse it can apply cached changes over changes made in the persistent store. 
You may have some luck with this by playing around with the managed object context's `stalenessInterval` property.
By default it is set to infinite staleness, meaning that changes to the cache are never overwritten by changes in the persistent store.

To summarize, keep the following in mind when working with Core Data off of the main thread:

1. _Centralize_ Core Data management to save headache.
2. _Do not_ share an `NSManagedObject` context between the main thread and a background thread (just don't do it). 
3. _Do_ share an `NSPersistentStoreCoordinator` between all managed object contexts in all threads.
4. _Be careful_ about how you handle Core Data change notifications.

That's all for now!
