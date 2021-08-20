# ShadowProxy

What is that? Objects created like proxies, with the handler traps set internally from the object instantiation.

## Why?

Today, proxies are heavily used in membrane systems to control objects sent to execution over sandboxed code. 

The usage of handlers remains loose and the Proxy instance allow changes over the existing traps and runs an excessive amount of lookups.

This proposal aims to provide a new proxy-like object that stores a snapshot of the handler object during the proxy instantiation, creating a simplified access to handler traps after instantiation, without any need to freeze a handler object for it.

Proxies are not often seen in many applications, but there are several kinds of applications, beyond niche, with a heavy usage of Proxies when they are necessary. Reactive-like frameworks, such as LWC and Vue.js, are popular examples. Sandbox Code execution frameworks and their membranes, are also applications that often use a high amount of Proxies. 

Each new instance of a proxy will have exotic internals might trigger from at least one to many lookups to the proxies internal handler. The lookups for the handlers are user observable and represent an extra step. The handler is a 3rd object beyond the proxy itself and its target. This handler object can be avoided with the Shadow Proxy, even more if a trap is not defined during instantiation. The handlers are also dynamic to the proxies. Any changes to its properties, including those from its inheritance chain, will affect the subsequent Proxies using it. A Shadow Proxy won't reuse the handler object after the instance is created, only the traps are stored.

## Status Quo

In order to preserve a Proxy consistent, it's common that a handler is a fully frozen Object. By fully frozen this means a frozen object that inherits to a fully immutable prototype, usually null.


```js
const target = {/*...*/};
const handler = Object.create(null);
handler.set = (...args) => {/*...*/};

const p = new Proxy(target, handler);

// Additional traps can be added after the proxy instantiation
handler.get = (target, prop, receiver) => {
  console.log(`get ${prop}`);
  return Reflect.get(target, prop, receiver);
};

// handler is frozen, preventing the addition or modification of traps
Object.freeze(handler);

p.foo; // undefined
// Logs: get foo
```


The lookup may become excessive, all possible traps are still analyzed:


```js
const target = {/*...*/};
const handler = Object.create(null);

Object.freeze(handler);

const handlerProxy = new Proxy(handler, {
  get(target, prop, receiver) {
    console.log(`get ${prop}`);
    return Reflect.get(target, prop, receiver);
  }
});

const p = new Proxy(target, handlerProxy);

// The original handler is permanently empty, but
// the Proxy still runs lookups over it
p.foo = 42;
// Logs 3 lookups:
// get set
// get getOwnPropertyDescriptor
// get defineProperty
```

If the proxies have a fully frozen handler, all these lookups become unnecessary more than once. Word from implementers, there is no caching over the lookups for proxies.

Another not amazing fact, for each regular Proxy created, the handler objects are kept alive depending on the Proxy object they are used, and they can't be collected. This means a Proxy instance adds footprint for at least two other objects (the target and the handler).

## This is not [Transparent Proxies](https://sankhs.com/static/tproxy-ecoop15.pdf)

Today with Proxies you have a distinction between proxies and their target objects:

```
const target = {/*...*/};
const handler = {/*...*/};
const p = new Proxy(target, handler);
target === p; // false

const set = new Set();
set.add(target);
set.has(p); // false
```

Transparent Proxies aim to equalize values of the proxies and their respective target. Shadow Proxies don't aim to equalize those values.

## The Shadow Proxy Constructor

There are two options to implement the Shadow Proxy constructor:


* `ShadowProxy(target, handler)`, just adding a new name to the global.
* `Proxy.shadow(target, handler)`, with precedent on `Proxy.revocable`


For illustration purposes, this document is using `ShadowProxy`. The name and namespace remains up to discussion.

## Instantiation of a Shadow Proxy

### Shadow Proxy Handler Methods

|Internal Method	|Handler Method	|
|---	|---	|
|[[GetPrototypeOf]]	|`getPrototypeOf`	|
|[[SetPrototypeOf]]	|`setPrototypeOf`	|
|[[IsExtensible]]	|`isExtensible`	|
|[[PreventExtensions]]	|`preventExtensions`	|
|[[GetOwnProperty]]	|`getOwnPropertyDescriptor`	|
|[[DefineOwnProperty]]	|`defineProperty`	|
|[[HasProperty]]	|`has`	|
|[[Get]]	|`get`	|
|[[Set]]	|`set`	|
|[[Delete]]	|`deleteProperty`	|
|[[OwnPropertyKeys]]	|`ownKeys`	|
|[[Call]]	|`apply`	|
|[[Construct]]	|`construct`	|

### ShadowProxy (target, handler)

When `ShadowProxy` is called with arguments target and handler, it performs the following steps:

1. If NewTarget is undefined, throw a TypeError exception.
2. Return ? ShadowProxyCreate(target, handler).

### ShadowProxyCreate (target, handler)

The abstract operation `ShadowProxyCreate` takes arguments target and handler. It is used to specify the creation of new Proxy exotic objects. It performs the following steps when called:

1. If [Type](https://tc39.es/ecma262/#sec-ecmascript-data-types-and-values)(target) is not Object, throw a TypeError exception.
2. If [Type](https://tc39.es/ecma262/#sec-ecmascript-data-types-and-values)(handler) is not Object, throw a TypeError exception.
3. Let P be ! [MakeBasicObject](https://tc39.es/ecma262/#sec-makebasicobject)(« [[ShadowProxyHandler]], [[ShadowProxyTarget]] »).
4. Set P's essential internal methods, except for [[Call]] and [[Construct]], to the definitions specified in the Shadow Proxy Handler Methods table[.](https://tc39.es/ecma262/#sec-proxy-object-internal-methods-and-internal-slots)
5. If [IsCallable](https://tc39.es/ecma262/#sec-iscallable)(target) is true, then
    1. Set P.[[Call]] as specified in [10.5.12.](https://tc39.es/ecma262/#sec-proxy-object-internal-methods-and-internal-slots-call-thisargument-argumentslist)*
    2. If [IsConstructor](https://tc39.es/ecma262/#sec-isconstructor)(target) is true, then
        1. Set P.[[Construct]] as specified in [10.5.13](https://tc39.es/ecma262/#sec-proxy-object-internal-methods-and-internal-slots-construct-argumentslist-newtarget).*
6. Set P.[[ShadowProxyTarget]] to target.
7. Set P.[[ShadowProxyHandler]] to a new empty List.
8. For each ***name*** of the Handler Methods listed ins the Shadow Proxy Handler Methods table,
    1. Let ***trap*** be ? GetMethod(handler, name).
    2. If ***trap*** is not undefined, then
        1. Add an internal slot with the respective Internal Method name matching the given Handler Method.
        2. Set the value of this new internal slot to ***trap***.
9. Return P.

## Handler Snapshots

A new Shadow Proxy should use a handler object just once, storing any current available trap as an internal of a new object.

This new object should be almost transparent as an object. It's still a new identity compared to the original target, but the available traps are set in the Shadow Proxy during its instantiation. A copy of the handler functions is set in internal slots of the new proxy and the exotics of the shadow proxy limit looking up at these internals.

The exotics internals of a shadow proxy are pretty similar to the [regular proxy exotic internals](https://tc39.es/ecma262/#sec-proxy-object-internal-methods-and-internal-slots), modulo the observable lookups.

### Regular Proxy [[Get]]

[Image: Screen Shot 2021-08-18 at 11.25.49 AM.png]

### Shadow Proxy [[Get]]

1. [Assert](https://tc39.es/ecma262/#assert): [IsPropertyKey](https://tc39.es/ecma262/#sec-ispropertykey)(P) is true.
2. If O has no [[ShadowProxyHandler]] internal, throw a **TypeError** exception.
3. Let handler be O.[[ShadowProxyHandler]].
4. [Assert](https://tc39.es/ecma262/#assert): [Type](https://tc39.es/ecma262/#sec-ecmascript-data-types-and-values)(handler) is a List whose elements are handler trap functions.
5. Let target be O.[[ShadowProxyTarget]].
6. If handler has no [[Get]] internal, then
    1. Return ? target.[[Get]](P, Receiver).
7. Let trap be hander.[[Get]].
8. Let trapResult be ? [Call](https://tc39.es/ecma262/#sec-call)(trap, handler, « target, P, Receiver »).
9. Let targetDesc be ? target.[[GetOwnProperty]](P).
10. If targetDesc is not undefined and targetDesc.[[Configurable]] is false, then
    1. If [IsDataDescriptor](https://tc39.es/ecma262/#sec-isdatadescriptor)(targetDesc) is true and targetDesc.[[Writable]] is false, then
        1. If [SameValue](https://tc39.es/ecma262/#sec-samevalue)(trapResult, targetDesc.[[Value]]) is false, throw a TypeError exception.
    2. If [IsAccessorDescriptor](https://tc39.es/ecma262/#sec-isaccessordescriptor)(targetDesc) is true and targetDesc.[[Get]] is undefined, then
        1. If trapResult is not undefined, throw a TypeError exception.
11. Return trapResult.

## Examples of ShadowProxy

```js
const target = {/*...*/};
const handler = Object.create(null);

handler.get = (target, prop, receiver) => {
  console.log(`get ${prop}`);
  return Reflect.get(target, prop, receiver);
};

const p = new ShadowProxy(target, handler);

// Additional or modified traps won't affect the new proxy
handler.get = (target, prop, receiver) => {
  console.log(`WELL WELL WELL! get ${prop}`);
  return Reflect.get(target, prop, receiver);
};

p.foo; // undefined
// Logs: get foo
```

```js
const target = {/*...*/};
const handler = Object.create(null);

Object.freeze(handler);

const handlerProxy = new Proxy(handler, {
  get(target, prop, receiver) {
    console.log(`get ${prop}`);
    return Reflect.get(target, prop, receiver);
  }
});

const shadow = new ShadowProxy(target, handlerProxy);

// The original handler is permanently empty, and
// the ShadowProxy won't runs lookups over it for
// traps that were not initially defined
shadow.foo = 42;
// Logs no lookups:

const regular = new Proxy(target, handlerProxy);

// Using a regular Proxy, the lookups would be:
regular.foo = 42;
// Logs 3 lookups:
// get set
// get getOwnPropertyDescriptor
// get defineProperty
```

## Lookup and Current Usage Data

In an example using an application in production, a quick overview shows the lookup number being high. A 3 clicks interaction on a page of a ticket system, shows that a framework system has the `get` trap is invoked 74828 times and `getOwnPropertyDescriptor` is invoked 29988 times.

> __ACTION__: Grab more data

## Questions

* Does the ShadowProxy needs a revocable capability?
* Should the new shadow proxy instance also inherit from the other object's internals for a better transparency with the target?

## Maintain this proposal repo

  1. Make your changes to `spec.emu` (ecmarkup uses HTML syntax, but is not HTML, so I strongly suggest not naming it ".html")
  1. Any commit that makes meaningful changes to the spec, should run `npm run build` and commit the resulting output.
  1. Whenever you update `ecmarkup`, run `npm run build` and commit any changes that come from that dependency.

  [explainer]: https://github.com/tc39/how-we-work/blob/HEAD/explainer.md
