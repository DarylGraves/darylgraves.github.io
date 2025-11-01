---
title: Adding Swagger to C# API Projects
date: 2025-11-1 00:00:00 -000
image: assets/img/BlogPosts/CSharpIcon.png
categories: [Dev]
description: Since Dotnet 9.0 was released, Swagger no longer comes with the webapi project and needs to be installed manually instead.
tags: [CSharp, Dev, Api]
---

# Problem
Since Dotnet 9.0 was released, Swagger no longer comes with the webapi project and needs to be installed manually instead. Below are the steps to do this.

# Solution
1) Add the package to your dotnet project by running the below in the Command Line:

```bash
dotnet add package Swashbuckle.AspNetCore
```

2) In the `Program.cs` file, add these two lines before `builder.Build();`

```csharp
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();
```

3) Still in the `Program.cs` file, add after `var app = builder.Build();`

```csharp
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}
```

4) On any endpoint you want viewable in Swagger should use `WithOpenApi()`. For example:

```csharp
app.MapGet("/example", () => "Hello)
    .WithOpenApi();
```

And that is it! Now when you run your Dotnet project in dev you can navigate to https://localhost:<port>/swagger to view your endpoints and try them through the browser.

# Would you like to learn more?
See the Microsoft Documentation [here](https://learn.microsoft.com/en-us/aspnet/core/tutorials/getting-started-with-swashbuckle?view=aspnetcore-8.0&tabs=visual-studio).

# Personal Note
I have been struggling to think of things to write about hence skipping a few months but I have been working on a lot of C# projects recently so the next few posts will probably be focused around this.