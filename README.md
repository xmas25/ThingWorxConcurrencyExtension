# ThingWorx Concurrency Extension

The aim of this extension it's to provide the Swiss Tool for Concurrency in ThingWorx.

## Contents

- [How It Works](#how-it-works)
- [Compatibility](#compatibility)
- [Installation](#installation)
- [Usage](#usage)
    - [Mutex](#mutex)
    - [Counter](#counter)
    - [Concurrency Script Functions](#concurrency-script-functions)
- [Build](#build)
- [Acknowledgments](#acknowledgments)
- [Author](#author)


## How It Works

It publishes standard Java concurrency features in order to be used easily on ThingWorx Server Side Javascript. Also, it may try to solve typical concurrency problems like doing an autoincrement counter.

## Compatibility

ThingWorx 7.3 and above. It's set to minimum ThingWorx 6.5 and built with ThingWorx 6.5 SDK but I didn't tested on it.

## Installation

Import the extension (ConcurrencyExtension.zip) with ThingWorx Composer Import Extension feature.

## Usage

### Mutex 

_We implemented first a mutex ThingShape with Java ReentrantLook with fair execution enabled which allows to synchronize and block thread execution. Blocking it's done at Thing's level, it means that each Thing which implements the wupMutexTS ThingShape has it's own ReentrantLook (on a lazy loading way)._

We had to change the ThingShape implementation to a ScriptFunction based approach, we left the existing Mutex via ThingShape for the record, but it's recommended to use the ScriptFunction based one, which it's more error prone. With the ThingShape implementation, if Thing Restarts exactly when you where doing the Unlock it crashes and lefts the block Mutex locked forever. You can get an almost error prone version of the ThingShape version whith a try/catch approach:

```javascript
for(var i=0;i<120;i++) { try { Things[meName].Unlock_wupMutexTS(); break; } catch(err) { pause(1000); } } 
```

but you don't get a clean code, it's slower on the remote case of a restart, and also on an extreme case can fail (after 2 minutes of trying without success).

#### Mutex Usage Samples

You just need ot use the Script Functions (Lock_wupMutexTS, Unlock_wupMutexTS, TryLock_wupMutexTS):

In order to lock a piece of code in Javascript, and ensure that only one thread its entering on it at a time for the given Thing:

```javascript
var meName = me.name;
Lock_wupMutexTS(meName);
try {
    // -- whatever code that needs to be mutex
} finally { 
    Unlock_wupMutexTS(meName);
}
```

You can also tryLock a piece of code, in order to allow one thread and only one and discard the others.
For instance it may be interesting if you have a timer which triggers once in a while and you don't want that two
consecutive triggers are executed at the same time:

```javascript
var meName = me.name;
if (TryLock_wupMutexTS(meName)) {
    try {
        // -- whatever code that needs to be mutex
    } finally { 
        Unlock_wupMutexTS(meName);
    }
} else {
    // -- The lock was already got and previous code its skipped
}
```

You can create more than one mutex per thing, just add the sub-block id after the thing name to have a more quirurgic mutex.
Each different passed "id" will create its own ReentrantLook. Sample with previous code but with a specific lock for one specific timer.

```javascript
var meName = me.name;
if (TryLock_wupMutexTS(meName+"/timer1")===true) {
    try {
        // -- whatever code that needs to be mutex
    } finally { 
        Unlock_wupMutexTS(meName+"/timer1");
    }
} else {
    // -- The lock was already got and previous code it's skipped
}
```

### Counter 

A thread safe "autoincrement" ThingShape. It provides a "counter" property and the corresponding services in order to increase (one by one) it's value.

#### Counter Usage Samples

To increase the counter value, no worries about having any amount of threads incrementing the property value, all will get it's own and unique value:

```javascript
var newCounterValue = me.Increase_wupCounterTS();
```

You can reset or reset counter value with Set_wupCounterTS method:

```javascript
me.Set_wupCounterTS({ value: 0 });
```

The counter it's stored and update on a persistent property named: counter_wupCounterTS

#### Mutex Usage Samples (OLD VERSION better to not to use it)

Just add the wupMutexTS to the Thing or ThingTemplate to whom you want to add mutex blocking features.

On the Mutex Usage samples you will see the usage of copying the me.name in memory? The reason, it's that in between your locking a ThingRestart/ThingDesignSave may happen, and if garbage collector takes place "me" it's not valid anymore as the Thing had restarted and it's another instance in memory and you will get a "zombie" mutex blocking any execution on it!

In order to lock a piece of code in Javascript, and ensure that only one thread its entering on it at a time:

```javascript
var meName = me.name;
me.Lock_wupMutexTS();
try {
    // -- whatever code that needs to be mutex
} finally { 
    for(var i=0;i<120;i++) { 
        try { 
            Things[meName].Unlock_wupMutexTS(); 
            break;
        } catch(err) { 
            pause(1000); 
        } 
    } 
}
```
You can also tryLock a piece of code, in order to allow one thread and only one and discard the others.
For instance it may be interesting if you have a timer which triggers once in a while and you don't want that two
consecutive triggers are executed at the same time:

```javascript
var meName = me.name;
if (me.TryLock_wupMutexTS()===true) {
    try {
        // -- whatever code that needs to be mutex
    } finally { 
         for(var i=0;i<120;i++) { 
            try { 
                Things[meName].Unlock_wupMutexTS(); 
                break;
            } catch(err) { 
                pause(1000); 
            } 
        } 
    }
} else {
    // -- The lock was already got and previous code its skipped
}
```

You can create more than one mutex per thing, all wupMutexTS services has an optional "id" parameter, which allows to create a more quirurgic mutex.
Each different passed "id" will create its own ReentrantLook. Sample with previous code but with a specific lock for one specific timer.

```javascript
var meName = me.name;
if (me.TryLock_wupMutexTS({ id: "timer1" })===true) {
    try {
        // -- whatever code that needs to be mutex
    } finally { 
        for(var i=0;i<120;i++) { 
            try { 
                Things[meName].Unlock_wupMutexTS({ id: "timer1" }); 
                break;
            } catch(err) { 
                pause(1000); 
            } 
        } 
    }
} else {
    // -- The lock was already got and previous code it's skipped
}
```

### Counter 

A thread safe "autoincrement" ThingShape. It provides a "counter" property and the corresponding services in order to increase (one by one) it's value.

#### Counter Usage Samples

To increase the counter value, no worries about having any amount of threads incrementing the property value, all will get it's own and unique value:

```javascript
var newCounterValue = me.Increase_wupCounterTS();
```

You can reset or reset counter value with Set_wupCounterTS method:

```javascript
me.Set_wupCounterTS({ value: 0 });
```

The counter it's stored and update on a persistent property named: counter_wupCounterTS

### Concurrency Script Functions

List of script helper functions related with this concurrency extension. This services should go on a subsystem like entity, but subsystems on ThingWorx can't be created through extensions :(

#### GetTotalActiveLocks_wupMutexTS

Returns the total active locks, in the whole ThingWorx running system.

#### GetTotalActiveWaiting_wupMutexTS

Returns the total active threads which are waiting on a lock, in the whole ThingWorx running system.

#### GetTotalThingsLocksUsage_wupMutexTS

Returns the total number of mutex created on Things (ReentranLocks), in the whole ThingWorx running system since last start.

#### Lock_wupMutexTS

Creates/Reuses a mutex with the given id and blocks execution until it gets the mutex. See [Mutex](#mutex) Section.

#### Unlock_wupMutexTS

Unlocks a mutex with the given id. See [Mutex](#mutex) Section.

#### TryLock_wupMutexTS

Creates/Reuses a mutex with the given id and blocks execution until it gets the mutex or the given timeout happens. See [Mutex](#mutex) Section.

#### IsLocked_wupMutexTS

Checks if the mutex with the given id it's locked right now.

## Build

If you need to build it, it's built with ant and java 8 on a MacOS, the scripts are on the repository. Version change it's done by hand and should be automated.

## Acknowledgments

I've started from the [code](https://community.ptc.com/t5/ThingWorx-Developers/Concurrency-Synchronisation-ConcurrentModificationException/m-p/624921) posted by [@antondorf](https://community.ptc.com/t5/user/viewprofilepage/user-id/290654) on the ThingWorx Developer Community.


## Author

[Carles Coll Madrenas](https://linkedin.com/in/carlescoll)
