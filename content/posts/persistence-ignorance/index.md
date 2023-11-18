---
title: "Persistence (Almost) Ignorance"
date: 2022-12-11T22:40:29+07:00
draft: false
author: Teerawat Wuttiwat
---

{{<figure src="images/floppy-disks.png" width="600"
          alt="Floppy Disks Image" title="Old School Persistance Style">}}

Welcome back to my [building webapp with f# series](../intro-fs-webapp-series/).
Last week, I was kinda exausted from writing and capturing the screenshot for the safe stack post.
So this week, we will do something simple. We will not add any new functionality. We will do following things.

1. Extract Persistence Layer
2. Implement new NoSQL storage
3. Create Storage class library

Why Persistance Ignorance?
==========================

If you know about Domain Driven Design already, you could skip this section. But if you don't, please keep along.
The concept of this is that the persistence storage should not interwind with the Core Domain logic.
You should be able to switch and change persistence technology as you see fit in which you'll see in this post. 

You will also found that sometimes there are no golden rule to achive all the thing. So sometimes, change of core domain logic is needed. But we'll try our best to change as little as possible.


Extract Persistence Api
-----------------------
If you look at the existing code, you will find that we are using the Storage module for load and restore BoQ items. You will find that the Api function sometimes read data directly from the Storage resize array. This will be a problem for coupling, the Api layer should not know how the implementation is done. 

We should give the api layer, the Interface and let the functions cal that storage Interface. 
Let's start by creating IStorageApi

```fsharp
type IStorageApi =
    abstract member getBoQItems : unit -> Result<BoQItem list,string>
    abstract member addBoQItem : boqItem:BoQItem -> Result<unit, string>
    abstract member updateBoQItem : boqItem:BoQItem -> Result<unit, string>
    abstract member deleteBoQItem : itemId:Guid -> Result<Unit, string>
    abstract member loadFactorFTable : unit -> FactorFTable
```

Next, we will create new Class ResizeArray which will implement this interface

```fsharp
type ResizeArrayStorage() =
    let boqItems = ResizeArray<BoQItem>()
    do
        let qty = Quantity (10.0, "m^2")
        let material = Material { Name = "Pool Tile"; Unit = "m^2"; UnitCost = 100.0 }
        let labor = Labor { Name = "Do Tiling"; Unit = "m^2"; UnitCost = 50.0 }
        let defaultItem = BoQItem.tryCreate (Guid.NewGuid()) "Pool Tile" qty material labor
        match defaultItem with
        | Ok defaultItem' ->
            boqItems.Add defaultItem' |> ignore
        | _ -> ()

    interface IStorageApi with
        member _.getBoQItems() =
            boqItems |> List.ofSeq |> Ok

        member _.addBoQItem(boqItem) =
            boqItems.Add boqItem
            Ok ()

        member _.updateBoQItem(boqItem) =
            let index = boqItems.FindIndex( fun x -> x = boqItem)
            boqItems[index] <- boqItem
            Ok ()

        member _.deleteBoQItem(itemId) =
            let index = boqItems.FindIndex( fun x -> x |> BoQItem.value |> fun y -> y.Id = itemId)
            boqItems.RemoveAt(index)
            Ok ()

        member _.loadFactorFTable() =
            FactorFTable [(10,1.1); (100,1.5); (1000, 1.9)]
```

I decide to go with class and interface instead of using record of function like ICoCoTenderApi. Since it's likely there will be intialization for each storage type. Having that done in the class constructor should simplify the caller code.

Now, we have our ResizeStorage ready. Let's use it in our cocoTenderApi definition.

```fsharp
let cocoTenderApi (storage:IStorageApi) = {
    ...
}
```

We just supply the storage api to the application api. The cocoTenderApi implementation will not know what is underlying storage. It will just use the given storage api.

Replace all the storage related function to use this new api such as.
```fsharp
let! boqItems = storage.getBoQItems()
```

Now we need to create the actual storage class and send to the api implementation. 
Go to webApp definition
```fsharp
let webApp =
    Remoting.createApi ()
    |> Remoting.withRouteBuilder Route.builder
    |> Remoting.fromValue (cocoTenderApi (ResizeArrayStorage()))
    |> Remoting.buildHttpHandler
```

Check the result. The web application should run as usual.


Implement new NoSQL storage
---------------------------

Initially, I plan to use SQL database but the safe stack documentation suggests **LiteSharpDB** as a simpler solution.
When I think about it, the CoCoTender application does not need integrity that much. Since all we need is just saving list of BoQ items ins and outs of the storage. So No SQL database is suffice for our application at the moment. And it's my first time using no sql database as well so I think it should be fun.

When working with new library, I like to play with it in script file first. 
We will try to implement all StorageApi operation in script file then move it to actual project when it works.

Go to scripts/scratch.fsx.

Start by importing the library we need.
See that we also reference to our CoCoTender.Domain library.

```fsharp
#r "nuget:LiteDB.FSharp"
#r "nuget:FsToolkit.ErrorHandling"
#r @"C:\projs\CoCoTender\src\CoCoTender.Domain\bin\Debug\net6.0\CoCoTender.Domain.dll"
```

Next, create database file in the script folder.

```fsharp
let database =
      let mapper = FSharpBsonMapper()
      let connStr = $"Filename={__SOURCE_DIRECTORY__}\\CoCoTender.db;mode=Exclusive"
      new LiteDatabase (connStr, mapper)
```

Then, implement all database operations. Coming from the SQL database, I found that the No SQL one is more like a magic. I mean if you follow all persistent requirement then suddenly all serialization id done for you. No need for schema. This maybe a good or bad thing though; depends on the application a hand.

Here is the implementation. First, we will list all the boq items.

```fsharp
let boqitems = database.GetCollection<BoQItem> "boqitems"
boqItems.FindAll () |> List.ofSeq
```

Next is the insert operation.

```fsharp
boqitems.Insert boqItem
```
This is super easy but ,like I said before, everything comes with a prices. First the Record type must have Id or id properties with value either int or Guid. Luckily, we already have that Id in our BoQItem. so this is not a problem. 

But there is one requirement that we cannot ignore(pun intended). The record could not be **private** or else LiteDB will not be able to serialize the type. 

This is bad since we are breaking the domain. I don't like this but at the cost of doing serialization code by myself. I think I am going to ignore that and go with the requirement. 

Here is the change from
```fsharp
type private BoQItem = {
```

to
```fsharp
[<CLIMutable>]
type private BoQItem = {
```

Adding CLIMutable is also required for the LiteDB as well.
Too much mumble, let's do with update and delete

```fsharp
let id = BsonValue(Guid("...")) 
let boqItem = boqitems.FindById(id) // Find item by id

let boqItem = { boqItem with Description = "Foo" }
boqitems.Update(boqItem) // Update 

boqitems.Delete("...") // Delete
```

We also has issue with FactorF type since the type is just tuple of float which is not directly support by LiteDB. We fix the issue by creating intermediate type for FactorF called FactorFRec

```fsharp
type  FactorFRec = {
    Id: int
    DirectCost: float
    FactorF: float
}
```

And changes the load factor f to map from FactorFRect to tuple

```fsharp
let factorFs = database.GetCollection<FactorFRec> "boqitems"
factorFs.Insert { Id = 1; DirectCost = 10.0; FactorF = 1.1 } |> ignore
factorFs.FindAll() |> List.ofSeq |> List.map (fun x -> x.DirectCost,x.FactorF)
```

I am happy with how to work with the library. Now, I will move the code to Server code now. It's quite simple. We just need to implement function for IStorageApi with the code we try in the script.

```fsharp
type LiteDBStorage () =

    let database =
        let mapper = FSharpBsonMapper()
        let connStr = $"Filename=CoCoTender.db;mode=Exclusive"
        new LiteDatabase (connStr, mapper)

    let boqItems = database.GetCollection<BoQItem> "boqItems"
    let factorFs = database.GetCollection<FactorFRec> "factorFs"

    do
        if factorFs.Count() = 0 then
            factorFs.Insert { Id = 1; DirectCost = 10.0; FactorF = 1.1 } |> ignore
            factorFs.Insert { Id = 2; DirectCost = 100.0; FactorF = 1.5 } |> ignore
            factorFs.Insert { Id = 3; DirectCost = 1000.0; FactorF = 1.9 } |> ignore


    interface IStorageApi with
        member _.getBoQItems () =
            boqItems.FindAll() |> List.ofSeq |> Ok

        member _.addBoQItem(boqItem) =
            boqItems.Insert boqItem |> ignore
            Ok ()

        member _.updateBoQItem(boqItem) =
            if boqItems.Update boqItem then Ok ()
            else Error "LiteDbStorage:Could not update boq item"

        member _.deleteBoQItem(itemId) =
            if boqItems.Delete itemId then Ok ()
            else Error "LiteDbStorage:Could not delete boq item"

        member _.loadFactorFTable () =
            factorFs.FindAll() |> List.ofSeq |> List.map (fun x -> x.DirectCost,x.FactorF) |> FactorFTable
```

Also, change the webApp initialization to use LiteDBStorage class.

```fsharp
    |> Remoting.fromValue (cocoTenderApi (LiteDBStorage()))
```

Now, checking the running application. Everything should work as expected.

We are at the last step now. When I see a lot of storage api scattered around in the Server code, I feel unconfortable. I think this code should belong into other place. The Api layer should be just orchestration between different component. 

Let's extract the storage code to dedicated class library.

```fsharp
cd src
dotnet new classlib -lang F# -o CoCoTender.Storage
dotnet add reference ../CoCoTender.Domain
cd ..
dotnet paket add LiteSharpDB.FSharp -p CoCoTender.Storage
dotnet paket add FsToolkit.ErrorHandling -p CoCoTender.Storage
dotnet sln add src/CoCoTender.Storage
```

We will organize the code such that 
- Common type such as IStorageApi interface will be in CoCoTender.Storage namespace
- Each type of storage will have it's own file and module. For example for LiteDB, we will create LiteDB.fs file with module CoCoTender.Storage.LiteDB.

I think you can do that by yourself. Don't forget that all the new file should be include in the CoCoTender.Storage.fsproj with correct order. Common.fs should be at the top.

Now, we just need to add reference to Storage library in the Server file and open the module

```fsharp
cd src/Server
dotnet add reference ../CoCoTender.Storage
```

That's it for the simple persistence layer. You can get all the finished code by cloning tag v0.4.0.

```fsharp
git clone -b v0.4.0 https://github.com/twuttiwat/CoCoTender 
```

Good luck and see you at another post.