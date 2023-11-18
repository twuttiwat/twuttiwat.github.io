---
title: "Domain Modeling With F#"
date: 2022-10-26T09:28:16+07:00
draft: false
author: Teerawat Wuttiwat
---
Welcome back. Today, I am going to explain the project we are going to created, discuss all the requirements with imaginary domain experts, list all the use cases, and (finally) model the domain in F# code.

Construction Cost Estimation Project
-----------------------------------

All the construction in the world, be it buildings, dams, roads or bridges. How small or large are they. All of them, starts with architecture blueprints. 

No matter how beatutiful the blueprints are, the only thing matters are how much the budget is needed for that construction.
This is very important since most of the owner of the building do not build the building themselves. Instead, they need to announce the project for the contractors. Each contractor need to bid in order to win the project.

Hence, the project owner needs to know the budget estimation of the project. So that they will know if the bidding from the contractor make sense or not.
The larger the project, the more important the estimation of the project is. 

1. Prevent pricing scheme between contractors. in which, all contractos have a deal to propose very high estimation.

2. Check the feasibility of the project. in which, the project owner will need to check if the cost involved of the project is worthwhile or not. Usually, the owner will have their limit of budget, they might need to change blueprints in order to decrease the cost to be within the project.

3. Prevent wrong doing of estimator of project owner. The estimation if done correctly will be standardized and transparent such that there are very difficult for the estimator to have bias over specific material vendor.

Getting Requirements
--------------------

Without further ado, let's start getting into the project by discussin with domain experts. To make it fun, I will make it like conversation between the experts and me.
Here we go.

I usually like to start with how are the domain experts (DE) working at the moment. Usually, most of them will have Excel or paper based in placed.

**Me**: Hello. Could you please explain how are you working at the moment?  
**Expert**: Sure, we are working in Construction Project Estimate Department. Usually, we will get input as **blueprint** and **spec book** from architureture department. We then **decode** the blueprint into **Bill Of Quantity (BoQ)** which usually is an Excel file. We sum up the **cost for each BoQ Item** to get the **direct cost** of the project. We then look up for **Factor F** using the direct cost. Finally, we multiply the Factor with the direct cost to get the **estimated cost of the prject**.

Phew, I think to myself that quite a lot of domain jargon which I have no idea. I highlight each jargon to clarify each of them in detail.

**Me**: There are many words that I do not understand yet. Could you please clarify one by one. Let's start with **blueprint**.
**Expert**: Ok, the **blueprint** is what the architecture draw construction in detail to give you construction sub-contractor to build. We also use the same blueprint to calculate or in our term, decoding, to Bill of Quantity (BoQ).
**Me**: Could you show me the example of **Bill of Quantity (BoQ)**. Let's keep it short by referring it as BoQ then.
Expert: Yes, we usually refer that way. Here is what it looks like. **BoQ** of the project is the list of work that need to be done to finish the project. The work is itemized as **BoQ Item**. Each item will contain **materials**, **labors**, and their **cost**. Here is how it looks like:


| No. | Work Description | Quantity | Unit | Material | Material Unit Cost | Labor     | Labor Unit Cost | Item Cost |
|-----|------------------|----------|------|----------|--------------------|-----------|-----------------|------------|
|   1 |  Pool Tiling     |    10    | m^2  | Big Tile |       100          | Do Tiling |       50        |     1500   |



I remind myself that this looks like *Data* which can be defined using F# *Record*
Anyway, there are more to that, we should confirm about calculation of total cost:

**Me**: Can I assume that the **Item Cost** is the summation of Material Cost and Labor Cost.    

**Expert**: Yes, it is just like that. To get **Material Cost**, you just need to multiply **Quantity** with **Material Unit Cost**. Same goes with Labor Cost. In the case of Pool Tiling above, the Material Cost = 10 * 100 = 1000 and the Labor Cost = 10 * 50 = 500. Hence, the **Item Cost** is 1000 + 500 = 1500.  

**Me**: How about the **Unit**? Do we have to specify **Unit** for Material and Labor?  

**Expert**: Not needed at the moment. You just need to make sure that both Unit Cost must use the same **Unit** per money (Thai Baht in this case). For example, the Material Unit Cost for Pool Tiling is 100 m^2 / baht and Labor Unit Cost is 50 m^2 / bath. It is very important though that conversion is needed to make sure that **Unit** is correct.  

Here is our first formula - **Item Cost Calculation**

    ItemCost = Quantity * MaterialUnitCost + Quantity * LaborUnitCost

I think we are ready to create our first function now. Open up your VSCode, create new fsi, names it scratch.fsx and write down following snippet:

```fsharp
let calcItemCost qty materialUnitCost laborUnitCost = 
  qty * materialUnitCost + qty * laborUnitCost
```

This is very naive sketch for this function. We can do better. Let think about what errors can occur in this situation.
1. User is confused between quantity and unit cost since both are float.
2. User is confused between material and labor unit cost.
3. User is confused between unit of quantity and unit cost since none of them are provided.

We will improve the code iteratively. Start with the first item.

1. User is confused between quantity and unit cost since both are float.

We can enforce this by introducing single discriminated union type to Quantity and UnitCost and requires the types of input of calcItemCost function 

```fsharp
type Quantity = Quantity of float
type UnitCost = UnitCost of float

let calcItemCost (Quantity qty) (UnitCost materialUnitCost) (UnitCost laborUnitCost) =  
  qty * materialUnitCost + qty * laborUnitCost
```

2. User is confused between material and labor unit cost.

Again, we can use type to enforce this by introducing MaterialUnitCost and LaborUnitCost types. 

```fsharp
type MaterialUnitCost = Material of UnitCost
type LaborUnitCost = Labor of UnitCost

let calcItemCost quantity materialUnitCost laborUnitCost =
  match quantity, materialUnitCost, laborUnitCost with
  | (Quantity qty), (Material (UnitCost materialUnitCost')), (Labor (UnitCost laborUnitCost')) ->
    qty * materialUnitCost' + qty * laborUnitCost'
```
As you can see, I decide to use match to extract actual values of material and labor unitcost.

3. User is confused between unit of quantity and unit cost since none of them are provided.

We just need to introduce Unit for both Quantity and Material and make sure that when we calculate the item cost, it will check if both of them are the same. Here is the code.

```fsharp
type Quantity = Quantity of float*string
type UnitCost = 
  {
    Name: string
    UnitCost: float
    Unit: string
  }
type MaterialUnitCost = Material of UnitCost
type LaborUnitCost = Labor of UnitCost

let tryCalcItemCost (Quantity (qty,qtyUnit)) (Material materialUnitCost) (Labor laborUnitCost) =  
  if qtyUnit = materialUnitCost.Unit && qtyUnit = laborUnitCost.Unit then 
    (qty * materialUnitCost.UnitCost + qty * laborUnitCost.UnitCost) |> Some
  else
    None

let testCalcItemCost = 
  let materialUnitCost = Material { Name = "Big Tile"; UnitCost = 100; Unit = "m2" }
  let laborUnitCost = Labor { Name = "Do Tiling"; UnitCost = 50; Unit = "m2" }
  tryCalcItemCost (Quantity (10, "m2")) materialUnitCost laborUnitCost = Some 1500 
```

I change the name of calcItemCost to tryCalcItemCost since we are going to return Option of float as result. When the input of calculation invalid (i.e. unit is not equal), we will return None.

Usually, when we start to have some non-trivial function, I will create test function in the fsx file. F# protect us for most error, but sometimes Unit Testing is necessary.



```fsharp
type BoQItem =
  {
    OrderNo: int
    Description: string
    Quantity: float
    Unit: string
    Material: string
    MaterialUnitCost: float
    Labor: string
    LaborUnitCost: float        
  }

let calcItemCost item = 
  (item.Quantity * item.MaterialUnitCost) + (item.Quantity * item.LaborUnitCost)  

/// Test 
/// 
let poolItem = 
  { 
    OrderNo = 1
    Description = "Pool Tiling"
    Quantity = 10
    Unit = "m^2"
    Material = "Big Tile"
    MaterialUnitCost = 100
    Labor = "Do Tiling"
    LaborUnitCost = 50
  }

let testCalcItemCost = (calcItemCost poolItem) = 1500
```

The function is done. We will go to next task, implementing **BoQItem**. From the way BoQItem is represented, I think it is more appropriated to implement in Record type.

```fsharp
type BoQItem =
  {
    Description : string
    Quantity : Quantity
    MaterialUnitCost : MaterialUnitCost
    LaborUnitCost : LaborUnitCost
    TotalCost : float
  }
```

The new TotalCost field should always include up-to-date **Item Cost**. To enforce this, we need to make sure that whenever new BoQItem is created, TotalCost will be calculated and update. This can be done by make the type private to module. Here is the code:

```fsharp
type BoQItem = private {
    //...Same as above
  }

module BoQItem =
  let tryCalcItemCost (Quantity (qty,qtyUnit)) (Material materialUnitCost) (Labor laborUnitCost) =  
    if qtyUnit = materialUnitCost.Unit && qtyUnit = laborUnitCost.Unit then 
      (qty * materialUnitCost.UnitCost + qty * laborUnitCost.UnitCost) |> Some
    else
      None

  let tryCreate desc qty materialUnitCost laborUnitCost =
    tryCalcItemCost qty materialUnitCost laborUnitCost
    |> Option.map (fun cost -> 
      {
        Description = desc
        Quantity = qty
        MaterialUnitCost = materialUnitCost
        LaborUnitCost = laborUnitCost
        TotalCost = cost
      }
    ) 
```

Users will want to update BoQItem, we should create CRUD operation on the BoQItem type.

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
```
Here, we create update function for Description, Quantity, and MaterialUnitCost. Since both Quantity and MaterialUnitCost can have some invalid result (i.e. Units are not matched.), both will return Option type.
Let's create functions for testing this CRUD operations.

```fsharp
let poolItem =
  let qty = Quantity (10, "m^2")
  let materialUnitCost = Material { Name = "Big Tile"; UnitCost = 100; Unit = "m^2" }
  let laborUnitCost = Labor { Name = "Do Tiling"; UnitCost = 50; Unit = "m^2" }
  tryCreate "Pool Tile"  qty materialUnitCost laborUnitCost
 
let testUpdateDesc = 
  poolItem 
  |> Option.map (updateDesc "New Pool") 
  |> Option.map (fun x -> x.Description = "New Pool") 
  |> Option.defaultValue false

let testUpdateQty = 
  let newQty = Quantity (20,"m^2")
  poolItem 
  |> Option.bind (tryUpdateQty  newQty) 
  |> Option.map (fun x -> x.TotalCost = 3000) 
  |> Option.defaultValue false

let testUpdateMaterialUnitCost = 
  let newMaterialUnitCost = 
    Material <| { Name = "Small Tile"; UnitCost = 50; Unit = "m^2"}
  poolItem 
  |> Option.bind (tryUpdateMaterialUnitCost  newMaterialUnitCost) 
  |> Option.map (fun x -> x.TotalCost = 1000) 
  |> Option.defaultValue false
```

I found that combining lots of Option related functions together makes the code hard to read. I decide to use **maybe** computation expression to make the code more succint.

```fsharp 
type MaybeBuilder() =

  member this.Bind(x, f) =
    match x with
    | None -> None
    | Some a -> f a

  member this.Return(x) =
    Some x

let maybe = new MaybeBuilder()

let testUpdateDesc = 
  maybe {
    let! poolItem' = poolItem
    let updatedPoolItem = updateDesc "New Pool" poolItem'
    return updatedPoolItem.Description = "New Pool"
  }
  |> Option.defaultValue false

let testUpdateQty = 
  maybe {
    let! poolItem' = poolItem
    let newQty = Quantity (20,"m^2")
    let! updatedPoolItem = tryUpdateQty newQty poolItem'
    return updatedPoolItem.TotalCost = 3000
  }
  |> Option.defaultValue false

let testUpdateMaterialUnitCost = 
  maybe {
    let! poolItem' = poolItem
    let newMaterialUnitCost = Material <| { Name = "Small Tile"; UnitCost = 50; Unit = "m^2"}
    let! updatedPoolItem = tryUpdateMaterialUnitCost newMaterialUnitCost poolItem'
    return updatedPoolItem.TotalCost = 1000
  }
  |> Option.defaultValue false
```
The number of lines might be almost the same but the meaning of codes are clearer.

I think we are halfway through the domain modeling now. I will go sip some coffee before continuing. Also, it's a good idea to show the result of test especially the calculation one to the client. 

Ok, we finish the coffee. The next thing is to calculate the **Estimate Cost** of the construction project. 
We start by showing the calculation to the expert to confirm our understanding:

    EstimateCost = DirectCost * FactorF
    
The main calculation is very simple. We need to go from there one bye one.

    DirectCost = Summation of BoQItem Cost

This one is easy. We can write the code as follow.

```fsharp
type DirectCost = DirectCost of float

let calcDirectCost items = 
  items 
  |> List.sumBy (fun a -> a |> value |> (fun b -> b.TotalCost))
  |> DirectCost
```
Note that we create new simple type for DirectCost to make it different from normal float type.

Next is **FactorF**, we do not know how to get this. It's time to talk with the Expert.

**Me**: We know that **FactorF** is important in estimating the cost but we have no idea how to get it. Could you please shed some light?

**Expert**: Easy. There is a table lookup for **FactorF** provided by the government. All you need to do is **look up** using **Direct Cost** of your project.

**Me**:  Can you show me how to lookup? 
**Expert**: Sure. Here is a sample of **FactorF Table**. 

| Direct Cost     | Factor F |
|-----------------|----------|
| < 10            | 1.1      |
| 10 <= x <= 100  | 1.5      |
| > 100           | 1.9      |

If your project direct cost is less than 10 then the Factor F will be 1.1 but if is greater than 100 then it will be 1.9.
Things will get more complicated when the cost is between 10 and 100. We call this as in **Bound**. You need to get proportion of your direct cost and multiply that to range of factor f. It's mouthful. Let's show the expert the formula.

    BoundFactorF = DirectCostRatio * FactorFRange + LowerBoundF
    DirectCostRatio = (DirectCost - LowerBoundCost) /(UpperBoundCost - LowerBoundCost) 
    FactorFRange = UpperBoundF - LowerBoundF

Let's try out the formular for DirectCost of 55.

    FactorFRange = 1.5 - 1.1 = 0.4
    DirectCostRatio = (55 - 10) / (100 - 10 ) = 0.5
    BoundFactorF = 0.5 * 0.4 + 1.1 = 1.3

Let's convert this formula to f#

```fsharp
let calcBoundF lowerBound upperBound directCost =
  let lowerBoundCost,lowerBoundF = lowerBound
  let upperBoundCost,upperBoundF = upperBound
  let fRange = upperBoundF - lowerBoundF
  let costRatio = (directCost - lowerBoundCost) / (upperBoundCost - lowerBoundCost)
  costRatio * fRange + lowerBoundF

let testCalcBoundF = (=) (calcBoundF (10.0,1.1) (100.0,1.5) 55.0) 1.3 
```

Next, we will take care of DirectCost which exceed either minimum or maximum bound. The FactorF table is simply defined as a list of two float tuple. The firt is cost and the second is factor f. For example, the example table can de defined as follow:

```fsharp
let exampleFTable = 
  ( (10.0, 1.1); (100.0, 1.5); (1000, 1.9) )
  |> FactorFTable
```

```fsharp
type FactorFTable = FactorFTable of float*float list
let calcFactorF (FactorFTable fTable) (DirectCost directCost) = 
  let findBound() = 
    let lowerBoundIndex = 
      fTable 
      |> List.findIndexBack (fun (cost,_) -> directCost > cost)
    let upperBoundIndex = lowerBoundIndex + 1
    (fTable |> List.item lowerBoundIndex), (fTable |> List.item upperBoundIndex)

  let (minCost, minF), (maxCost, maxF) = (fTable |> List.head), (fTable |> List.last)
  
  if directCost < minCost then minF
  else if directCost > maxCost then maxF
  else 
    findBound()
    |> calcBoundF
```

We have all the ingredients, let combine everything to our main function **etimateCost**.

```fsharp
let applyFactorF loadFactorFTable (DirectCost directCost) =
  let factorF = calcFactorF loadFactorFTable()
  directCost * factorF

let roundCost = function
  | a when a < 1000.0 -> a 
  | b ->
    b / 1000.0
    |> System.Math.Truncate
    |> (*) 1000.0

let estimateCost loadFactorFTable =
  calcDirectCost >> (applyFactorF loadFactorFTable) >> roundCost
```

We have introduced 3 functions here. Function applyFactorF is like the name apply FactorF to DirectCost. Function loadFactorFTable is an input to the function so we can test the applyFactorF without really using any persistence layer. For example here is the example function

```fsharp
let testFactorFTable () = 
  ( (10.0, 1.1); (100.0, 1.5); (1000, 1.9) )
  |> FactorFTable
```

The roundCost function is just function to strip the number after thousand digit. The last function estimate cost use combination to combine all small functions together.

We have finish ourt first use case

    1. Estimate construction cost

All the code can be clone from https://github.com/twuttiwat/CoCoTender using tag v0.1.1 

Or you can use following command

```bash
git clone -b v0.1.1 https://github.com/twuttiwat/CoCoTender.git
```

Next, we will growing this use case from this simple fsharp script to full-blown webapp. Stay Tune!

