## Using DevExtreme with ABP Based Applications

Hi, in this step by step article, I will show you how to integrate DevExtreme components into ABP Framework based applications.

## Create the Project

ABP Framework offers startup templates to get into the business faster. We can download a new startup template using [ABP CLI](https://docs.abp.io/en/abp/latest/CLI):

````bash
abp new DevExtremeSample
````

After the download is finished, open the solution in the Visual Studio (or your favorite IDE):

![initial-project](initial-project.png)

Run the `DevExtremeSample.DbMigrator` application to create the database and seed initial data (which creates the admin user, admin role, related permissions, etc). Then we can run the `DevExtremeSample.Web` project to see our application working.

> _Default admin username is **admin** and password is **1q2w3E\***_

## Install DevExtreme

You can follow [this documentation](https://js.devexpress.com/Documentation/17_1/Guide/ASP.NET_MVC_Controls/Prerequisites_and_Installation/) to install DevExpress packages into your computer.

> Don't forget to add _"DevExpress NuGet Feed"_ to your **Nuget Package Sources**.

### Adding DevExtreme NuGet Packages

Add the `DevExtreme.AspNet.Core` NuGet package to the `DevExtremeSample.Application.Contracts` project.

```
Install-Package DevExtreme.AspNet.Core
```

Add the `DevExtreme.AspNet.Data` package to your `DevExtremeSample.Web` project.

```
Install-Package DevExtreme.AspNet.Data
```



### Adding DevExtreme NPM Depencies

Open your `DevExtremeSample.Web` project folder with a command line and add `devextreme` NPM package:

````bash
npm install devextreme
````

### Adding Resource Mappings

The `devextreme` NPM package is saved under `node_modules` folder. We need to move the needed files in our `wwwroot/libs` folder to use them in our web project. We can do it using the ABP [client side resource mapping](https://docs.abp.io/en/abp/latest/UI/AspNetCore/Client-Side-Package-Management) system.

Open the `abp.resourcemapping.js` file in your `DevExtremeSample.Web` project and add the following definition to inside `mappings` object.

````json
"@node_modules/devextreme/dist/**/*": "@libs/devextreme/"
````

The final `abp.resourcemapping.js` file should look like below:

```
module.exports = {
  aliases: {},
  mappings: {
    "@node_modules/devextreme/dist/**/*": "@libs/devextreme/",
  },
};
```

Open your `DevExtremeSample.Web` project folder with a command line and run the `gulp` command. This command will copy the needed library files into the ``/wwwroot/libs/devextreme/` folder.

![gulp](gulp.png)

You can see `devextreme` folder inside the `wwwroot/libs`:

![wwwroot-lib](wwwroot-lib.png)

### Adding DevExtremeStyleContributor

We will add DevExtreme CSS files to the global bundle by creating a [bundle contributor](https://docs.abp.io/en/abp/latest/UI/AspNetCore/Bundling-Minification). 

Create a `Bundling` folder in the `DevExtremeSample.Web` project and a `DevExtremeStyleContributor.cs` file with the following content:

```csharp
using System.Collections.Generic;
using Volo.Abp.AspNetCore.Mvc.UI.Bundling;

namespace DevExtremeSample.Web.Bundling
{
    public class DevExtremeStyleContributor : BundleContributor
    {
        public override void ConfigureBundle(BundleConfigurationContext context)
        {
            context.Files.AddIfNotContains("/libs/devextreme/css/dx.common.css");
            context.Files.AddIfNotContains("/libs/devextreme/css/dx.light.css");
        }
    }
}
```

> You can choose another theme than the light theme. Check the `/libs/devextreme/css/` folder and the DevExtreme documentation for other themes.

Open your `DevExtremeSampleWebModule.cs` file in your `DevExtremeSample.Web` project and add following code into the `ConfigureServices` method:

```csharp
Configure<AbpBundlingOptions>(options =>
{
    options
        .StyleBundles
        .Get(StandardBundles.Styles.Global)
        .AddContributors(typeof(DevExtremeStyleContributor));
});
```

### Adding DevExtremeScriptContributor

We can not add DevExtreme js packages to Global Script Bundles, just like done for the CSS files. Because DevExtreme requires to add its JavaScript files into the `<head>` section of the HTML document, while ABP Framework adds all JavaScript files to the end of the `<body>` (as a best practice).

Fortunately, ABP Framework has a [layout hook system](https://docs.abp.io/en/abp/latest/UI/AspNetCore/Customization-User-Interface#layout-hooks) that allows you to add any code into some specific positions in the HTML document. All you need to do is to create a `ViewComponent` and configure the layout hooks.

Let's begin by creating a `DevExtremeScriptContributor.cs` file in the `Bundling` folder by copying the following code inside it:

```csharp
using System.Collections.Generic;
using Volo.Abp.AspNetCore.Mvc.UI.Bundling;
using Volo.Abp.AspNetCore.Mvc.UI.Packages.JQuery;
using Volo.Abp.Modularity;

namespace DevExtremeSample.Web.Bundling
{
    [DependsOn(
        typeof(JQueryScriptContributor)
        )]
    public class DevExtremeScriptContributor : BundleContributor
    {
        public override void ConfigureBundle(BundleConfigurationContext context)
        {
            context.Files.AddIfNotContains("/libs/devextreme/js/dx.all.js");
            context.Files.AddIfNotContains("/libs/devextreme/js/dx.aspnet.mvc.js");
            context.Files.AddIfNotContains("/libs/devextreme/js/dx.aspnet.data.js");
        }
    }
}
```

As you see, the `DevExtremeScriptContributor` is depends on `JQueryScriptContributor` which adds JQuery related files before the DevExpress packages (see the [bundling system](https://docs.abp.io/en/abp/latest/UI/AspNetCore/Bundling-Minification) for details).

#### Create DevExtremeJsViewComponent

Create a new view component, named `DevExtremeJsViewComponent` inside the `/Components/DevExtremeJs` folder of the Web project, by following the steps below:

1) Create a `DevExtremeJsViewComponent` class inside the `/Components/DevExtremeJs` (create the folders first):

```csharp
using Microsoft.AspNetCore.Mvc;
using Volo.Abp.AspNetCore.Mvc;

namespace DevExtremeSample.Web.Components.DevExtremeJs
{
    public class DevExtremeJsViewComponent : AbpViewComponent
    {
        public IViewComponentResult Invoke()
        {
            return View("/Components/DevExtremeJs/Default.cshtml");
        }
    }
}
```

2) Create `Default.cshtml` file in the same folder with the following content:

```csharp
@using DevExtremeSample.Web.Bundling
@addTagHelper *, Volo.Abp.AspNetCore.Mvc.UI.Bundling

<abp-script type="typeof(DevExtremeScriptContributor)" />
```

Your final Web project should be like as the following:

![devextreme-js](devextreme-js.png)

3) Now, we can add this view component to `<head>` section by using the layout hooks.

Open your `DevExtremeSampleWebModule.cs` file in your `DevExtremeSample.Web` project and add following code into the `ConfigureServices` method:

```csharp
Configure<AbpLayoutHookOptions>(options =>
{
    options.Add(
        LayoutHooks.Head.Last, //The hook name
        typeof(DevExtremeJsViewComponent) //The component to add
    );
});
```

#### Known Issue: Uncaught TypeError: MutationObserver.observe: Argument 1 is not an object.

> This issue does exists in the ABP Framework v3.0 and earlier versions. If you are using ABP Framework v3.1 or a latter version, you can skip this section.

When you run your `*.Web` project, you will see an exception (`Uncaught TypeError: MutationObserver.observe: Argument 1 is not an object.`) at your console.

To fix that issue, download this file [abp.jquery.js](https://github.com/abpframework/abp/blob/dev/npm/packs/jquery/src/abp.jquery.js) and replace with the `wwwroot/libs/abp/jquery/abp.jquery.js` file of your Web project.

### Result

The installation step was done. You can use any DevExtreme component in your application.

Example: A button and a progress bar component:

![devexp-result](devexp-result.gif)

This example has been created by following [this documentation](https://js.devexpress.com/Demos/WidgetsGallery/Demo/ProgressBar/Overview/NetCore/Light/).

## Sample Application

We have created a sample application with [Tree List](https://demos.devexpress.com/ASPNetCore/Demo/TreeList/Overview/) and [Data Grid](https://demos.devexpress.com/ASPNetCore/Demo/DataGrid/Overview/) examples.

You can download the source code from [here](https://github.com/abpframework/abp-samples/tree/master/DevExtreme-Mvc).

We have some notes about this sample and general usages of DevExtreme at ABP based application.

### Data Storage

We will use an in-memory list for using data storage for this sample.

There is a `SampleDataService.cs` file in `Data` folder at `*.Application.Contracts` project. We store all sample data here.

We did not create `Entities` etc. Because we want to show "How to use DevExtreme?", because of that, in this sample we focused to application and UI layer.

### JSON Serialization

You can see some `[JsonProperty(Name = "OrderId")]` attributes at DTO's. In this sample, we use that attribute on DTO's properties because DevExtreme official resource is suggesting to _disable the conversion in the JSON serializer_ [(ref)](https://js.devexpress.com/Documentation/19_1/Guide/Angular_Components/Visual_Studio_Integration/Add_DevExtreme_to_an_ASP.NET_Core_Angular_Application/#Troubleshooting). **DO NOT DO THAT!**

If you change **the conversion in the JSON serializer**, some pre-build abp modules may occur a problem.

### MVC

You can use some DevExtreme functions to create UI. The following code blocks show you how you can use it with ABP Applicaion Services.

```csharp
Html.DevExtreme().DataGrid<Order>()
            .DataSource(d => d.Mvc()
                .Controller("Order") // Application Service Name 'without **AppService**'
                .LoadAction("GetOrders") // Method Name 'without **Async**'
                .InsertAction("InsertOrder")
                .UpdateAction("UpdateOrder")
                .DeleteAction("DeleteOrder")
                .Key("OrderID")
            )
```

```csharp
public class OrderAppService : DevExtremeSampleAppService, IOrderAppService
{
    public async Task<LoadResult> GetOrdersAsync(DataSourceLoadOptions loadOptions)
    {
        ...
    }

    public async Task<Order> InsertOrder(string values)
    {
        ...
    }
    ...
}
```