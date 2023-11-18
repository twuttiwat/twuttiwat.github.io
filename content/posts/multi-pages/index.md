---
title: "From Single to Multiple Pages in SPA"
date: 2022-12-20T14:57:37+07:00
draft: false
author: Teerawat Wuttiwat
---

Introduction
============

Supporting multiple pages is one of the obstacle I found when using safe stack. I think it might be the nature of single page application. It's like we are trying to fight against simple server side rendering somehome. Anyway, I will try my best to write this post in step by step manners. So everyone will understand.


There are quite a few tutorial on adding multi-page support for safe stack platform.
Most of them will focus on using propagate message from parent page to child pages.

After trying with many routes, I decide to go with component styled based on Yobo application.
Each child page will be implemented as React function component. It's the responsibility of the Index page to render the child page component inside it's main view function.

```goat
.-----------.
|   Index   |
| .-------. |
| |Content| |
| '-------' |      
'-----------'
```    

The Index page will contain navigation bar (navbar) which should be shared across all pages. The Content will be content specific in that page. For example, we can have BoQ page which is a react component hosting BoQItems grid implemented in the past posts.

Without further ado, let's see our plan of attack for this post:

Our Plans
==========
1. Create BoQ Page and Home Page
2. Modify Index Page to show content of Home Page
3. Modify Home Page to show list of dummy Projects
4. Route user from Home Page to BoQ Page when clicking on Project row

Create BoQ Page and Home Page
-----------------------------
Start by creating Pages folder inside the src/Client folder and create each Page file as fsharp one.
Do not forget to have the correct order. You can both create fsharp file and order it with Ionide sidebar. Here is what order should be like in Client.fsproj:

```fsharp
    <Compile Include="Pages\Home.fs" />
    <Compile Include="Pages\BoQ.fs" />
    <Compile Include="Index.fs" />
```
Since both Home and BoQ are needed in Index.fs (for rendering), they must be declare above Index.fs file. 

Next, copy content of current Index.fs to BoQ.fs. You can leave the Index.fs blank at the moment. We will re-code them in the next step.

Create Blank Home Page
-----------------------
One of my favourite feature of safe stack is the Hot Module Replacement (HMR). I like to always run the app while coding so I know whenever something wrong happens.

```bash
dotnet run
```

Next, we will create simple Index page with no Content in it. Just the blank Bulma container box.

```fsharp
module Index

open System
open Elmish
open Fable.Remoting.Client
open Elmish.Toastr
open Shared

type Model =
    {
        PageName : string
    }

type Msg =
    | DoNothing


let init () : Model * Cmd<Msg> =
    { PageName = "Index" }, Cmd.none

let update (msg: Msg) (model: Model) : Model * Cmd<Msg> =
    match msg with
    | DoNothing ->
        model, Cmd.none

open Feliz
open Feliz.Bulma

let navBrand =
    Bulma.navbarBrand.div [
        Bulma.navbarItem.a [
            prop.href "https://safe-stack.github.io/"
            navbarItem.isActive
            prop.children [
                Html.img [
                    prop.src "/favicon.png"
                    prop.alt "Logo"
                ]
            ]
        ]
    ]

let containerBox (model: Model) (dispatch: Msg -> unit) =
    Bulma.container []

let view (model: Model) (dispatch: Msg -> unit) =
    Bulma.hero [
        hero.isFullHeight
        color.isPrimary
        prop.style [
            style.backgroundSize "cover"
            style.backgroundImageUrl "https://unsplash.it/1200/900?random"
            style.backgroundPosition "no-repeat center center fixed"
        ]
        prop.children [
            Bulma.heroHead [
                Bulma.navbar [
                    Bulma.container [ navBrand ]
                ]
            ]
            Bulma.heroBody [
                Bulma.container [
                    containerBox model dispatch
                ]
            ]
        ]
    ]
```

If you look at the result, you should see just blank page.
Now we will create Home page and put it in the container box.

{{<figure src="images/blank-index.jpg" width="600"
          alt="Blank Index Page" title="Blank Index Page">}}

Before creating Home page, we need to install one library.
**Feliz.UseElmish** is the package that act as a bridge between Elmish and React. 

```bash
dotnet paket add Feliz.UseElmish -p Client
```

Go to Home.fs and paste below code.

```fsharp
module Pages.Home

open System
open Elmish
open Shared

type Model =
    {
        PageName : string
    }

type Msg =
    | DoNothing

let init () : Model * Cmd<Msg> =
    let model =
        {
            PageName = "Home"
        }

    model, Cmd.none

let update (msg: Msg) (model: Model) : Model * Cmd<Msg> =
    match msg with
    | DoNothing ->
        model, Cmd.none

open Feliz
open Feliz.UseElmish
open Feliz.Bulma

let view = React.functionComponent(fun () ->
    let (model: Model), dispatch = React.useElmish(init, update, [| |])

    Bulma.title model.PageName
)

```

The part I would like you to focus on is the **view** function. The view function will return React function component which will be used as the content in the Index page. The bridge between Elmish and React is on the next line. **React.useElmish** will receive **init** and **update** function as input and return **model** and **dispatch**. 

By using the **Feliz.UseElmish** library you can introduce React component into Elmish world.

Back to the view function, for simplicity we will just show the name of the page.

Next go to containerBox function in Index.fs and change it to render Home page by calling **view** function.

```fsharp
let containerBox (model: Model) (dispatch: Msg -> unit) =
    Pages.Home.view()
```

If you look at the browser, you should see "Home" show in the Index page. This render of **Home** title comes from Home.fs

{{<figure src="images/blank-home.jpg" width="600"
          alt="Blank Home Page" title="Blank Home Page">}}

Show List of Projects
----------------------

Showing just the page name is not fun. Let's try showing list of projects in the Home page. 
First, define ProjectDto and initialise it with some test data.

```fsharp
module Pages.Home

...

type ProjectDto = {
    Id : Guid
    Name : string
    EstimateCost : float
}

type Model =
    {
        PageName : string
        Projects : ProjectDto list
    }


let init () : Model * Cmd<Msg> =
    let model =
        {
            PageName = "Home"
            Projects = [ for _ in 1..3 do {Id = Guid.NewGuid(); Name = "Swimming Pool"; EstimateCost = 1500.0} ]
        }

    model, Cmd.none

```

Next, create new projectsGrid function to render list of projects from our test data.
You can check how to use grid in previous safe-stack post.

```fsharp
open Feliz.AgGrid


let projectsGrid model dispatch =

    Html.div [
        prop.className ThemeClass.Balham
        prop.children [
            Bulma.button.button [
                color.isInfo
                prop.text "Add"
            ]
            AgGrid.grid [
                AgGrid.rowData (model.Projects |> Array.ofList)
                AgGrid.defaultColDef [
                    ColumnDef.resizable true
                ]
                AgGrid.domLayout AutoHeight
                AgGrid.onGridReady (fun x -> x.AutoSizeAllColumns())
                AgGrid.enableCellTextSelection true
                AgGrid.ensureDomOrder true
                AgGrid.columnDefs [
                    ColumnDef.create<string> [
                        ColumnDef.headerName "Name"
                        ColumnDef.valueGetter (fun x -> x.Name)
                    ]
                    ColumnDef.create<float> [
                        ColumnDef.headerName "Estimate Cost"
                        ColumnDef.valueGetter (fun x -> x.EstimateCost)
                        ColumnDef.width 75
                    ]
                ]
            ]
        ]
    ]
```

Finally, let's render the grid in view function.

```fsharp
let view = React.functionComponent(fun () ->
    let (model: Model), dispatch = React.useElmish(init, update, [| |])

    projectsGrid model dispatch
)
```

If you look at the browser, you should see list of test projects in Home page.

{{<figure src="images/projects-grid.jpg" width="600"
          alt="Project Grids" title="List of Projects">}}

Route from Home page to BoQ page
--------------------------------

So far so good, we can show child page (Home) inside parent page (Index). But this it rarely useful, we should be able to go between pages.

We are going to use Feliz.Router. Start by installing it.

```bash
dotnet paket add Feliz.Router -p Client
```

Feliz.Router will take cares of observing Url changes and trigger our supplied function.
Start by creating new function that will render main parent page with supplied child page.

```fsharp
let viewPage (model: Model) (dispatch: Msg -> unit) (pageContent:ReactElement) =
    Bulma.hero [
        hero.isFullHeight
        color.isDark
        prop.children [
            Bulma.heroHead [
                Bulma.navbar [
                    Bulma.container [ navBrand ]
                ]
            ]
            Bulma.heroBody [
                Bulma.container [
                    Bulma.title [
                        text.hasTextCentered
                        prop.text "CoCoTender"
                    ]
                    pageContent
                ]
            ]
        ]
    ]
```

Next, change the view function to use Feliz.Router

```fsharp
let view (model: Model) (dispatch: Msg -> unit) =
    let activePage =
        Pages.Home.view()
        |> viewPage model dispatch

    React.router [
        router.children [ activePage ]
    ]
```

The Router will render the ReactElement within children. This is not exciting since we will still see only Home page as the content. 

Let's introduce Page domain into our app.
What I really like about F# is the way that everthing will be solved by introducing small type suitable for the task. Like this app, we are working on page so it's natural to define Page type for this.

Enough of mumbling. Let's create Router.fs in src/Client. And don't forget to include that file in Client.fsproj. It should comes right after index.html.

We will define Page type in Router module.

```fsharp
type Page =
    | Home
    | BoQ
```

Since we have only 2 pages at the moment. We will create new choice for Page when new page is created. 
Next, go to Index.fs. Since the view function needs to know which is the current page so we can call content view function correctly. We should define the Index model as follow:

```fsharp
type Model = {
    ...
    CurrentPage : Page
}
```
Then, go to the init and set Home to be the default page.

```fsharp
let init () =
    let model = {
        ...
        CurrentPage = Home
    }
```

Next in the view function of Index page, find activePage based on CurrentPage in the model.

```fsharp
    let activePage =
        match model.CurrentPage with
        | Home -> Pages.Home.view()
        | BoQ -> Pages.BoQ.view()
        |> viewPage model dispatch
```

The only things left is to update CurrentPage when url has changes.
We can do that in React.router

```fsharp
    React.router [
        router.onUrlChanged (UrlChanged >> dispatch)        
    ]
```

When url changes (such as clicking on the hyperlink), the UrlChanged message will be dispatch with segments of Url. We need to create new Message union.

```fsharp
type Msg =
    | UrlChanged of string list
```

and add the case for update function

```fsharp
    match msg with
    | UrlChanged segments ->
        let currentPage = 
            match segments with
            | [  ] -> Home
            | [ "boq" ] -> BoQ
            | _ -> Home 

        { model with CurrentPage = currentPage }, Cmd.none
```

This case will check the url segments and set the related Page. The default one will always be Home. We also match "boq" to BoQ Page.
For example
1. If user browse with http://localhost:8080 -> Home page
2. If user browse with http://localhost:8080/#/boq -> BoQ page
3. The rest will always go to Home page

To trigger the routing, we will add Edit link in each Project row. The link will have href point to BoQ page.

Go to Home.fs and add more column to projectsGrid

```fsharp
                    ColumnDef.create<Guid> [
                        ColumnDef.valueGetter (fun x -> x.Id)
                        ColumnDef.cellRendererFramework (fun id _ ->
                            Html.a [
                                prop.href (Router.format("boq"))
                                prop.text "Edit"
                            ]
                        )
                    ]
```

If you click on the Edit link, it will redirect you to BoQ page.
Now try to either 
1. Open Edit in new tab
2. Type the boq url directly in navigation bar

In both cases, you will found that the browser will just show home page. The problem arises because the UrlChanged is not triggered when page first load. The only place you could check for initial url is in the init function.

Go to init function and do following:

```fsharp
    let initUrlCmd = Router.currentUrl() |> UrlChanged |> Cmd.ofMsg
    model, initUrlCmd
```

This will dispatch UrlChanged message with current url from the location bar. Now if you try aboved cases, it should working now.

That's it for today.
You can get the finished version by cloning from my github repo:

```bash
git clone -b v0.5.0 https://github.com/twuttiwat/CoCoTender 
```

Thanks for reading!!!