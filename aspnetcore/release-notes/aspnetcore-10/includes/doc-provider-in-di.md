## Support for IOpenApiDocumentProvider in the DI container.

ASP.NET Core has added support for `IOpenApiDocumentProvider` in the DI container.
This allows you to inject the `IOpenApiDocumentProvider` into your application and use it to access the OpenAPI document. This is useful for scenarios where you need to access the OpenAPI document outside of the context of an HTTP request, such as in a background service or a custom middleware.

Currently, if a user wants to run the startup logic associated with an application without launching an actual application service, they must use the HostFactoryResolver to launch the application and inject a no-op IServer implementation into the DI container. This helps users achieve the desired goal of launching the application without actually listening to HTTP requests and is a helpful vector for entites that want to execute some logic on the state of the application. One of our most popular scenarios for this is the need to launch the application in order to resolve endpoints that should be mapped to an OpenAPI document.

This feature takes inspiration from Aspire's `IDistributedApplicationPublisher` interface and infrastructure to provide a more streamlined API for interacting with the application without listening on an HTTP server.

For more information, see [dotnet/aspnetcore #61463](https://github.com/dotnet/aspnetcore/pull/61463)
