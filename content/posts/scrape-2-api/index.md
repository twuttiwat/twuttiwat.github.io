+++
title = 'From Scrape To API in 1 Day'
date = 2023-12-07T14:17:09+07:00
draft = false
+++

{{<figure src="images/intro-cartoon.webp" width="600"
          alt="Intro Cartoon" title="Converting Scrape Data to Api in my Dream">}}

Introduction
-------------

**TLDR** If you prefer just code reading like me, you can skip to the [repo](https://github.com/twuttiwat/Scrape2Api)

One of the somewhat mundane but necessary tasks in programming is to get data from externa Web sites (a.k.a Web Scraping) and store in easy to use format. These raw data when format properly can be consumed by multiple type of client through Api endpoints.

Since my planned advent of calendar post is not working as I plan. I will challenge myself to see if I can really use F# proficiently or not. 

I love to read tutorial following first principle in that it won't try to give the correct answer upfront. Instead, it will show problem along the way. I ,especially, like to understand what is inside the head of programmer when coding.

This post will follow this style.


Background
-----------
All construction projects require price estimation before start bidding the project. Price for each material comes from Website provided by ministry of commerce. Unfortunately, the website is created in ASP era. (means ActiveX object with ODBC and Oracle). It would be very hard for any client to consume the data.

Your mission is to convert such Web site into Api in one day.

Our starting point is from following website:

{{<figure src="images/start-materials-web.png" width="600"
          alt="Start Materials Web" title="Materials Price List">}}

In fact, we just want following data:

| Code | Description | Unit | Price |
|------|-------------|------|-------|
| 0101010107200000 |  รูปลูกบาศก์ 300 กก./ตร.ซม. และรูปทรงกระบอก 250 กก./ตร.ซม. ตราซีแพค | ลบ.ม. | 2,493.4 |



When entering following curl:

```bash
curl https://localhost:5051/materials
```

It should return all materials in json array:

```json
[ 
    {
        "Code": "0101010107200000"
        "Description": "รูปลูกบาศก์ 300 กก./ตร.ซม...."
        "Unit": "ลบ.ม."
        "Price": "2493.4"
    }    
]
```

Sketch Pipeline
---------------

When starting any F# project, I like to sketch core pipeline to see how data flow from start to end.
Implement each one by one as F# function and combine it at the end.

1. Request page from given url and save in local file system
2. Parse file into F# Domain type
3. Store F# type into SQLite + Donald
4. Create Api Service with Falco


Requesting Data
---------------

We will start by browsing with [materials url](http://www.indexpr.moc.go.th/PRICE_PRESENT/tablecsi_month_region.asp?DDMonth=11&DDYear=2566&DDProvince=10&B1=%B5%A1%C5%A7)

Well, if you click the link, you won't get the price list table. Instead, you will see following:


{{<figure src="images/materials-web-blank.png" width="600"
          alt="Materials Web Blank" title="Initial Url has no Price List">}}

Firefox DevTools is your friend here. Press Ctrl-Shift-I, go to Network Tab, and click 'ตกลง' (Ok in English) in the Web page.
You will see the price list so clicking on the button should do the trick.
Now, look at below request:

{{<figure src="images/materials-network-request.png" width="600"
          alt="Materials Network Request" title="View Network Request in Materials Web">}}

The request require form data: **DDGroupCode** and **Submit**

Let's make our hand (actually keyboard) dirty. Fire up VSCode and create new file call Scratch.fsx.
BTW: I literally copy [HttpClient code](https://dev.to/tunaxor/making-http-requests-in-f-1n0b) from Daniel
Thanks! Daniel.

We are going to use HttpClient to Post form data to Url, get the response and save it to file.

```fsharp
let url = "http://www.indexpr.moc.go.th/PRICE_PRESENT/tablecsi_month_region.asp?DDMonth=11&DDYear=2566&DDProvince=10&B1=%B5%A1%C5%A7"            
let materialsHtml = @"c:/temp/Scrape2Api/materials.html" 

let request () =
    task {
        
        use file = File.OpenWrite(materialsHtml)

        // Post form data to material url
        use client = new HttpClient()
        let formData = 
            [ "DDGroupCode", ""; "Submit", "%B5%A1%C5%A7" ] 
            |> Seq.map (fun (k, v) -> KeyValuePair<string, string>(k, v))
        let content = new FormUrlEncodedContent(formData)

        let! response = client.PostAsync(url, content)

        // Save web page to local file
        do! response.Content.CopyToAsync(file)
    }   
    |> Async.AwaitTask
    |> Async.RunSynchronously 

```

Run following code in fsx file, you should get materials.html in c:\temp\Scratch2Api folder.
We can get raw html file now. 

Parsing Page
-------------

Next task is to parse web page to Material domain type:

```fsharp
type Material = {
    Code: string
    Description: string
    Unit: string
    UnitPrice: decimal
}
```

The web is encoded in Windows-874 language, we need to convert to Unicode (UTF-8) wit following funtion:

```fsharp
let toUTF8 sourceFilePath =
    // Read the contents of the file with the source encoding
    let sourceEncoding = Encoding.GetEncoding(874) // ISO-8859-1 encoding    
    let content = File.ReadAllText(sourceFilePath, sourceEncoding)

    // Write the contents to a new file with the target encoding
    let targetEncoding = Encoding.UTF8 // UTF-8 encoding
    File.WriteAllText(materialsUtfHtml, content, targetEncoding)

```

Personally, I don't like to create new file just for encoding like this. It should be just copy input to output stream and change encoding on the fly. But this is good enough.


We are going to extract html document. I choose class HtmlDocument from FSharp.Data.
The step to extract material row is as follow
- Extract all TR rows


```fsharp
let htmlStream = File.OpenRead(materialsUtfHtml)
let html = HtmlDocument.Load(htmlStream)

// This will extract all TR elements from HTML document
let trs = html.Descendants("tr")

```

- Filter out only row containg price at column 4th

```fsharp
trs 
|> Seq.map (fun tr -> tr.Descendants("td") |> Seq.map (fun td -> td.InnerText()))
|> Seq.filter (fun tds -> tds |> Seq.length |> fun len -> len >= 4)
|> Seq.filter (fun tds ->                 
                    match tds |> Seq.item 3 |> System.Decimal.TryParse with
                    | isDecimal, _ -> isDecimal)

```

- Convert to Material type

```fsharp

type Material = {
    Code: string
    Description: string
    Unit: string
    UnitPrice: decimal
}

// Just set each td string into Material field
let toMaterial (tds:string seq) = 
    let getText i = tds |> Seq.item i |> fun x -> x.Trim()

    {
        Code = getText 0
        Description = getText 1
        Unit = getText 2
        UnitPrice = getText 3 |> System.Decimal.Parse  
    }

```

Storing Data
-------------

Phew, we are halfway through the process and I am staring to feel sleepy now. 
First, we need to create Material table in SQLite:

```sql
CREATE TABLE "material" (
	"code"	TEXT,
	"description"	TEXT,
	"unit"	TEXT,
	"unit_price"	NUMERIC,
	PRIMARY KEY("code")
)
```

I prefer to create DTO myself, I will use Donald for this project since I never use it before so why not.
What I like about about Donald and DustyTables are the simplicity of them. I can start by writing raw SQL like this:

```fsharp

// SQL to upsert data into material table
let sql = "
    INSERT INTO material (code, description, unit, unit_price)
    VALUES (@code, @description, @unit, @unit_price)
    ON CONFLICT do UPDATE SET description = @description, unit = @unit, unit_price = @unit_price";


// Strongly typed input parameters
let param = [ 
    "code", sqlString firstMaterial.Code
    "description", sqlString firstMaterial.Description
    "unit", sqlString firstMaterial.Unit
    "unit_price", sqlDecimal firstMaterial.UnitPrice
]
```

And just apply it through Donald:

```fsharp
    conn
    |> Db.newCommand sql
    |> Db.setParams param
    |> Db.exec // unit
```


Make Api
--------

I started to like simplicity of Donald so I decide to use Falco as well to see how it fits.
I did create HelloWorldApp in Falco, adding one end point like this.

```fsharp
[<EntryPoint>]
let main args =
    webHost args {
        endpoints [
            get "/materials" (getAllMaterials ())
        ]
    }
    0
```

The handler getAllMaterials will just fetch data from SQLite using Donald.

```fsharp
let getAllMaterials () =
    use conn = new SQLiteConnection(@"Data Source=C:\Scrape2Api\material.db;Version=3;New=true;")

    let sql = "SELECT code, description, unit, unit_price FROM material"
    
    conn
    |> Db.newCommand sql
    |> Db.query Material.ofDataReader
    |> Response.ofJson
````

Of all the component, I found that I like Falco the most. The code are so minimal.

Putting Everything Together
---------------------------

Finally, we are at my most favourite part. Just assemble all the piece together.
All code so far (except Api), is created in Scratch.fsx file.
We need to package everything into application. There are 2 applications for this project

1. Scraper console application which do request, parse, and store data into SQLite.
2. Material Api will provide Rest api to list all materials information.

Scraper Console Application
----------------------------

This is versy simple just create new F# console project. Add necessary package and add following function.

```fsharp
let scrape () =
    request ()
    |> toUTF8
    |> parse
    |> saveToDb
```

We just pipe all function we have so far into one single __scrape__ function. To protect the data server, we will run just once a day.

```fsharp
async {
    let oneDay = 1000 * 60 * 60 * 24    
    while true do
        do scrape ()
        do! Async.Sleep oneDay
}
|> Async.RunSynchronously
```

Running the scraper should create material.db database in "C:\temp\Scrape2Api\"

{{<figure src="images/done-scraping.png" width="300"
          alt="Done Scraping" title="Result of Scraping">}}

Material Api
------------

Since all the function has been implemented in test project. Just move that project into solution and it's done.

And here my small moment of truth:

{{<figure src="images/done-api.png" width="600"
          alt="Done Api" title="Material Api">}}


Conclusion
----------

Whew, I really finish scrape data and convert to api in one day. The most time-consuming part is the parsing since you somehow need to know the existing logic. The rest is very simple especially creating Api with Falco. I plan to try Falco web framework in my next side project.

Happy X'Mas!!
Hope this year will be have more lambda to apply! 😄


