#### No.1
```Swift
 protocol is                     {}
 protocol possible               {}

 struct Everything: is, possible {}
```

#### No.2
```Swift
 struct Everything {
     @ResultDiscardable
     func is(x: ()-> Bool) 
          ->Bool { return true }
 }
 let possible = true

 Everything().is{ possible }
```

#### No.3 
```Swift
 typealias possible = Any
 func fun(){
     let everything: Any? 
     guard everything as possible
           { return }

     print("It's true")
 }
```

你中意哪一款？

PS：目前还没有在编译器验证过，明天看看。
