This is a collection of Haxe libraries in the functional style intended to cover most functionality you'd need to bring the pertinent parts of your programming up front.

It covers async handling, control flow, composition, parsing, logging, errors, testing, dependency injection, configuration, scheduling and data types that can cover the full range of needs in a coherent way with a carefully constructed, consistent Api.

It's by no means complete, and not documented well presently, but the programming style is well balanced, and traces the extents of the Haxe type system well.

## Highlights

### Simple integration

```haxe
using stx.Log;
```

Each library can be used in full with a one liner: `using stx.****` 


### Wildcard Static Extensions

from `stx.Nano` upwards, there is a global reference: `__` which allows for things such as:

```haxe
using stx.Assert;

function assert(){
  var a = null;
  __.assert().exists(a)//throws error
  //or
  __.assert().that().exists().apply(a);//Forward optional error.
}
```

Creating a `Wildcard` static extension is as simple as

#### Main.hx
```haxe
  using Main;

  function insertion_point(wildcard:Wildcard){
    return new Module();
  }
  private class Module{
    public function new(){}
  }
  class Main{
    static public function main(){
      final module = __.insertion_point();//Instance of Module
    }
  }
```

### Coherent Async Story

There are fine grained return types for async handling in functional programming style, with a simple API that can 
be lifted from synchronous to asynchronous functionality very simply.

in `stx.Nano`

`Report` is an optional `Refuse` with `Alert` being the asynchronous version  

```haxe
  __.report();//produces `Happened`

  var report  = __.report(f -> f.of(OhNoes));//injects Fault, expexts `Reported<Refuse<E>>`
  var alert   = report.alert();//Future<Report<E>>
```
`Res` is either some value, or some `Refuse` (see stx.Fail) with `Pledge` being the async version.


```haxe
  __.accept(true);//produces `Accept(true)`;
  var res     = __.reject(__.fault().of(Whassat))// produces `Reject(Whassat)`;
  var pledge  = res.pledge()// Future<Res<T,E>> known as Pledge<T,E>
```

You can conditionally handle parts of the Res/Pledge structure.

either:
 //point  
`Res<R,E> -> (R -> Report<E>) -> Report<E>`  
or:   
//point
`Pledge<R,E> -> (R -> Report<E>) -> Alert<E>`

```haxe
  var res = __.accept(true);
  var eaten = res.point(
    (b:Bool) -> {
      final is_ok = some_outside_handler(b);
      return is_ok ? __.report() : __.report(f -> f.of(SomethingWentWrong));
    }
  );
```

And there are various different mechanisms to carve out error subsets, integrate to upstream systems and produce cleaned values.

```haxe
  enum SubSystemError{
    OOF;
  }
  enum SuperSystemError{
    Wot(err:SubSystemError)
  }
  function test_error(){
    final oops = __.fault().of(OOF);//Refuse<SubSystemError>;
    final upstream = oops.errate(Wot);//Refuse<SuperSystemError>;
  }
```
Each of these are monad instances with a couple extra methods for integration.

### Advanced Programming Space.     
    
    
Beyond Futures and Streams lie `stx.Fletcher`, `stx.Coroutines` and `stx.Proxy`, capable of sophisticated interpolation and ordering schemes over bi-directional streaming data, and a complete set of primitives to describe any operation existing between functions, the ordering of their effects, and the scheduling of those effects.

```haxe
  //Describes a composable arrowlet/function that can return work for a scheduler to perform.
  typedef FletcherDef<P,R> = P -> (Void -> R) -> Work
```

### Highly Speciated Errors

Much of Haxe's strengh lies in integrating with pre-existing systems. `stx.Fail` allows for fine grained control over known and unknown errors and provides the discipline to take on any and all states of your program.

#### Main.hx
```haxe
  using Main;
  using stx.Nano;

  enum ErrorEnum{
    OhNoes;
  }
  class HiddenDigest extends Digest{
    public function new(){
      super("completely_unique_id","could be from a catch statement",500);
    }
    //the __.fault().explain function injects a lambda with a `Digests` wildcard that you can use.
    static public function hidden_digest(wildcard:Digests){
      return new HiddenDigest();
    }
  }
  function error_test(){
    final refuse : Refuse<ErrorEnum> = __.fault().of(OhNoes)//Error at exact Pos to be passed around;
    final digest : Refuse<ErrorEnum> = __.fault().explain(e -> e.hidden_digest())//compatible with Refuse<ErrorEnum> but hidden for many of the composition functions as considered to be unrecoverable.
    final composition = refuse.concat(digest);//collect all the errors
  }
```