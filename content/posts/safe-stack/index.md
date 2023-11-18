---
title: "From Domain to Web with Safe Stack"
date: 2022-12-05T09:21:13+07:00
draft: false
author: Teerawat Wuttiwat
---

Hi there. Welcome back to my F# web app development series. Just to recap for new people, we are going to create construction cost tender estimation (CoCoTender) using f#. If you are interested in the domain, you can read the [past](../domain-modeling-with-fs/) [posts](../unit-testing-console) in the series.
For this post, we wil emphasize how to connect the domain to web app using safe stack. So there is no need to understand the whole domain concept.

Let's kick off the process by cloning git repository I prepare for this post.

```bash
git clone --branch begin-safe-stack https://github.com/twuttiwat/CoCoTender.git
cd CoCoTender
dotnet run
```

Opening web browser at http://localhost:8080, you should see following default Todo app.

{{<figure src="images/default-todo-app.jpg" width="600"
          alt="Default Todo App" title="Default Todo App">}}

If you see this, then we are good to go now.

Use Cases
=========

One way to model the api is to list all of uses cases and implement each api function one by one. Here is the list of our use cases:

1. List BoQ Items
2. Add New BoQ Item
3. Update BoQ Item
4. Delete BoQ Item
5. Calculate All Cost
6. Show FactorF Table

There are previous posts mention domain modeling in the past. But for short, the ultimate goal for this CoCoTender app is to calculate the  tender cost of the project. The cost will depends on list of construction task (i.e. BoQItem). Our application should provide user interface to do CRUD operation on list of BoQItems and let the app calculate the cost and present that to client.

Let's start with our first use case

List BoQ Items
---------------

One advantage of working with **Safe Stack** is the ablility to share code between server and client. The **Shared** module will include interface of the Api and Dto (Data Transfer Object).

I like to start on Shared module then Server and finally at the Client. I found that checking error on Server is much simpler than on the client.

Go to the bottom of the Shared.fs and declare following Dto and Api for our app.

```fsharp
type BoQItemDto =
    {
        Id : Guid
        Description: string
        Quantity: float
        Unit: string
        Material: string
        MaterialUnitCost: float
        Labor: string
        LaborUnitCost: float
        TotalCost: float
    }

module BoQItemDto =
    let create itemId description quantity unit material materialUnitCost labor laborUnitCost totalCost =
        {
            Id = itemId
            Description = description
            Quantity = quantity
            Unit = unit
            Material = material
            MaterialUnitCost = materialUnitCost
            Labor = labor
            LaborUnitCost = laborUnitCost
            TotalCost = totalCost
        }

type ICoCoTenderApi =
    { getBoQItems: unit -> Async<Result<BoQItemDto list,string>> }
```

Next, go to Server.fs. We will implement the getBoQItems function.
As you can see from the signature of getBoQItems function, we will send back AsyncResult as output of the function. I think it's a good idea to rely on the Result, so we can be explicit about the error. 

To make it easier to work with AsyncResult type, it's better to use FsErrorToolkit.ErrorHandling computation expression. I will remove a lot of matching expression when we try to extract the result from Result type.

Press ctrl-` and type following command

```bash
dotnet add package FsToolkit.ErrorHandling
```

We also need to use function from our Domain library as well. So add reference to that library.

```bash
dotnet add reference ../CoCoTenderDomain
```

The default Todo app use ResizeArray as a way to store data. We will use the same approach at the moment and will change it that to some kind of database in the future.

add following line in module Storage

```fsharp
module Storage =
    let boqItems = ResizeArray<BoQItem>()

    let addBoQItem (boqItem: BoQItem) =
        boqItems.Add boqItem
        Ok ()

    do
        let qty = Quantity (10.0, "m^2")
        let material = Material { Name = "Pool Tile"; Unit = "m^2"; UnitCost = 100.0 }
        let labor = Labor { Name = "Do Tiling"; Unit = "m^2"; UnitCost = 50.0 }
        let defaultItem = BoQItem.tryCreate (Guid.NewGuid()) "Pool Tile" qty material labor
        match defaultItem with
        | Ok defaultItem' ->
            addBoQItem defaultItem' |> ignore
        | _ -> ()
```
**Note** there are some issues with Library.fs in CoCoTender.Domain project. You can download the file and replace it in your project. 


We need function to convert between Dto and Domain type. I tried to create Dto in Shared module but could not make it worked. So I decide to create the Dto conversion module in Server project instead. 
Just paste following code before the webApp line

```fsharp
module Dto =
    let toBoQItemDto (domain:BoQItem) =
        let domainVal =  domain |> BoQItem.value
        let (Quantity (qty, qtyUnit)) = domainVal.Quantity
        let (Material material) = domainVal.MaterialUnitCost
        let (Labor labor) = domainVal.LaborUnitCost
        BoQItemDto.create domainVal.Id domainVal.Description qty qtyUnit
                            material.Name material.UnitCost labor.Name labor.UnitCost
                            domainVal.TotalCost

    let toBoQItemDomain (dto:BoQItemDto) =
        let qty = Quantity (dto.Quantity, dto.Unit)
        let material = Material { Name = dto.Material; Unit = dto.Unit; UnitCost = dto.MaterialUnitCost }
        let labor = Labor { Name = dto.Labor; Unit = dto.Unit; UnitCost = dto.LaborUnitCost }
        BoQItem.tryCreate dto.Id dto.Description qty material labor
```

Finally, declare cocoTenderApi before webApp line

```fsharp
let cocoTenderApi =
    {
        getBoQItems = fun () -> asyncResult {
            return Storage.boqItems |> List.ofSeq |> List.map Dto.toBoQItemDto
        }
    }
```
and change webApp to use this new cocoTenderApi

```fsharp
let webApp =
    Remoting.createApi ()
    |> Remoting.withRouteBuilder Route.builder
    |> Remoting.fromValue cocoTenderApi
    |> Remoting.buildHttpHandler
```

You can test if the server api work correctly by calling api directly in Chrome. Change the url to http://localhost:8080/api/ICoCoTenderApi/getBoQItems. The result should be as follow:


{{<figure src="images/get-boq-items-response.jpg" width="600"
          alt="Response from GetBoQItems Api" title="GetBoQItem Response">}}

We have finished the Server part. Next task is to render our BoQItems on the client side. Since user like to use Excel style interface, I decide to use Feliz.AgGrid component for this. We need to install it first. Go to Client project folder and install using femto.

```bash
femto install Feliz.AgGrid
```

Open Index.fs, and start by modifying our Model type. Safe stack use Elmish for client-side programming. The only single source of state is from Model, the only place that can change the Model is from update function. The view will render the output using the states in the Model as an input.

Back to our Model, we need to maintain list of BoQItems to render it on the AgGrid. So define the model as follow:

```fsharp
type Model = { BoQItems: BoQItemDto list } 
```

There will be 2 types of Msg to update the Model: GetBoQItems and GotBoQItems. The first one is for requesting list of BoQ Items and the second one is for receiving the list.

```fsharp
type Msg =
    | GetBoQItems
    | GotBoQItems of Result<BoQItemDto list, string>
```

Replace todosApi with our cocoTenderApi 

```fsharp
let cocoTenderApi =
    Remoting.createApi ()
    |> Remoting.withRouteBuilder Route.builder
    |> Remoting.buildProxy<ICoCoTenderApi>
```

Initialize the Model in init fuction such that the item list will be empty at first, then the GetBoQItems will be dispatch to request list of items from the Server.

```fsharp
let init () : Model * Cmd<Msg> =
    let model = { BoQItems = [] }

    let cmd = Cmd.ofMsg GetBoQItems

    model, cmd
```

Next implement the update function to get and receive boq items.

```fsharp
let update (msg: Msg) (model: Model) : Model * Cmd<Msg> =
    match msg with
    | GetBoQItems ->
        let cmd = Cmd.OfAsync.perform cocoTenderApi.getBoQItems () GotBoQItems
        model, cmd
    | GotBoQItems (Ok items) ->
        { model with BoQItems = items}, Cmd.none
    | GotBoQItems (Error msg) -> failwith msg
```
Since GotBoQItems response will be of type Result, then it's good idea to match the result at the first level. 


Now the fun part begins. We will show the list of items using Feliz.AgGrid. I love to see the output as soon as possible, having the Hot Module Reload in Safe stack really helps in this. 

First replace function containerBox with the new one with AgGrid.

```fsharp
let containerBox (model: Model) (dispatch: Msg -> unit) =
    Html.div [
        prop.className ThemeClass.Balham
        prop.children [
            AgGrid.grid [
                AgGrid.rowData (model.BoQItems |> Array.ofList)
                AgGrid.defaultColDef [
                    ColumnDef.resizable true
                ]
                AgGrid.domLayout AutoHeight
                AgGrid.onGridReady (fun x -> x.AutoSizeAllColumns())
                AgGrid.singleClickEdit true
                AgGrid.enableCellTextSelection true
                AgGrid.ensureDomOrder true
                AgGrid.columnDefs [
                    ColumnDef.create<string> [
                        ColumnDef.headerName "Description"
                        ColumnDef.valueGetter (fun x -> x.Description)
                    ]
                    ColumnDef.create<float> [
                        ColumnDef.headerName "Quantity"
                        ColumnDef.valueGetter (fun x -> x.Quantity)
                        ColumnDef.width 75
                    ]
                    ColumnDef.create<string> [
                        ColumnDef.headerName "Unit"
                        ColumnDef.valueGetter (fun x -> x.Unit)
                        ColumnDef.width 50
                    ]
                    ColumnDef.create<string> [
                        ColumnDef.headerName "Material"
                        ColumnDef.valueGetter (fun x -> x.Material)
                        ColumnDef.width 150
                    ]
                    ColumnDef.create<float> [
                        ColumnDef.headerName "M. UnitCost"
                        ColumnDef.valueGetter (fun x -> x.MaterialUnitCost)
                        ColumnDef.width 75
                    ]
                    ColumnDef.create<string> [
                        ColumnDef.headerName "Labor"
                        ColumnDef.valueGetter (fun x -> x.Labor)
                        ColumnDef.width 150
                    ]
                    ColumnDef.create<float> [
                        ColumnDef.headerName "L. UnitCost"
                        ColumnDef.valueGetter (fun x -> x.LaborUnitCost)
                        ColumnDef.width 75
                    ]
                    ColumnDef.create<float> [
                        ColumnDef.headerName "Total Cost"
                        ColumnDef.valueGetter (fun x -> x.TotalCost)
                        ColumnDef.width 75
                    ]
                ]
            ]
        ]
    ]
```

The grid width will be too small. We will remove Bulma.column from Bulma.heroBody to let the container box expand to full width. 

```fsharp
            Bulma.heroBody [
                Bulma.container [
                    Bulma.title [
                        text.hasTextCentered
                        prop.text "CoCoTender"
                    ]
                    containerBox model dispatch
                ]
            ]
```

Here is the result of listing boq items:


{{<figure src="images/list-boq-items.jpg" width="600"
          alt="List BoQItems" title="List BoQItems">}}

Add New BoQ Item
----------------         

That's quite a lot of steps we do for the first use case. For the second one, the steps will almost be the same. 

First we add new function to the Api in Shared.fs

```fsharp
    addBoQItem: BoQItemDto -> Async<Result<BoQItemDto,string>>
```

Second, we implement the function in Server.fs
```fsharp
        addBoQItem =
            fun boqItemDto -> asyncResult {
                let! boqItem =  boqItemDto |> Dto.toBoQItemDomain
                do! Storage.addBoQItem boqItem
                let boqItemDto' = boqItem |> Dto.toBoQItemDto

                return boqItemDto'
            }
```

Third, we implement Add functionality in the Client.

Add 2 more discriminated union for Msg

```fsharp
    | AddBoQItem
    | AddedBoQItem of Result<BoQItemDto,string>        
```

Add 3 more cases for update function

```fsharp
    | AddBoQItem ->
        let cmd = Cmd.OfAsync.perform cocoTenderApi.addBoQItem (defaultNewItem()) AddedBoQItem
        model, cmd
    | AddedBoQItem (Ok boqItem) ->
            { model with BoQItems = model.BoQItems @ [ boqItem ] }, Cmd.none
    | AddedBoQItem (Error msg) -> failwith msg
```

See that we have new function defaultNewItem to create default new item. Just define the function before the update function

```fsharp
let defaultNewItem () =
    BoQItemDto.create (Guid.NewGuid()) "New Item" 0.0 "m" "Material 1"
        0.0 "Labor 1" 0.0 0.0
```

Add button before Ag.Grid
```fsharp
            Bulma.button.button [
                color.isPrimary
                prop.onClick (fun _ -> dispatch AddBoQItem)
                prop.text "Add"
            ]
            AgGrid [
                ...
            ]
```

Here is the result after clicking the Add button. The new boq item will be create locally, send to the server, store there and send back to client.

{{<figure src="images/add-boq-item.jpg" width="600"
          alt="Add BoQItem" title="Add BoQItem">}}

Update and Delete BoQ Item
--------------------------
I think we are ready to do 2 use cases at once now. 
Go to Shared.fs and add update and delete function.

```fsharp
    updateBoQItem: BoQItemDto -> Async<Result<BoQItemDto,string>>
    deleteBoQItem: Guid -> Async<Result<unit,string>>   
```

Implement these 2 functions in Server.fs. First, add 2 more functions to update and delete BoQItem from our Storage.

```fsharp
    let updateBoQItem boqItem =
        let index = boqItems.FindIndex( fun x -> x = boqItem)
        boqItems[index] <- boqItem
        Ok ()

    let deleteBoQItem itemId =
        let index = boqItems.FindIndex( fun x -> x |> BoQItem.value |> fun y -> y.Id = itemId)
        boqItems.RemoveAt(index)
        Ok ()
```

Next, implement Api function:

```fsharp
        updateBoQItem =
            fun boqItemDto -> asyncResult {
                let! boqItem =  boqItemDto |> Dto.toBoQItemDomain
                let! boqItem =  boqItem |> BoQItem.tryUpdate
                do! Storage.updateBoQItem boqItem

                let boqItemDto' = boqItem |> Dto.toBoQItemDto

                return boqItemDto'
            }
        deleteBoQItem =
            fun itemId -> asyncResult {
                return! Storage.deleteBoQItem itemId
            }
```

Next, is the View part in the client. We are going to do following things:
1. Let user edit column such as Description and UnitCost.
2. After user edit the column, we will call update Api on the server.
3. There will be delete button at the end of each row. Clicking that button will call delete Api on the server. The client then will load items from the server.

Start off by let the user edit the Description column.

```fsharp
    ColumnDef.create<string> [
        ColumnDef.headerName "Description"
        ColumnDef.valueGetter (fun x -> x.Description)
        ColumnDef.editable (fun _ _ -> true)
    ]
```

Then dispatch the UpdateBoQItem message after the updated Description has been set in the grid. Add following line before editable

```fsharp
        ColumnDef.valueSetter (fun newValue _ row  -> { row with Description = newValue } |> UpdateBoQItem |> dispatch)
```

We don't have UpdateBoQItem yet. We will add this message and its response counterpart in the Msg type.

```fsharp
    | UpdateBoQItem of BoQItemDto
    | UpdatedBoQItem of Result<BoQItemDto,string>
```

and implement these 2 new cases in the update function

```fsharp
    | UpdateBoQItem (boqItem: BoQItemDto) ->
        let cmd = Cmd.OfAsync.perform cocoTenderApi.updateBoQItem boqItem   UpdatedBoQItem
        model, cmd
    | UpdatedBoQItem (Ok updatedItem) ->
        let updatedItems = model.BoQItems |> List.map (fun x -> if x.Id = updatedItem.Id then updatedItem else x)
        { model with BoQItems = updatedItems}, Cmd.none
    | UpdatedBoQItem (Error msg) -> failwith msg
```

Just change all the ColumnDef to be editable and dispatch the Update msg in valueSetter function. If you change the value and tab out, you will see that row is changes. Refresh the browser will show the change value since the data is persist on the Server.

But if you enter blank value in the Description column, nothing will happen. Safe-stack usually will show the error in either in Terminal for Server code or Browser Console for Client Code. Checking the client code. 

{{<figure src="images/no-description-error.jpg" width="600"
          alt="No Description Error" title="No Description Error in Console">}}

This occurs because we do use failWith when Error is captured in UpdatedBoQItem case. We are going to show this error to the user instead using Elmish.Toastr.          

```bash
cd src/Client
femto install Elmish.Toastr
cd ../..
npm install jquery
```

Now create function showError and replace all failWith with the function:

```fsharp
let showError model msg =
    printfn $"Error occurred : {msg}"
    model, Toastr.message msg |> Toastr.error
```

In update function:

```fsharp
    | UpdatedBoQItem (Error msg) -> showError model msg
```

Now when we try to update blank Description, the error should shown in Toast like below:

{{<figure src="images/no-description-error-toastr.jpg" width="600"
          alt="No Description Error Toastr" title="No Description Error with Toastr">}}

Next, add delete button in Grid. Add new ColumnDef at the end:

```fsharp
    ColumnDef.create<Guid> [
        ColumnDef.valueGetter (fun x -> x.Id)
        ColumnDef.cellRendererFramework (fun itemId _ ->
            Html.button [
                prop.text "🗑️"
                prop.onClick (fun _ -> dispatch (DeleteBoQItem itemId))
            ]
        )
    ]
```

Add 2 more Msg for Delete and implement the case in update function as follow:

Msg type

```fsharp
    | DeleteBoQItem of itemId:Guid
```

update function

```fsharp
    | DeleteBoQItem itemId ->
        let cmd = Cmd.OfAsync.perform cocoTenderApi.deleteBoQItem itemId (fun _ -> GetBoQItems)
        model, cmd
```
See that after finish delete, we will dispatch GetBoQItems to reload the items from Server.

Clicking on Delete button should delete and refresh the grid.

Calculate All Cost
------------------

We would like to show user the latest cost of the Project. There are 3 cost that user are interested. They are DirectCost, FactorF, and EstimateCost. The calculation for this costs has already been implemented in the Domain. So all you have to do is just call the domain from the Api and return the result back to client.

Since we want the uesr to see the cost after each update. So we decide to return the AllCost to the user whenever there is changes for BoQItem.

Modify the Api in Shared module to return AllCost as follow:

```fsharp
type AllCost =
    {
        DirectCost : float
        FactorF : float
        EstimateCost : float
    }

type ICoCoTenderApi = {
    getBoQItems: unit -> Async<Result<BoQItemDto list*AllCost,string>>
    addBoQItem: BoQItemDto -> Async<Result<BoQItemDto*AllCost,string>>
    updateBoQItem: BoQItemDto -> Async<Result<BoQItemDto*AllCost,string>>
    deleteBoQItem: Guid -> Async<Result<unit,string>>
}    
```

Go to Server.fs. First, we will implement getAllCost function so that we can reuse in all Api functions. Put the function before definition of cocoTenderApi.

```fsharp
open Project

let getAllCost () = result {
    let! (DirectCost directCost, FactorF factorF, EstimateCost estimateCost) =
        Project.tryGetAllCost Storage.loadFactorFTable (Storage.boqItems |> List.ofSeq)

    return
        {
            DirectCost = directCost
            FactorF = factorF
            EstimateCost = estimateCost
        }
}
```

We need to load factor f from the Storage. Implement following function in the Storage module.

```fsharp
    let loadFactorFTable () =
        FactorFTable [(10,1.1); (100,1.5); (1000, 1.9)]
```

Modify the Api to return allCost as follow:
```fsharp
    getBoQItems = fun () -> asyncResult {
        let boqItems = Storage.boqItems |> List.ofSeq
        let! allCost = getAllCost()

        return (boqItems |> List.map Dto.toBoQItemDto), allCost
    }
    addBoQItem =
        fun boqItemDto -> asyncResult {
            let! boqItem =  boqItemDto |> Dto.toBoQItemDomain
            do! Storage.addBoQItem boqItem
            let boqItemDto' = boqItem |> Dto.toBoQItemDto
            let! allCost = getAllCost()

            return boqItemDto', allCost
        }
    updateBoQItem =
        fun boqItemDto -> asyncResult {
            let! boqItem =  boqItemDto |> Dto.toBoQItemDomain
            let! boqItem =  boqItem |> BoQItem.tryUpdate
            do! Storage.updateBoQItem boqItem

            let boqItemDto' = boqItem |> Dto.toBoQItemDto
            let! allCost = getAllCost()

            return boqItemDto', allCost
        }
```

Go back to Client.fs. We need to maintain AllCost received from the server. So we need to modify the Model as follow:

```fsharp
type Model =
    {
        BoQItems: BoQItemDto list
        AllCost: AllCost
    }
```
 
Change the init to initialize AllCost

```fsharp
    let model =
        {
            BoQItems = []
            AllCost = {DirectCost = 0.0; FactorF = 0.0; EstimateCost = 0.0}
        }
```

Change the Msg to accept AllCost from response from the Server:

```fsharp
    | GotBoQItems of Result<BoQItemDto list*AllCost,string>
    | AddedBoQItem of Result<BoQItemDto*AllCost,string>
    | UpdatedBoQItem of Result<BoQItemDto*AllCost,string>
```

Modify the response cases in update as follow:

```fsharp
    | GotBoQItems (Ok (items,allCost)) ->
        { model with BoQItems = items; AllCost = allCost }, Cmd.none

    | AddedBoQItem (Ok (boqItem, allCost)) ->
            { model with BoQItems = model.BoQItems @ [ boqItem ]; AllCost = allCost }, Cmd.none        

    | UpdatedBoQItem (Ok (updatedItem,allCost)) ->
        let updatedItems = model.BoQItems |> List.map (fun x -> if x.Id = updatedItem.Id then updatedItem else x)
        { model with BoQItems = updatedItems; AllCost = allCost }, Cmd.none
```

We will show code below the Grid. At following Bulma Level under grid inside containerBox function:

```fsharp
            Bulma.level [
                prop.style [
                    style.paddingRight 300
                    style.fontSize 14
                    style.fontWeight.bold
                ]
                color.hasBackgroundLight
                prop.children [
                    Bulma.levelLeft []
                    Bulma.levelRight [
                        Bulma.levelItem (Bulma.label $"Direct Cost")
                        Bulma.levelItem (Bulma.label $"{model.AllCost.DirectCost}")
                        Bulma.levelItem (Bulma.label $"Estimate Cost")
                        Bulma.levelItem (Bulma.label $"{model.AllCost.EstimateCost}")
                    ]
                ]
            ]
```

The all cost should be shown as follow:

{{<figure src="images/show-all-cost.jpg" width="600"
          alt="Show All Cost" title="Show All Cost">}}

Show FactorF Table
------------------

Finally, we are at the last use case for this post. The user should be able to see FactorF table used for the calculation. We have factor f returned from the Server but we need to somehow show the table and highlight which factor f range we are using.

First, add new function in Api shared module to get FactorFInfo

```fsharp
type FactorFInfo = (string*float) list

type ICoCoTenderApi {
    ...
    getFactorFInfo: unit->Async<Result<FactorFInfo,string>>
}
```

Next implement it in the Server.fs.
```fsharp
    getFactorFInfo = fun () -> asyncResult { return Project.getFactorFInfo Storage.loadFactorFTable }
```

For the user interface, we will use Bulma.QuickView.
We need to install it first.

```bash
femto install Feliz.Bulma.QuickView
```

The QuickView requires custom css style. We need to import that when we bundle client-side fsharp to js file. The client pipeline is done through webpack. 

Go to webpack.config.js and add cssEntry in CONFIG object after fsharpEntry.

```js
    cssEntry: './src/Client/style.scss',
```

Next go to entry property in Module.export. We need to bundle css entry when we build the project.

```js
        entry: isProduction ? {
            app: [resolve(CONFIG.fsharpEntry), resolve(CONFIG.cssEntry)]
        } : env.test ? {
                app: resolve(config.fsharpEntry)
            }
            :   {
                    app: resolve(CONFIG.fsharpEntry),
                    style: resolve(CONFIG.cssEntry)
                },
```

Next create the file style.scss inside src/Client folder and put below code to import quickview css

```scss
@import '~bulma-quickview/dist/css/bulma-quickview.min.css';
```

Go to Index.fs and add 2 more item in the Model type
```fsharp
type Model =
    {
        ...
        ShowFactorFView: bool
        FactorFInfo: FactorFInfo
    }
```

The first one is the flag indicating that the FactorF quickview should be shown or not. The second one is just the factor f table we got from the Server.

Two more Msgs are added:

```fsharp
    | ToggleFactorFView
    | GotFactorFInfo of Result<FactorFInfo,string>
```

The first message will be used when we toggle the factorf view to open or close. The second one will be when we receive the table.

Change the init function so that we will send 2 request in the beginning. First request will load the boq items, The second one will load factor f table.

```fsharp
let init () : Model * Cmd<Msg> =
    let model =
        {
            BoQItems = []
            AllCost = {DirectCost = 0.0; FactorF = 0.0; EstimateCost = 0.0}
            ShowFactorFView = false
            FactorFInfo = []
        }

    let cmdGetBoQItems = Cmd.OfAsync.perform cocoTenderApi.getBoQItems () GotBoQItems
    let cmdGetFactorFInfo = Cmd.OfAsync.perform cocoTenderApi.getFactorFInfo () GotFactorFInfo

    model, Cmd.batch [ cmdGetBoQItems; cmdGetFactorFInfo ]
```

Add 2 more cases in the update function

```fsharp
    | GotFactorFInfo (Ok info) -> { model with FactorFInfo = info }, Cmd.none
    | GotFactorFInfo (Error msg) -> showError' msg

    | ToggleFactorFView ->
        { model with ShowFactorFView = model.ShowFactorFView |> not }, Cmd.none
```

In the view, we will toggle the view through button. Add new level item next to EstimateCost.

```fsharp
    Bulma.levelItem (
        Bulma.button.button [
            prop.text "Show Factor F"
            prop.onClick (fun _ -> ToggleFactorFView |> dispatch)
        ]
    )
```

Last but not least, create the factorFView function

```fsharp
let factorFView (model: Model) dispatch =
    printfn "Info %A Factor F %A" model.FactorFInfo model.AllCost.FactorF
    let lowerBoundIndex = model.FactorFInfo |> List.tryFindIndexBack (fun x -> x |> snd |> (>=) model.AllCost.FactorF)
    printfn "LowerBoundIndex %A" lowerBoundIndex
    let factorFTr i (condition: string,factorF) =
        let isSelected =
            match lowerBoundIndex with
            | Some index -> i = index || i = (index + 1)
            | _ -> false

        Html.tr [
            if isSelected then yield prop.className "is-selected"
            yield prop.children [Html.td condition; Html.td (factorF |> string)]
        ]

    QuickView.quickview [
        if model.ShowFactorFView then yield quickview.isActive
        yield prop.children [
            QuickView.header [
                Html.div [
                    prop.style [ style.color.black ]
                    prop.text "Factor F"
                ]
                Bulma.delete [ prop.onClick (fun _ -> ToggleFactorFView |> dispatch) ]
            ]
            QuickView.body [
                QuickView.block [
                    Bulma.table [
                        Html.thead [
                            Html.tr [ Html.th "Direct Cost Condition"; Html.th "Factor F" ]
                        ]
                        Html.tbody (model.FactorFInfo |> List.mapi factorFTr)
                    ]
                ]
            ]
        ]
    ]
```

And place the function below the containerBox in main view function
```fsharp
            Bulma.heroBody [
                Bulma.container [
                    Bulma.title [
                        text.hasTextCentered
                        prop.text "CoCoTender"
                    ]
                    containerBox model dispatch
                    factorFView model dispatch
                ]
            ]
```

The result should be as follow:

{{<figure src="images/show-factor-f.jpg" width="600"
          alt="Show Factor F" title="Show Factor F">}}

The final code could be clone from tag 0.3.0

```bash
git clone -b v0.3.0 https://github.com/twuttiwat/CoCoTender 
```

If you come this far, thank you so much for reading this lenghtly post with a lot of code. I am still not good at F# yet but I am still enjoy writing in the language everyday. I would like to thank Compositional-It who develop this Safe-stack framework. I really like it. 
Also I have a lot of questions in F# in the beginning. People in the F# community (slack) is very helpful. I would like to thank Zaid Ajai,Shmew, and AngelMunoz who answer my question relentlessly. 

Last but not least, thanks Sergey Tihon to let me participate in F# Advent of Code this year.