# asynquence

A lightweight (1.6k minzipped) API for asynchronous flow control using sequences and gates.

## Explanation

### By Example

* [Sequences & gates](https://gist.github.com/getify/5959149), at a glance.
* Example/explanation of [promise-style sequences](https://gist.github.com/jakearchibald/0e652d95c07442f205ce#comment-977119).
* More advanced example of ["nested" composition of sequences](https://gist.github.com/getify/10273f3de07dda27ebce).

### Sequences
If you want to perform two or more asynchronous tasks one after the other (like animation delays, XHR calls, etc). You want to set up an ordered series of tasks and make sure the previous one finishes before the next one is processed. You need a **sequence**.

You create a sequence by calling `ASQ(...)`. **Each time you call `ASQ()`, you create a new, separate sequence.**

To create a new step, simply call `then(...)` with a function. That function will be executed when that step is ready to be processed, and it will be passed as its first parameter the completion trigger. Subsequent parameters, if any, will be any messages passsed on from the immediately previous step.

The completion trigger that your step function(s) receive can be called directly to indicate success, or you can add the `fail` flag (see examples below) to indicate failure of that step. In either case, you can pass one or more messages onto the next step (or the next failure handler) by simply adding them as parameters to the call.

If you register a step using `then(...)` on a sequence which is already currently complete, that step will be processed at the next opportunity. Otherwise, calls to `then(...)` will be queued up until that step is ready for processing.

You can register multiple steps, and multiple failure handlers. However, messages from a previous step (success or failure completion) will only be passed to the immediately next registered step (or the next failure handler). If you want to propagate along a message through multiple steps, you must do so yourself by making sure you re-pass the received message at each step completion.

To listen for any step failing, call `or(...)` on your sequence to register a failure callback. You can call `or()` as many times as you would like. If you call `or()` on a sequence that has already been flagged as failed, the callback you specify will just be executed at the next opportunity.

### Gates
If you have two or more tasks to perform at the same time, but want to wait for them all to complete before moving on. You need a **gate**.

Calling `gate(..)` with two or more functions creates a step that is a parallel gate across those functions, such that the single step in question isn't complete until all segments of the parallel gate are complete.

For parallel gate steps, each segment of that gate will receive a copy of the message(s) passed from the previous step. Also, all messages from the segments of this gate will be passed along to the next step (or the next failure handler, in the case of a gate segment indicating a failure).

### Conveniences
There are a few convenience methods on the API, as well:

* `pipe(..)` takes one or more completion triggers from other sequences, treating each one as a separate step in the sequence in question. These completion triggers will, in turn, be piped both the success and failure streams from the sequence.

* `seq(..)` takes one or more functions, treating each one as a separate step in the sequence in question. These functions are expected to return new sequences, from which, in turn, both the success and failure streams will be piped back to the sequence in question.

    This method will also accept *asynquence* sequence instances directly.

* `val(..)` takes one or more functions, treating each one as a separate step in the sequence in question. These functions can each optionally return a value, each value of which will, in turn, be passed as the completion value for that sequence step.

    This method will also accept non-function values as sequence messages.

You can also `abort()` a sequence at any time, which will prevent any further actions from occurring on that sequence (all callbacks will be ignored). The call to `abort()` can happen on the sequence API itself, or using the `abort` flag on a completion trigger in any step (see example below).

`ASQ.messages(..)` wraps a set of values as a ASQ-branded array, making it easier to pass multiple messages at once, and also to make it easier to distinguish a normal array (a value) from a value-messages container array, using `ASQ.isMessageWrapper(..)`.

`ASQ.noConflict()` rolls back the global `ASQ` identifier and returns the current API instance to you. This can be used to keep your global namespace clean, or it can be used to have multiple simultaneous libraries (including separate versions/copies of *asynquence*!) in the same program without conflicts over the `ASQ` global identifier.

### Plugin Extensions
`ASQ.extend( {name}, {build} )` allows you to specify an API extension, giving it a `name` and a `build` function callback that should return the implementation of your API extension. The `build` callback is provided two parameters, the sequence `api` instance, and an `internals(..)` method, which lets you get or set values of various internal properties (generally, don't use this if you can avoid it).

Example:

```js
// "foobar" plugin, which injects message "foobar!"
// into the sequence stream
ASQ.extend("foobar",function __build__(api,internals){
    return function __foobar__() {
        api.val(function __val__(){
            return "foobar!";
        });

        return api;
    };
});

ASQ()
.foobar() // our custom plugin!
.val(function(msg){
    console.log(msg); // foobar!
});
```

See the `/contrib/*` plugins for more complex examples of how to extend the *asynquence* API.

The `/contrib/*` plugins provide a variety of [optional plugins](https://github.com/getify/asynquence/blob/master/contrib/README.md) as helpers for async flow controls.

### Multiple parameters
API methods take one or more functions as their parameters. `gate(..)` treats multiple functions as segments in the same gate. The other API methods (`then(..)`, `or(..)`, `pipe(..)`, `seq(..)`, and `val(..)`) treat multiple parameters as just separate subsequent steps in the respective sequence. These methods don't accept arrays of functions (that you might build up programatically), but since they take multiple parameters, you can use `.apply(..)` to spread those out.

### A+ Promises
**You should be able to use *asynquence* as your primary async flow control library, without the need for other Promises implementations.**

This lib is intentionally designed to hide/abstract the idea of Promises, such that you can do quick and easy async flow control programming without using Promises directly. As such, *asynquence* is *not [Promises/A+](http://promisesaplus.com/) compliant*, nor *should* it be, because the "promises" used are hidden underneath *asynquence*'s API.

If you are already using other Promises implementations, you *can* quite easily receive and consume a regular Promise value from some other method and wire it up to signal/control flow for an *asynquence* instance.

**However**, despite API similarities, an *asynquence* instance is **not** designed to be used *as a Promise value* passed to a regular Promises-based system. Trying to do so will likely cause unexpected behavior, because Promises/A+ suggests problematic (read: "dangerous") duck-typing for objects that have a `then()` method, as `asynquence` instances do.

## Browser, node.js (CommonJS), AMD: ready!

The *asynquence* library is packaged with a light variation of the [UMD (universal module definition)](https://github.com/umdjs/umd) pattern, which means the same file is suitable for inclusion either as a normal browser .js file, as a node.js module, or as an AMD module. Can't get any simpler than that, can it?

**Note:** The `ASQ.noConflict()` method really only makes sense when used in a normal browser global namespace environment. It **should not** be used when the node.js or AMD style modules are your method of inclusion.

## Usage Examples

Using the following example setup:

    function fn1(done) {
       alert("Step 1");
       setTimeout(done,1000);
    }

    function fn2(done) {
       alert("Step 2");
       setTimeout(done,1000);
    }

    function yay() {
       alert("Done!");
    }

Execute `fn1`, then `fn2`, then finally `yay`:

    ASQ(fn1)
    .then(fn2)
    .then(yay);

Pass messages from step to step:

    ASQ(function(done){
        setTimeout(function(){
            done("hello");
        },1000);
    })
    .then(function(done,msg1){
        setTimeout(function(){
            done(msg1,"world");
        },1000);
    })
    .then(function(_,msg1,msg2){ // basically ignoring this step's completion trigger (`_`)
        alert("Greeting: " + msg1 + " " + msg2); // 'Greeting: hello world'
    });

Handle step failure:

    ASQ(function(done){
        setTimeout(function(){
            done("hello");
        },1000);
    })
    .then(function(done,msg1){
        setTimeout(function(){
            done.fail(msg1,"world"); // note the `fail` flag here!!
        },1000);
    })
    .then(function(){
        // sequence fails, won't ever get called
    })
    .or(function(msg1,msg2){
        alert("Failure: " + msg1 + " " + msg2); // 'Failure: hello world'
    });

Create a step that's a parallel gate:

    ASQ()
    // normal async step
    .then(function(done){
        setTimeout(function(){
            done("hello");
        },1000);
    })
    // parallel gate step (segments run in parallel)
    .gate(
        function(done,greeting){ // gate segment
            setTimeout(function(){
                done(greeting,"world");             // <-- 2 gate messages
            },500);
        },
        function(done,greeting){ // gate segment
            setTimeout(function(){
                done(greeting + " mikey");          // <-- only 1 gate message!
            },100); // segment finishes first, but message still kept "in order"
        }
    )
    .then(function(_,msg1,msg2){
        // msg1 is an array of the 2 gate messages from the first segment
        // msg2 is the single message (not an array) from the second segment

        alert("Greeting: " + msg1[0] + " " + msg1[1]); // 'Greeting: hello world'
        alert("Greeting: " + msg2); // 'Greeting: hello mikey'
    });

Use `pipe(..)`, `seq(..)`, and `val(..)` helpers:

    var seq = ASQ()
    .then(function(done){
        ASQ()
        .then(function(done){
            setTimeout(function(){
                done("Hello World");
            },100);
        })
        .pipe(done); // pipe sequence output to `done` completion trigger
    })
    .val(function(msg){ // NOTE: no completion trigger passed in!
        return msg.toUpperCase(); // map return value as step output
    })
    .seq(function(msg){ // NOTE: no completion trigger passed in!
        var seq = ASQ();

        seq
        .then(function(done){
            setTimeout(function(){
                done(msg.split(" ")[0]);
            },100);
        });

        return seq; // pipe this sub-sequence back into the main sequence
    })
    .then(function(_,msg){
        alert(msg); // "HELLO"
    });

Abort a sequence in progress:

    var seq = ASQ()
    .then(fn1)
    .then(fn2)
    .then(yay);

    setTimeout(function(){
        seq.abort(); // will stop the sequence before running steps `fn2` and `yay`
    },100);

    // same as above
    ASQ()
    .then(fn1)
    .then(function(done){
        setTimeout(function(){
            done.abort(); // `abort` flag will stop the sequence before running steps `fn2` and `yay`
        },100);
    })
    .then(fn2)
    .then(yay);

## Builds

There are two utilities included which you can use to build the files.

* `./build-core.js` builds (minifies) the `asq.js` file (in the package root). The recommended way to invoke this utility is via npm:

    `npm run-script build-core`

* `contrib/bundle.js` builds `contrib.src.js` (in the package root), and then builds (minifies) `contrib.js` (in the package root). The recommended way to invoke this utility is via npm:

    `npm run-script bundle-contrib`

    By default, the build includes all the `contrib/plugin.*` plugins. You can manually specify which plugins you want, like this:

    `contrib/bundle.js any none try` (which would bundle only `any`, `none`, and `try` plugins)

    **Note:** `npm run-script ..` [doesn't *currently*](https://github.com/isaacs/npm/issues/3494) support passing the extra command line params, so you must use `contrib/bundle.js` instead of `npm run-script bundle-contrib` if you want to pick which plugins to bundle.

## License

The code and all the documentation are released under the MIT license.

http://getify.mit-license.org/
