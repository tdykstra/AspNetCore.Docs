---
title: Part 6, controller methods and views in ASP.NET Core
author: wadepickett
description: Part 6, add a model to an ASP.NET Core MVC app
monikerRange: '>= aspnetcore-3.1'
ms.author: wpickett
ms.date: 03/02/2025
uid: tutorials/first-mvc-app/controller-methods-views
---

# Part 6, controller methods and views in ASP.NET Core

[!INCLUDE[](~/includes/not-latest-version.md)]

By [Rick Anderson](https://twitter.com/RickAndMSFT)

:::moniker range=">= aspnetcore-9.0"

We have a good start to the movie app, but the presentation isn't ideal, for example, **ReleaseDate** should be two words.

![Index view: Release Date is one word (no space) and every movie release date shows a time of 12 AM](~/tutorials/first-mvc-app/working-with-sql/_static/9/m90.png)

Open the `Models/Movie.cs` file and add the highlighted lines shown below:

[!code-csharp[](~/tutorials/first-mvc-app/start-mvc/sample/mvcmovie90/Models/Movie.cs?name=snippet_Second&highlight=2,3,12,13,16)]

`DataAnnotations` are explained in the next tutorial. The [Display](xref:System.ComponentModel.DataAnnotations.DisplayAttribute) attribute specifies what to display for the name of a field (in this case "Release Date" instead of "ReleaseDate"). The [DataType](xref:System.ComponentModel.DataAnnotations.DataTypeAttribute) attribute specifies the type of the data (Date), so the time information stored in the field isn't displayed.

The `[Column(TypeName = "decimal(18, 2)")]` data annotation is required so Entity Framework Core can correctly map `Price` to currency in the database. For more information, see [Data Types](/ef/core/modeling/relational/data-types).

Browse to the `Movies` controller and hold the mouse pointer over an **Edit** link to see the target URL.

![Browser window with mouse over the Edit link and a link Url of https://localhost:5001/Movies/Edit/5 is shown](~/tutorials/first-mvc-app/controller-methods-views/_static/9/edit90.png)

The **Edit**, **Details**, and **Delete** links are generated by the Core MVC Anchor Tag Helper in the `Views/Movies/Index.cshtml` file.

[!code-cshtml[](~/tutorials/first-mvc-app/start-mvc/sample/MvcMovie90/Views/Movies/IndexOriginal.cshtml?highlight=1-3&range=46-50)]

[Tag Helpers](xref:mvc/views/tag-helpers/intro) enable server-side code to participate in creating and rendering HTML elements in Razor files. In the code above, the `AnchorTagHelper` dynamically generates the HTML `href` attribute value from the controller action method and route id. You use **View Source** from your favorite browser or use the developer tools to examine the generated markup. A portion of the generated HTML is shown below:

```html
 <td>
    <a href="/Movies/Edit/4"> Edit </a> |
    <a href="/Movies/Details/4"> Details </a> |
    <a href="/Movies/Delete/4"> Delete </a>
</td>
```

Recall the format for [routing](xref:mvc/controllers/routing) set in the `Program.cs` file:

[!code-csharp[](~/tutorials/first-mvc-app/start-mvc/sample/MvcMovie90/Program.cs?name=snippet_MapControllerRoute&highlight=3)]

ASP.NET Core translates `https://localhost:5001/Movies/Edit/4` into a request to the `Edit` action method of the `Movies` controller with the parameter `Id` of 4. (Controller methods are also known as action methods.)

[Tag Helpers](xref:mvc/views/tag-helpers/intro) are one of the most popular new features in ASP.NET Core. For more information, see [Additional resources](#additional-resources).

<a name="get-post"></a>

Open the `Movies` controller and examine the two `Edit` action methods. The following code shows the `HTTP GET Edit` method, which fetches the movie and populates the edit form generated by the `Edit.cshtml` Razor file.

[!code-csharp[](~/tutorials/first-mvc-app/start-mvc/sample/MvcMovie90/Controllers/MoviesController.cs?name=snippet_EditGet)]

The following code shows the `HTTP POST Edit` method, which processes the posted movie values:

[!code-csharp[](~/tutorials/first-mvc-app/start-mvc/sample/MvcMovie80/Controllers/MoviesController.cs?name=snippet_EditPost)]

The `[Bind]` attribute is one way to protect against [over-posting](/aspnet/mvc/overview/getting-started/getting-started-with-ef-using-mvc/implementing-basic-crud-functionality-with-the-entity-framework-in-asp-net-mvc-application#overpost). You should only include properties in the `[Bind]` attribute that you want to change. For more information, see [Protect your controller from over-posting](/aspnet/mvc/overview/getting-started/getting-started-with-ef-using-mvc/implementing-basic-crud-functionality-with-the-entity-framework-in-asp-net-mvc-application). [ViewModels](https://rachelappel.com/use-viewmodels-to-manage-data-amp-organize-code-in-asp-net-mvc-applications/) provide an alternative approach to prevent over-posting.

Notice the second `Edit` action method is preceded by the `[HttpPost]` attribute.

[!code-csharp[](~/tutorials/first-mvc-app/start-mvc/sample/MvcMovie90/Controllers/MoviesController.cs?name=snippet_EditPost&highlight=4)]

The `HttpPost` attribute specifies that this `Edit` method can be invoked *only* for `POST` requests. You could apply the `[HttpGet]` attribute to the first edit method, but that's not necessary because `[HttpGet]` is the default.

The `ValidateAntiForgeryToken` attribute is used to [prevent forgery of a request](xref:security/anti-request-forgery) and is paired up with an antiforgery token generated in the edit view file (`Views/Movies/Edit.cshtml`). The edit view file generates the antiforgery token with the [Form Tag Helper](xref:mvc/views/working-with-forms).

[!code-cshtml[](~/tutorials/first-mvc-app/start-mvc/sample/MvcMovie90/Views/Movies/EditOriginal.cshtml?range=13)]

The [Form Tag Helper](xref:mvc/views/working-with-forms) generates a hidden antiforgery token that must match the `[ValidateAntiForgeryToken]` generated antiforgery token in the `Edit` method of the Movies controller. For more information, see <xref:security/anti-request-forgery>.

The `HttpGet Edit` method takes the movie `ID` parameter, looks up the movie using the Entity Framework `FindAsync` method, and returns the selected movie to the Edit view. If a movie cannot be found, `NotFound` (HTTP 404) is returned.

[!code-csharp[](~/tutorials/first-mvc-app/start-mvc/sample/MvcMovie90/Controllers/MoviesController.cs?name=snippet_EditGet)]

When the scaffolding system created the Edit view, it examined the `Movie` class and created code to render `<label>` and `<input>` elements for each property of the class. The following example shows the Edit view that was generated by the Visual Studio scaffolding system:

[!code-cshtml[](~/tutorials/first-mvc-app/start-mvc/sample/MvcMovie90/Views/Movies/EditOriginal.cshtml)]

Notice how the view template has a `@model MvcMovie.Models.Movie` statement at the top of the file. `@model MvcMovie.Models.Movie` specifies that the view expects the model for the view template to be of type `Movie`.

The scaffolded code uses several Tag Helper methods to streamline the HTML markup. The [Label Tag Helper](xref:mvc/views/working-with-forms) displays the name of the field ("Title", "ReleaseDate", "Genre", or "Price"). The [Input Tag Helper](xref:mvc/views/working-with-forms) renders an HTML `<input>` element. The [Validation Tag Helper](xref:mvc/views/working-with-forms) displays any validation messages associated with that property.

Run the application and navigate to the `/Movies` URL. Click an **Edit** link. In the browser, view the source for the page. The generated HTML for the `<form>` element is shown below.

[!code-HTML[](~/tutorials/first-mvc-app/start-mvc/sample/MvcMovie90/Views/Shared/edit_view_source.html?highlight=1,6,10,17,24,28)]

The `<input>` elements are in an `HTML <form>` element whose `action` attribute is set to post to the `/Movies/Edit/id` URL. The form data will be posted to the server when the `Save` button is clicked. The last line before the closing `</form>` element shows the hidden [XSRF](xref:security/anti-request-forgery) token generated by the [Form Tag Helper](xref:mvc/views/working-with-forms).

## Processing the POST Request

The following listing shows the `[HttpPost]` version of the `Edit` action method.

[!code-csharp[](~/tutorials/first-mvc-app/start-mvc/sample/MvcMovie90/Controllers/MoviesController.cs?name=snippet_EditPost)]

The `[ValidateAntiForgeryToken]` attribute validates the hidden [XSRF](xref:security/anti-request-forgery) token generated by the antiforgery token generator in the [Form Tag Helper](xref:mvc/views/working-with-forms)

The [model binding](xref:mvc/models/model-binding) system takes the posted form values and creates a `Movie` object that's passed as the `movie` parameter. The `ModelState.IsValid` property verifies that the data submitted in the form can be used to modify (edit or update) a `Movie` object. If the data is valid, it's saved. The updated (edited) movie data is saved to the database by calling the `SaveChangesAsync` method of database context. After saving the data, the code redirects the user to the `Index` action method of the `MoviesController` class, which displays the movie collection, including the changes just made.

Before the form is posted to the server, client-side validation checks any validation rules on the fields. If there are any validation errors, an error message is displayed and the form isn't posted. If JavaScript is disabled, you won't have client-side validation but the server will detect the posted values that are not valid, and the form values will be redisplayed with error messages. Later in the tutorial we examine [Model Validation](xref:mvc/models/validation) in more detail. The [Validation Tag Helper](xref:mvc/views/working-with-forms) in the `Views/Movies/Edit.cshtml` view template takes care of displaying appropriate error messages.

![Edit view: An exception for an incorrect Price value of abc states that The field Price must be a number. An exception for an incorrect Release Date value of xyz states Please enter a valid date.](~/tutorials/first-mvc-app/controller-methods-views/_static/9/val90.png)

All the `HttpGet` methods in the movie controller follow a similar pattern. They get a movie object (or list of objects, in the case of `Index`), and pass the object (model) to the view. The `Create` method passes an empty movie object to the `Create` view. All the methods that create, edit, delete, or otherwise modify data do so in the `[HttpPost]` overload of the method. Modifying data in an `HTTP GET` method is a security risk. Modifying data in an `HTTP GET` method also violates HTTP best practices and the architectural [REST](http://rest.elkstein.org/) pattern, which specifies that GET requests shouldn't change the state of your application. In other words, performing a GET operation should be a safe operation that has no side effects and doesn't modify your persisted data.

## Additional resources

* [Globalization and localization](xref:fundamentals/localization)
* [Introduction to Tag Helpers](xref:mvc/views/tag-helpers/intro)
* [Author Tag Helpers](xref:mvc/views/tag-helpers/authoring)
* <xref:security/anti-request-forgery>
* Protect your controller from [over-posting](/aspnet/mvc/overview/getting-started/getting-started-with-ef-using-mvc/implementing-basic-crud-functionality-with-the-entity-framework-in-asp-net-mvc-application)
* [ViewModels](https://rachelappel.com/use-viewmodels-to-manage-data-amp-organize-code-in-asp-net-mvc-applications/)
* [Form Tag Helper](xref:mvc/views/working-with-forms)
* [Input Tag Helper](xref:mvc/views/working-with-forms)
* [Label Tag Helper](xref:mvc/views/working-with-forms)
* [Select Tag Helper](xref:mvc/views/working-with-forms)
* [Validation Tag Helper](xref:mvc/views/working-with-forms)

> [!div class="step-by-step"]
> [Previous](~/tutorials/first-mvc-app/working-with-sql.md)
> [Next](~/tutorials/first-mvc-app/search.md)

:::moniker-end

[!INCLUDE[](~/tutorials/first-mvc-app/controller-methods-views/includes/controller-methods-views8.md)]

[!INCLUDE[](~/tutorials/first-mvc-app/controller-methods-views/includes/controller-methods-views7.md)]

[!INCLUDE[](~/tutorials/first-mvc-app/controller-methods-views/includes/controller-methods-views6.md)]

[!INCLUDE[](~/tutorials/first-mvc-app/controller-methods-views/includes/controller-methods-views3-5.md)]
