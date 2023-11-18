---
title: "Unit Testing and Console Application"
date: 2022-11-19T11:57:43+07:00
draft: false
author: Teerawat Wuttiwat
---

Good day. Sorry, I am back after busy weeks.
To bring back our past weeks, we did finish domain modeling in fsx file. 

This week we will start touching the road. We create class library from our script file, do unit testing for our library, and finally let the client test our functionality with our console application.

So many things to do in this post, let's start... 

Class Library
=============
Since we are going to create real application this time, we should create our solution before creating any project. I like to use command line so I follow script from Ian Russel as follow:

```bash
dotnet new sln -o CoCoTender
dotnet new gitignore
cd CoCoTender
mkdir src
cd src
dotnet new classlib -lang F# -o CoCoTender.Domain
cd ..
dotnet sln add src/CoCoTender.Domain
dotnet build
```

There should be no error after you run 'dotnet build' command.
Also, I did create gitignore for entire solution by using nifty dotnet new command. That's awesome.

Like what we did in the past, we will do it one by one. 

1. Copy BoQItem module from scratch.fsx file to Library.fs. 
Here is the code:

```fsharp
namespace CoCoTender.Domain

type Quantity = Quantity of float*string
type UnitCost = 
  {
    Name : string
    UnitCost : float
    Unit : string
  }
type MaterialUnitCost = Material of UnitCost
type LaborUnitCost = Labor of UnitCost

type BoQItem = private {
    Description : string
    Quantity : Quantity
    MaterialUnitCost : MaterialUnitCost
    LaborUnitCost : LaborUnitCost
    TotalCost : float
  }

module BoQItem =
  let calcCost qty unitCost =
    match qty, unitCost with
    | Quantity (qty, qtyUnit), {UnitCost = unitCost; Unit = unitCostUnit} 
      when qtyUnit = unitCostUnit ->
        (qty * unitCost) |> Some
    | _ -> None

  let tryCalcItemCost qty (Material materialUnitCost) (Labor laborUnitCost) =  
    match calcCost qty materialUnitCost, calcCost qty laborUnitCost with
    | Some materialCost, Some laborCost -> Some (materialCost + laborCost)
    | _ -> None

  let isValid desc qty materialUnitCost laborUnitCost =
    match qty, materialUnitCost, laborUnitCost with
    | Quantity (_, qtyUnit), Material {Unit = materialUnit}, Labor {Unit = laborUnit} ->
      qtyUnit = materialUnit && qtyUnit = laborUnit  

  let tryCreate desc qty materialUnitCost laborUnitCost = 
    if isValid desc qty materialUnitCost laborUnitCost  then 
      tryCalcItemCost qty materialUnitCost laborUnitCost
      |> Option.map(fun cost -> {
          Description = desc
          Quantity = qty
          MaterialUnitCost = materialUnitCost
          LaborUnitCost = laborUnitCost
          TotalCost = cost
        })
    else
      None

  let value item = 
    {| 
      Description = item.Description
      Quantity = item.Quantity
      MaterialUnitCost = item.MaterialUnitCost
      LaborUnitCost = item.LaborUnitCost
      TotalCost = item.TotalCost
    |}

```
It's the exact same code as script file. Just add namespace CoCoTender.Domain at the top.

Next, we need to move our test code from our script file to unit testing project. Here is the command to create the project. 

```bash
mkdir tests
cd tests
dotnet new xunit -lang F# -o CoCoTender.DomainTests
dotnet add reference ../../src/CoCoTender.Domain
dotnet add package FsUnit
dotnte add Package FsUnit.XUnit
cd ..
dotnet sln add tests/CoCoTender.DomainTests
dotnet test
```

The result should show that all tests are passed. 
Let's create our firs test.

```fsharp
module Tests

open System
open Xunit
open FsUnit
open CoCoTender
open CoCoTender.Domain

module ``Create boq item`` =

    let desc = "Pool Item"
    let qty = Quantity (10.0, "m^2")
    let material = Material { Name = "Big Tile"; UnitCost = 100.0; Unit = "m^2" }
    let labor = Labor { Name = "Do Tiling"; UnitCost = 50.0; Unit = "m^2" }
    let totalCost = 10*100 + 10*50

    [<Fact>]
    let ``should succeed if Quantity, Material and Labor use the same Unit`` () = 
        let result = BoQItem.tryCreate desc qty material labor
        match result with
        | Some item -> 
            let itemVal = item |> BoQItem.value
            itemVal.Description |> should equal desc
            itemVal.Quantity |> should equal qty
            itemVal.MaterialUnitCost |> should equal material
            itemVal.LaborUnitCost |> should equal labor
        | None -> Assert.Fail "Could not create BoQItem"

    [<Fact>]
    let ``should fail if Quantity, Material and Labor have different Units`` () = 
        let material' = material |> fun (Material m) -> {m with Unit = "m"} |> Material
        let labor' = labor |> fun (Labor l) -> {l with Unit = "cm"} |> Labor

        let result = BoQItem.tryCreate desc qty material' labor'
        match result with
        | None -> Assert.True(true)
        | Some _ -> Assert.Fail "Should not create BoQItem with different Unit"        

    [<Fact>]
    let ``should always calculate total cost when create`` () = 
        let result = BoQItem.tryCreate desc qty material labor
        match result with
        | Some item -> 
            let itemVal = item |> BoQItem.value
            itemVal.TotalCost |> should equal totalCost
        | None -> Assert.Fail "Could not create BoQItem" 
```

There are 3 tests we want to verify when creating boq item.
1. It should succeeded when all units are matched.
2. It should failed when some units are not matched.
3. It should always calculate total cost when create new item.

Run
```bash
dotnet test
```
To see if all tests are passed.

We are done with item creation logic. Next, we will add updating boq function from the database.

```fsharp
  let updateDesc newDesc item = { item with Description = newDesc }

  let tryUpdateQty newQty item = 
    tryCalcItemCost newQty item.MaterialUnitCost item.LaborUnitCost
    |> Option.map (fun cost -> 
      {
        item with
          Quantity = newQty
          TotalCost = cost
      })

  let tryUpdateMaterialUnitCost (Material newUnitCost) item =
    let newMaterialUnitCost = Material newUnitCost
    tryCalcItemCost item.Quantity newMaterialUnitCost item.LaborUnitCost
    |> Option.map (fun cost -> 
      {
        item with
          MaterialUnitCost = newMaterialUnitCost
          TotalCost = cost
      })

  let tryUpdateLaborUnitCost (Labor newUnitCost) item =
    let newLaborUnitCost = Labor newUnitCost
    tryCalcItemCost item.Quantity item.MaterialUnitCost newLaborUnitCost 
    |> Option.map (fun cost -> 
      {
        item with
          LaborUnitCost = newLaborUnitCost
          TotalCost = cost
      })
```

Don't forget to build the library first before create another unit tests. But the ionide extension should show any error immediately without compilation though.

Here is the unit tests for the update functions:

```fsharp
module ``Update boq item`` =

    let desc = "Pool Item"
    let qty = Quantity (10.0, "m^2")
    let material = Material { Name = "Big Tile"; UnitCost = 100.0; Unit = "m^2" }
    let labor = Labor { Name = "Do Tiling"; UnitCost = 50.0; Unit = "m^2" }
 
    let item = 
        match BoQItem.tryCreate desc qty material labor with
        | Some item' -> item'
        | _ -> failwith "Could not create item for updating"

    let totalCost = item |> BoQItem.value |> fun x -> x.TotalCost

    [<Fact>]
    let ``with quantity should always re-calculate total cost`` () =
        let result = item |> BoQItem.tryUpdateQty (Quantity (20, "m^2"))
        match result with
        | Some item' -> 
            let itemVal' = item' |> BoQItem.value
            itemVal'.TotalCost |> should equal (totalCost * 2.0)
        | _ -> Assert.Fail "Could not update quantity" 

    [<Fact>]
    let ``with description should not update total cost`` () =
        let result = item |> BoQItem.updateDesc "New Pool" 
        let resultVal = result |> BoQItem.value
        resultVal.TotalCost |> should equal totalCost
```

The explanation in function name should be suffice. What we want here is to make sure that whenever there are changes in boq item, recalculation should occur.

The last unit testing is for estimating total cost of the project.

```fsharp
module ``Estimate construction cost`` =

    open Project

    let loadFactorFTableFn = 
        function () -> FactorFTable [(10,1.1); (100,1.5); (1000, 1.9)]

    [<Fact>]
    let ``should return correct result`` () =
        let result = estimateCost loadFactorFTableFn [item] 
        result |> should equal 2000

    [<Fact>]
    let ``should not round when less than one thousand`` () =
        let smallPool = 
            match item |> BoQItem.tryUpdateQty (Quantity (1.0, "m^2")) with
            | Some item' -> item'
            | _ -> failwith "Could not update quantity"
        let result = estimateCost loadFactorFTableFn [smallPool] 
        result |> should (equalWithin 0.11) 228.33
```

As you can see, all test functions are originated from our script file. We just organice into module and function to make the intention of the tests clearer.

Console
========

I found that it's best to show tangible result to the users as soon as I can. Since, creating whole webapp might take too long at this stage. So I think it's best to start with just console application.

The goal of this console is to let client test the input. The input should be easy for client to create and at the same time should not take too long for us to code. I think Csv files are the optimal choice for this since it is very easy to create using just Excel or Google Sheet. Also there are f# type provider just for this.

The input files we are going to create for testing are for BoQ item and Factor F table.
Here is the format of these 2 files:

BoQ Items

| Description | Quantity | Unit | Material | MaterialUnit | MaterialUnitCost | Labor | LaborUnit | LaborUnitCost |
| ----------- | -------- | ---- | -------- | ------------ | ---------------- | ----- | --------- | ------------- |
| Pool Tile   |  10      | m^2  | Big Tile |    m^2       |     100          | Do Tiling |  m^2  |      50       |

Factor F Table

| DirectCost | FactorF |
| ---------- | ------- |
|     10     |    1.1  |
|    100     |    1.5  |
|   1000     |    1.9  |

Again, we will start with fsx script.

```fsharp
#r "nuget: FSharp.Data, 5.0.2"
#r @"C:\projs\CoCoTender\src\CoCoTender.Domain\bin\Debug\net6.0\CoCoTender.Domain.dll"

open FSharp.Data
open CoCoTender.Domain
open BoQItem
open Project

[<Literal>]
let ResolutionFolder = __SOURCE_DIRECTORY__

type BoQItems = CsvProvider<"boq.csv", ResolutionFolder=ResolutionFolder>

let toBoQItem (row:BoQItems.Row) =
  let qty = Quantity (row.Quantity, row.Unit.Trim())
  let material = 
    {
        Name = row.Material
        Unit = row.MaterialUnit.Trim()
        UnitCost = row.MaterialUnitCost
    }
    |> Material

  let labor = 
    {
      Name = row.Labor
      Unit = row.LaborUnit.Trim()
      UnitCost = row.LaborUnitCost
    }
    |> Labor

  BoQItem.tryCreate row.Description qty material labor

let loadBoQFile (filePath:string) =
  BoQItems
    .Load(__SOURCE_DIRECTORY__ + "/boq.csv")
    .Rows
  |> Seq.choose toBoQItem 
  |> List.ofSeq

open Project

type FactorFFile = CsvProvider<"FactorF.csv", ResolutionFolder=ResolutionFolder>

let loadFactorFFile (filePath:string) () = 
  FactorFFile
    .Load(filePath)
    .Rows
  |> Seq.map (fun x -> x.DirectCost,x.FactorF) 
  |> List.ofSeq 
  |> FactorFTable

let boqFile = __SOURCE_DIRECTORY__ + "/boq.csv"
let factorFFile = __SOURCE_DIRECTORY__ + "/FactorF.csv"
let cost = estimateCost (loadFactorFFile factorFFile) (loadBoQFile boqFile) 

match fsi.CommandLineArgs |> List.ofArray with 
| _::boqFile::factorFFile::_ -> 
  let cost = estimateCost (loadFactorFFile factorFFile) (loadBoQFile boqFile) 
  printfn $"Estimate Cost is {cost}"
| _::_ -> 
  let boqFile = __SOURCE_DIRECTORY__ + @"\boq.csv"
  let factorFFile = __SOURCE_DIRECTORY__ + @"\FactorF.csv"
  printfn "%s" boqFile
  printfn "%s" factorFFile
  let cost = estimateCost (loadFactorFFile factorFFile) (loadBoQFile boqFile) 
  printfn $"Estimate Cost is {cost}"
| _ -> 
  printfn "Should not be here"
```

We use Csv Type Provider to read data from csv files and transform each csv row to designated type (BoQItem and FactorFTable).
The domain code could be referenced in the script by referring to the class library dll (CoCoTender.Domain.dll).
The transformation is straightforward, so I will skip that.

The interesting bit of code is the way that we can actually create console application by just using script file. 
The **fsi.CommandLineArgs** will read argument from the command line. For simplicity, we will assume that the first argument will be the path to boq item file and the second one will be for factor f file. If user does not specify the path, we will just use the default one

We can test this script by running following command

```bash
dotnet fsi scratch.fsx <path to boq.csv> <path to factorf.csv>
```
The result should be the same as unit test since we use the same test data.

Now we are ready to create actual console application.
Type following commands to create console project.

```bash
cd src
dotnet new console -lang F# -o CoCoTender.Console
cd CoCoTender.Console
dotnet add reference ../CoCoTender.Domain
dotnet add package FSharp.Data 
```
Then you just need to copy the code from fsx file to Program.fs.
Here is the final result:

```fsharp
open System.IO
open FSharp.Data
open CoCoTender.Domain
open BoQItem
open Project

[<Literal>]
// let ResolutionFolder = @"c:/projs/CoCoTender/src/scripts/"// __SOURCE_DIRECTORY__ 
let ResolutionFolder =  __SOURCE_DIRECTORY__ 
module BoQFile =
    type BoQItems = CsvProvider<"boq.csv", ResolutionFolder=ResolutionFolder>
    let toBoQItem (row:BoQItems.Row) =
        let qty = Quantity (row.Quantity, row.Unit.Trim())
        let material = 
            {
                Name = row.Material
                Unit = row.MaterialUnit.Trim()
                UnitCost = row.MaterialUnitCost
            }
            |> Material

        let labor = 
            {
            Name = row.Labor
            Unit = row.LaborUnit.Trim()
            UnitCost = row.LaborUnitCost
            }
            |> Labor

        BoQItem.tryCreate row.Description qty material labor

    let load (filePath:string) =
        BoQItems
            .Load(filePath)
            .Rows
        |> Seq.choose toBoQItem 
        |> List.ofSeq

module FactorFFile =
    type FactorFs = CsvProvider<"FactorF.csv", ResolutionFolder=ResolutionFolder>

    let load (filePath:string) () = 
        FactorFs
            .Load(filePath)
            .Rows
        |> Seq.map (fun x -> x.DirectCost,x.FactorF) 
        |> List.ofSeq 
        |> FactorFTable

[<EntryPoint>]
let main args =
    printfn "%A" args
    match args |> List.ofArray with 
    | boqFile::factorFFile::_ -> 
        let cost = estimateCost (FactorFFile.load factorFFile) (BoQFile.load boqFile) 
        printfn $"Estimate Cost is {cost}"
    | [] -> 
        let boqFile = "boq.csv"
        let factorFFile =  "factorf.csv"
        printfn "%s" boqFile
        printfn "%s" factorFFile
        let cost = estimateCost (FactorFFile.load factorFFile) (BoQFile.load boqFile) 
        printfn $"Estimate Cost is {cost}"
    | _ -> 
        printfn "Should not be here"

    0
```

Let's try running the console:

```bash
dotnet run boq.csv factorf.csv
```

I show the application to the domain expert and let their play with the program. 

**Expert**: Can I create boq item with invalid data? Such as having item with different unit?
**Me**: Sure, if you do that those line will be skipped. 
**Expert**: Hmmm, I don't think that's right. User should immediately see the problem. 

You might felt bad about not creating the right solution for the client in the first place. But don't worry, this way of showing immediate feed back to the client (aka. REPL on steroid) is so much better for both developer and the users of the system. Since, all the misunderstanding could be cleared out in days; not weeks, or even months.

**Me**: We can fix that. Should I show the error for that line in particular? 
**Expert**: We would like to see the whole error. It should show all the error for each line. So the user can fix all of them before re-estimate the data.

What we need to do is to keep error message for each items if any and show them. If there is no error, we just return the estimated code. This is where FsErrorToolkit comes to play.

FsErrorToolKit
==============

I really like railway-oriented programming style. I think it's straightforward and can blend well with function programming style. Recently, F# has already include Result type into the core language. But I heard many good things from library FsErrorToolkit. So I will give it a shot for this project.

Before using the toolking in the Domain library, it's often a good idea to pivot any library with fsx script first.
Here is my plan of introducing new code to fsharp solution.
1. Try out in fsx script
2. Change the actual code
3. Change unit testing 

Start by copying the code from Domain to fsx file and change the most outer function (estimateCost) to use Result type and gradually change from there.

```fsharp
  let estimateCost' loadFactorFTable = calcDirectCost >> (applyFactorF loadFactorFTable) >> roundCost 

  let estimateCost loadFactorFTable items = 
    let cost = estimateCost' loadFactorFTable items
    Ok cost
```

Since this function has no error, so we just return cost with Ok constructor. 
How about the one that can cause error like creating boq item. Previously, if we could not create item, we just return None. But now we want to return Error with helpful message instead.

```fsharp
  let areUnitsMatched  qty materialUnitCost laborUnitCost =
    match qty, materialUnitCost, laborUnitCost with
    | Quantity (_, qtyUnit), Material {Unit = materialUnit}, Labor {Unit = laborUnit} ->
      qtyUnit = materialUnit && qtyUnit = laborUnit

  let tryRecalcTotalCost item = 
    let calcTotalCost() =
      match item.Quantity, item.MaterialUnitCost, item.LaborUnitCost with
      | Quantity (qty,_), Material {UnitCost = mUnitCost}, Labor {UnitCost = lbUnitCost} ->
        (qty * mUnitCost) + (qty * lbUnitCost)
      
    result 
      {
        do! areUnitsMatched item.Quantity item.MaterialUnitCost item.LaborUnitCost |> Result.requireTrue "Unit are Not matched." 
        return { item with TotalCost = calcTotalCost() }
      }

        
  let tryCreate desc qty materialUnitCost laborUnitCost = result {
    
    do! areUnitsMatched qty materialUnitCost laborUnitCost |> Result.requireTrue "Unit are Not matched." 

    let! item =  
      {
        
        Description = desc
        Quantity = qty
        MaterialUnitCost = materialUnitCost
        LaborUnitCost = laborUnitCost
        TotalCost = 0.0
      } 
      |> tryRecalcTotalCost

    return item      
  }
```

This requires some explanations. Function try create is inside result computation expression which means that when any error occurs, execution will return from the function immediately. This way of coding significantly remove boiler plat code. The intention is much clearer but at the same time, it comes with cause of understanding what CE (computation expression) is. But I think it's worthwhile.

Here we go for tryCreate:
1. Check if areUnitsMatched function which return boolean. If the validation function return false, return from CE immediately with error message "Unit are Not matched".
2. Recalculate the total cost. See that we use let! when calling function returning result type (tryRecalcTotalCost in this case)

After checking that the new script with result CE works as expected, just copy the code to Library.fs file.
Don't forget to add the toolkit to the project.

```bash
cd src\CoCoTender.Domain
dotnet add package FsErrorToolKit.ErrorHandling
```

We need to change the Test project to use Result type as well. Here is an example for creating boq item test:

```fsharp
    [<Fact>]
    let ``should succeed if Quantity, Material and Labor use the same Unit`` () = 
        let result = BoQItem.create desc qty material labor
        match result with
        | Ok item -> 
            let itemVal = item |> BoQItem.value
            itemVal.Description |> should equal desc
            itemVal.Quantity |> should equal qty
            itemVal.MaterialUnitCost |> should equal material
            itemVal.LaborUnitCost |> should equal labor
        | Error msg -> Assert.Fail msg
```
We just need to match the result with Ok and Error.
Finally, we are almost at the end of this post. The only thing left is to update the Console application to use Result type.
The only function that require understanding is the load function for BoQ item.

```fsharp
    let load (filePath:string) =
        BoQItems
            .Load(filePath)
            .Rows
        |> Seq.map toBoQItem 
        |> List.ofSeq
        |> List.sequenceResultA
        |> function
            | Ok items -> items
            | Error msgs -> failwith $"{msgs}"
```
Remember that the expert wants the error for each boq item (if any). After we map toBoQItem for each row, we will get sequence of Result type. But what we really want is just one Result with **a list** of Ok or Error instead. For example, if there are 2 error after load boq item. We want to have result as Error [ "error msg1"; "error msg2" ]. 
All of this can be accomplished by **List.sequenceResultA** provided by the toolkit.  
If there are any error, the program will fail and show list of error messages.

It's all done for this post now. You can get the code for this post by clone with tag v0.2.0.
Till next time!