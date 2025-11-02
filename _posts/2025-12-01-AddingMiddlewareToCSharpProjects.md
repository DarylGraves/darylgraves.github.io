---
title: Adding Middleware to C Sharp Web API Projects
date: 2025-12-1 00:00:00 -000
image: assets/img/BlogPosts/CSharpIcon.png
categories: [Dev]
description: A quick guide on how to set up Middleware on C Sharp Web APIs
tags: [CSharp, Dev, Api]
---

# Summary
Middleware is a class or delegate in a web app which runs for each incoming HTTP request. After executing its logic, it can choose whether to pass the request to the next middleware in the pipeline.

A good example of middleware is `UseAuthentication()`. This can stop users from proceeding further if they have not signed in. Using middleware for this prevents adding code to each endpoint to check if the user is authenticated first.

# Template
```csharp
public class ExampleMiddleware
{
    private readonly RequestDelegate _next; // Storing the next location

    public ExampleMiddleware(RequestDelegate next) // Constructor
    {
        _next = next;
    }

    public async Task InvokeAsync(HttpContext context) // Other Services can be injected here, e.g. DbContext
    {
        // Do something

        await _next(context); // Forward the request on
    }
}
```
To add this to your project you must then add the middleware to `program.cs`:
```csharp
app.UseAuthentication();
app.UseMiddleware<ExampleMiddleware>(); // The position of this line matters - Here, UseAuthentication() will run first!
app.UseAuthorization();
```

# Example
Below is a real example of middleware I wrote to help create new users in a database. Instead of storing passwords, I've configured my API to trust users who sign in from Entra and then when the authenticated user makes any request, the middleware below checks to see if the user exists in the database and adds them if required so we can start storing data against the user record.

```csharp
public class EnsureUserExistsMiddleware
{
    private readonly RequestDelegate _next;
    public EnsureUserExistsMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public async Task InvokeAsync(HttpContext context, DataContext dbContext)
    {
        if (context.User.Identity?.IsAuthenticated == true) // Check the user is authenticated
        {
            var email = context.User.FindFirst(ClaimTypes.Email)?.Value
                ?? context.User.FindFirst("email")?.Value
                ?? context.User.FindFirst("preferred_username")?.Value; // Entra Auth uses this as the claim

            if (email != null)
            {
                // Try to find the user in the database
                var user = await dbContext.Users.FirstOrDefaultAsync(u => u.Email == email);

                if (user is not null)
                {
                    // Keep a note of the user details.
                    // Since the HttpContext is passed down the middleware chain 
                    // we can retrieve this data later to prevent a duplicate call to the db
                    context.Items["User"] = user;
                }
                else
                {
                    // Create the user in the db
                    var newUser = new User()
                    {
                        Email = email,
                        DateCreated = DateTime.UtcNow,
                    };

                    try
                    {
                        await dbContext.Users.AddAsync(newUser);
                        await dbContext.SaveChangesAsync();
                        context.Items["User"] = newUser;
                    }
                    catch
                    {
                        // Another request created the user first so no action required.
                    }
                }
            }
        }

        // Go to the next middleware/function in the chain
        await _next(context);
    }
}
```
It should be noted that this isn't actually very efficient because every authenticated API call will attempt to retrieve the user from the database again! Instead, we should get the user once and then store it in a `MemoryCache` for a period of time. Then we can make future attempts to retrieve the user check the `MemoryCache` first to significantly reduce the strain against the database. Implementing `MemoryCache` will most likely be the topic of the next post.