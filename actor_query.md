# Actor Read Pattern with Promises

It is quite a common problem to fetch values from actors. We want to 
do that for various reasons, such as supplying external system with
data about the current state. And since we can't retrieve the data
with a simple function call, we need to do this with callbacks of
some kind and it quickly becomes messy and repetitive.

By using Promises, we can reduce the complexity a little bit,
especially by harnessing `join()` on `Promise` but it is still
quite a lot of boilerplate code required for each case.

I have collections of actors and the holder of those actors
need to fetch one or more values from each and report back to 
a caller.

Say we have

```
actor MaleActor
  let _name: String val
  
  new create( name':String ) => 
    _name = name'
    
actor FemaleActor
  let _name: String val
  
  new create( name':String ) => 
    _name = name'

    
actor Studio
  let _employed:Array[(MaleActor|FemaleActor)] = Array[(MaleActor|FemaleActor)]
  
  be addFemaleEmployee( actress':FemaleActor ) =>
    _employed.push(actress')

  be addMaleEmployee( actor':MaleActor ) =>
    _employed.push(actor')
    
```

But say that we have a Rest Server that wants to return the
employed actors and actresses of a given `Studio`. How do we do that?

Well, assuming that we don't store the name inside some collection
in the `Studio`, but instead add a query behavior to the `MaleActor`
and `FemaleActor`, such as


```
actor MaleActor
  let _name: String val
  
  new create( name':String ) => 
    _name = name'

  be name( promise: Promise[String] ) =>
    promise(_name)
    
actor FemaleActor
  let _name: String val
  
  new create( name':String ) => 
    _name = name'

  be name( promise: Promise[String] ) =>
    promise(_name)
```

Alright, that is rather clear. How do we use that in the `Studio` ?

Well, we need a behavior that takes a `Promise` to return a collection
of names, and in there we need to loop the `_employed` collection, 
create a `Promise` for each, then `join` all of those and a aggregation
code of what we need.

But all of that is very generic, so if we extract the generic bits into
a new class what we will end up with in our specific case is;

```
actor Studio
  let _employed:Array[(MaleActor|FemaleActor)] = Array[(MaleActor|FemaleActor)]
  
  be addFemaleEmployee( actress':FemaleActor ) =>
    _employed.push(actress')

  be addMaleEmployee( actor':MaleActor ) =>
    _employed.push(actor')
    
  be actors( promise: Promise[Array[String] val] ) =>
     Collector[(MaleActor|FemaleActor),String](
       _employed.values(),
       { (empl,p) => empl.name(p) },    // Call the name() behavior and a fresh Promise
       { (result) => promise(result) }  // Return the Array of results back to the  caller
    )
```

Of course, arbitrary code can be used to build the result, in case the colleccted
items doesn't fit the requesting `Prommise` as it does above. Maybe need to build
a JSON object, or create new domain object instances, or any other need. The point is
that a lot of the boilerplate needed can be hidden away. 

So, how does this magic Collector look like?

```
interface val Collectable[IN: Any #any !, OUT:Any #share]
  fun apply( c:IN, p:Promise[OUT] )

interface val Reducable[OUT:Any #share]
  fun apply( c: Array[OUT] val)
  
primitive Collector[IN: Any #any !, OUT:Any #share]
  fun apply( collection':Iterator[IN], fetch:Collectable[IN,OUT], reduce:Reducable[OUT] ) =>
    let promises = Array[Promise[OUT]]
    for c in collection' do
      let p = Promise[OUT]
      fetch(c,p)
      promises.push(p)
    end
    try
      let root = promises.pop()?
      let fulfill: Fulfill[Array[OUT] val, None] =  { (result: Array[OUT] val ) => reduce(result) }
      root.join( promises.values() ).next[None]( consume fulfill )
    else
      reduce(recover Array[OUT] end)
    end
 ```
