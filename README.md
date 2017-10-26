# Realm Mobile Database

An introduction to Realm Mobile Database

<!-- TOC -->

- [History](#history)
- [What is Realm](#what-is-realm)
- [Why](#why)
- [What people say](#what-people-say)
- [Simple](#simple)
    - [Swift](#swift)
    - [Objective-C](#objective-c)
    - [Compared with Core Data Stack](#compared-with-core-data-stack)
    - [Compared with SQLite in Swift](#compared-with-sqlite-in-swift)
- [Benchmark](#benchmark)
    - [iOS](#ios)
    - [Android](#android)
    - [Xamarin](#xamarin)
    - [React Native](#react-native)
- [Why not Realm](#why-not-realm)
- [Realm Mobile Platform (In progress)](#realm-mobile-platform-in-progress)
- [More](#more)

<!-- /TOC -->

## History

Realm launched in July 2014 – Since then also released Realm Java, Realm JavaScript, Realm Xamarin, as well as the Realm Mobile Platform.

## What is Realm

> Realm is not an ORM, and is not built on top of SQLite. Realm has own C++ core. 

> Realm is the first database built from the ground-up to run directly inside phones, tablets and wearables

> Cross platform

```swift
// Using Realm in Swift

var mydog = Dog(); mydog.name = "Rex"; mydog.age = 9

let realm = RLMRealm.defaultRealm()

realm.beginWriteTransaction()
realm.addObject(mydog)
realm.commitWriteTransaction()

var results = Dog.objectsWhere("name contains 'Rex'")
```

You can use it directly inside your iOS (and other mobile platforms) to store & query data locally on the device, allowing you to build apps faster, build apps that are faster.

## Why

There’s been an explosion in the number of database for the server-side, with several projects starting to rival old stalwarts MySQL and PostgreSQL in popularity. The issue of course is that none of these databases can actually run on phones, tablets or wearables, because they were never designed for that in the first place!

![Database Technology](https://images.contentful.com/3c9g5h7ou6jg/79erDXfQZ2qgCiKggQUcwm/2e4925499ac2c34155423681b630ec17/introducing-realm-timeline.png)

If you’re a mobile developer, your only viable option today is the same as it was in 2000: SQLite. It’s used for persistence by virtually every mobile app today — either directly or through one of the many libraries that provides a convenience wrapper around it such as Couchbase Lite, Core Data, ORMLite, etc.

## What people say

David Helgason, CEO, Unity3D
> “Finally an elegant, low-footprint and multi-platform database for apps… And so beautifully designed too.”

## Simple

### Swift

```swift
/* Dog.swift */
class Dog: RLMObject {
    var name = ""
    var age = 0
}

/* Elsewhere */
var mydog = Dog()
mydog.name = "Rex"; mydog.age = 9
println("Name of dog: \(mydog.name)")

let realm = RLMRealm.defaultRealm() // Access default realm (database) on disk

// Transactions for full ACID guarantees
realm.beginWriteTransaction()
realm.addObject(mydog)
realm.commitWriteTransaction()
// You can safely transact across threads as well

// Query
var results = Dog.objectsWhere("name contains 'x'")

// Link objects in a Graph
var person = Person()
person.name = "Tim"
person.dogs.addObject(mydog)
```

### Objective-C

```obj-c
/* Dog.h */
@interface Dog : RLMObject
@property NSString *name;
@property NSInteger age;
@end
RLM_ARRAY_TYPE(Dog)

/* Elsewhere */
Dog *mydog = [[Dog alloc] init];
mydog.name = @"Rex"; mydog.age = 9;
NSLog(@"Name of dog: %@", mydog.name);

RLMRealm *realm = [RLMRealm defaultRealm]; // Access default realm (database) on disk

// Transactions for full ACID guarantees
[realm beginWriteTransaction];
[realm addObject:mydog];
[realm commitWriteTransaction];
// You can safety transact across threads as well

// Query
RLMArray *results = [Dog objectsWhere:@"name contains 'x'"];

// Link objects in a Graph
Person *person = [[Person alloc] init];
person.name = @"Tim";
[person.dogs addObject:mydog];
```

### Compared with Core Data Stack

```swift
//
//  PersistenceStack.swift
import CoreData

public class PersistenceStack: NSObject {
    let mainContext: NSManagedObjectContext = {
        let context = NSManagedObjectContext(concurrencyType: .MainQueueConcurrencyType)
            context.undoManager = nil
            context.mergePolicy = NSMergeByPropertyObjectTrumpMergePolicy
            return context
    }()
    var isSetup: Bool { return mainContext.persistentStoreCoordinator != nil }
    override init() {
        super.init()
        setupStack()
    }
    class func sharedStack() -> PersistenceStack {
        struct Singleton {
            static let _sharedStack = PersistenceStack()
        }
        return Singleton._sharedStack
    }
    public func deleteAndResetStack() {
        var error: NSError?
        let coordinator = mainContext.persistentStoreCoordinator
        if let store = coordinator.persistentStoreForURL(storeURL) {
            mainContext.reset()
            let removedStore = coordinator.removePersistentStore(store, error: &error)
            if !removedStore {
                println("Unable to remove store: \(error)")
                return
            }
            let fm = NSFileManager.defaultManager()
            let deleted = fm.removeItemAtURL(storeURL, error: &error)
            if !deleted {
                println("Unable to remove Core Data DB at \(storeURL): \(error)")
            }
            addStoreToCoordinator(coordinator)
        }
    }
    private func setupStack() {
        let model = NSManagedObjectModel(contentsOfURL: modelURL)
        let coordinator = NSPersistentStoreCoordinator(managedObjectModel: model)
        addStoreToCoordinator(coordinator)
        mainContext.persistentStoreCoordinator = coordinator
    }
    private func addStoreToCoordinator(coordinator: NSPersistentStoreCoordinator) {
        var error: NSError?
        let store = coordinator.addPersistentStoreWithType(NSSQLiteStoreType, configuration: nil, URL: storeURL, options: nil, error: &error)
        if store == nil {
            println("Could not open store at \(storeURL): \(error)")
        }
    }
    public func printStoreInfo() {
        let model = mainContext.persistentStoreCoordinator.managedObjectModel
        let entities = model.entities as [NSEntityDescription]
        for entity in entities {
            println(entity)
        }
    }
    public func save() {
        let moc = mainContext
        moc.performBlock {
            var error: NSError?
            if !moc.save(&error) {
                println("Error saving context: \(error)")
            }
        }
    }
}

extension PersistenceStack { // MARK: URLs
    var modelURL: NSURL! {
        return NSBundle.mainBundle().URLForResource("Posessions", withExtension: "momd")
    }
    var storeURL: NSURL! {
        let urls = NSFileManager.defaultManager().URLsForDirectory(NSSearchPathDirectory.DocumentDirectory, inDomains: NSSearchPathDomainMask.UserDomainMask)
        let documentURL = urls.last as NSURL
        let storeURL = documentURL.URLByAppendingPathComponent("Posessions.sqlite")
        return storeURL
    }
}
```

> There is a lot of boilerplate code required to write a Core Data application. This is annoying. In pretty much everything I’ve written since Core Data came to iOS.

### Compared with SQLite in Swift

```swift 
func openDatabase() -> OpaquePointer? {
    var db: OpaquePointer? = nil
    if sqlite3_open(part1DbPath, &db) == SQLITE_OK {
        print("Successfully opened connection to database at \(part1DbPath)")
        return db
    } else {
        print("Unable to open database. Verify that you created the directory described " +
            "in the Getting Started section.")
        PlaygroundPage.current.finishExecution()
    }
}

func createTable() {
        // 1
        var createTableStatement: OpaquePointer? = nil
        // 2
        if sqlite3_prepare_v2(db, createTableString, -1, &createTableStatement, nil) == SQLITE_OK {
            // 3
            if sqlite3_step(createTableStatement) == SQLITE_DONE {
                print("Contact table created.")
            } else {
                print("Contact table could not be created.")
            }
        } else {
            print("CREATE TABLE statement could not be prepared.")
        }
        // 4
        sqlite3_finalize(createTableStatement)
    }

func insert() {
    var insertStatement: OpaquePointer? = nil
    
    // 1
    if sqlite3_prepare_v2(db, insertStatementString, -1, &insertStatement, nil) == SQLITE_OK {
        let id: Int32 = 1
        let name: NSString = "Ray"
        
        // 2
        sqlite3_bind_int(insertStatement, 1, id)
        // 3
        sqlite3_bind_text(insertStatement, 2, name.utf8String, -1, nil)
        
        // 4
        if sqlite3_step(insertStatement) == SQLITE_DONE {
            print("Successfully inserted row.")
        } else {
            print("Could not insert row.")
        }
    } else {
        print("INSERT statement could not be prepared.")
    }
    // 5
    sqlite3_finalize(insertStatement)
}
    
```

> SQLite written in ANSI-C with C style and you may not prefer

## Benchmark

### iOS

![](https://images.contentful.com/3c9g5h7ou6jg/6PQMUM8p20agQQY6sUo2W0/03beef834bebb49b0747a64b321c0d1e/benchmarks.002b.png)

![](https://images.contentful.com/3c9g5h7ou6jg/1stWlBzjbCGouKq8mwqOKy/06f1ebda2e7e171e16b846c832ba44aa/benchmarks.003b.png)

![](https://images.contentful.com/3c9g5h7ou6jg/6kiTSC5Wj6cu28kOWGsa6E/967d8dafe753c1753552ea17210dc2e9/benchmarks.001b.png)

> UPDATE: we’ve updated the charts above to reflect changes in measurements since this article was originally published. In particular, the original insert benchmark was not reusing compiled statements for SQLite, but now does. The numbers were obtained from tests run on an iPad Air using the latest available version of each library as of Sept 15, 2014.

### Android

![](https://images.contentful.com/3c9g5h7ou6jg/66WeyNzBDiAe8sIAO4C2YY/2f3ae71dda19a6ae585ac341a3959805/benchmarks-android.002.png)

![](https://images.contentful.com/3c9g5h7ou6jg/2PiuZpUccgIauOmKwCikyO/45b58133ec9552c8d06abb8f7eef7c63/benchmarks-android.003.png)

![](https://images.contentful.com/3c9g5h7ou6jg/48I7dCZ63eSKWOIyi0wGu6/ae000b1628a8811246ab7e170adcea82/benchmarks-android.001.png)

>Tests run on an Galaxy S3, using the latest available version of each library as of Sept 28, 2014.

UPDATE 11/06/2015: we have significantly updated Realm and the way we do benchmarks on Android and encourage people to refer to [these updated results instead](http://www.slideshare.net/ChristianMelchior/realm-building-a-mobile-database#25)

### Xamarin

![](https://images.contentful.com/3c9g5h7ou6jg/2QN86s8K4ogiCkooIIwMoq/9b16f68052c7a3db135bc6a0b9d7da93/xamarin-benchmarks-002.png)

![](https://images.contentful.com/3c9g5h7ou6jg/7rU2d0Jw1qAKMsGcq26m4S/29517ef0914ddc34dfee03fba6671df8/xamarin-benchmarks-003.png)

![](https://images.contentful.com/3c9g5h7ou6jg/6nXf41Wq0Eo6uaCoWiMMwm/2d035d545debd11a8ef54fa53f9509d2/xamarin-benchmarks-001.png)

### React Native

![](https://images.contentful.com/3c9g5h7ou6jg/6zltiFnHi06EwAc4WyewYI/4f09aa7b64262ead612119164fb9ccbe/react-native-benchmark-002.png)

![](https://images.contentful.com/3c9g5h7ou6jg/25nEt9TYqc0oAAeYE0qUa/a2e167e41a6467e330958f1e02e13d7c/react-native-benchmark-005.png)

![](https://images.contentful.com/3c9g5h7ou6jg/5fBPqUePoAsA2emeeOmmYe/0c0873575872d62e0cad257b74ecc28a/react-native-benchmark-003.png)

![](https://images.contentful.com/3c9g5h7ou6jg/6rPuRQxqDeSKyY20EMQWgW/3cdb9212383db13434742dbf742342f5/react-native-benchmark-006.png)

![](https://images.contentful.com/3c9g5h7ou6jg/6HYptDR6RqC8MEckUS8m2E/8b5daea31a839eb9265182273f31a519/react-native-benchmark-001.png)

![](https://images.contentful.com/3c9g5h7ou6jg/UXFFKznhgyaSsO4eWWYsS/f7562f5e94638a4bcc26d611b7ee9154/react-native-benchmark-004.png)

## Why not Realm

Because Realm aims to strike a balance between flexibility and performance, In order to accomplish this goal so here’s a list of our most commonly hit limitations.

* iOS (Realm Swift 3.0.0 - latest)
  * General
    * Class names are limited to a maximum of 57 UTF8 characters.
    * Property names are limited to a maximum of 63 UTF8 characters.
    * Data and String properties cannot hold data exceeding 16MB in size.
    * Any single Realm file cannot be larger than the amount of memory your application would be allowed to map in iOS.
    * String sorting and case insensitive queries are only supported for character sets in ‘Latin Basic’, ‘Latin Supplement’, ‘Latin Extended A’, ‘Latin Extended B’ (UTF-8 range 0-591).
  * Theads
    * Although Realm files can be accessed by multiple threads concurrently, you cannot directly pass Realms, Realm objects, queries, and results between threads
  * Models
    * Setters and getters: Since Realm overrides setters and getters to back properties directly by the underlying database, you cannot override them on your objects.
    * Auto-incrementing properties: Realm has no mechanism for thread-safe/process-safe auto-incrementing properties commonly used in other databases when generating primary keys
    * Properties from Objective-C: If you need to access your Realm Swift models from Objective‑C, List and RealmOptional properties will cause the auto-generated Objective‑C header (-Swift.h) to fail to compile because of the use of generics
    * Custom initializers for Object subclasses: When creating your model Object subclasses, you may sometimes want to add your own custom initialization methods for added convenience.
* Android (Realm Java 4.0.0 - latest)
  * Models: Realm models have no support for final and volatile fields. This is mainly to avoid discrepancies between how an object would behave as managed by Realm or unmanaged.
  * General (similar to iOS)
  * Sorting and querying on strings: Sorting and case-insensitive string matches in queries are only supported for character sets in Latin Basic, Latin Supplement, Latin Extended A, and Latin Extended B (UTF-8 range 0–591). Also, setting the case insensitive flag in queries when using equalTo, notEqualTo, contains, endsWith, beginsWith, or like will only work on characters from the English locale. Realm uses non-standard sorting for upper and lowercase letters, sorting them together rather than sorting uppercase first. That means that '- !"#0&()*,./:;?_+<=>123aAbBcC...xXyYzZ is the actual sorting order in Realm. Read more about these limitations [here](https://github.com/realm/realm-java/issues/581).
  * Threads (similar to iOS)
  * RealmObject’s hashCode: A `RealmObject` is a live object, and it might be updated by changes from other threads. Although two Realm objects returning `true` for `RealmObject.equals` must have the same value for `RealmObject.hashCode`, the value is not stable, and should neither be used as a key in `HashMap` nor saved in a `HashSet`.

## More

* <https://realm.io/docs>
