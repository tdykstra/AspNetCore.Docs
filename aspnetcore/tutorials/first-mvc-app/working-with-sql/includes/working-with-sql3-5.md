:::moniker range=">= aspnetcore-3.1 < aspnetcore-6.0"

The `MvcMovieContext` object handles the task of connecting to the database and mapping `Movie` objects to database records. The database context is registered with the [Dependency Injection](xref:fundamentals/dependency-injection) container in the `ConfigureServices` method in the `Startup.cs` file:

# [Visual Studio](#tab/visual-studio)

[!code-csharp[](~/tutorials/first-mvc-app/start-mvc/sample/MvcMovie3/Startup.cs?name=snippet_ConfigureServices&highlight=5-6)]

The ASP.NET Core [Configuration](xref:fundamentals/configuration/index) system reads the `ConnectionString` key. For local development, it gets the connection string from the `appsettings.json` file:

[!code-json[](~/tutorials/first-mvc-app/start-mvc/sample/MvcMovie/appsettings.json?highlight=2&range=8-10)]

# [Visual Studio Code / Visual Studio for Mac](#tab/visual-studio-code+visual-studio-mac)

[!code-csharp[](~/tutorials/first-mvc-app/start-mvc/sample/MvcMovie3/Startup.cs?name=snippet_UseSqlite&highlight=5-6)]

The ASP.NET Core [Configuration](xref:fundamentals/configuration/index) system reads the `ConnectionString`. For local development, it gets the connection string from the `appsettings.json` file:

[!code-json[](~/tutorials/first-mvc-app/start-mvc/sample/MvcMovie22/appsettingsSQLite.json?highlight=2&range=8-10)]

---

[!INCLUDE [managed-identities](~/includes/managed-identities-test-non-production.md)]

# [Visual Studio](#tab/visual-studio)

## SQL Server Express LocalDB

LocalDB:

* Is a lightweight version of the SQL Server Express Database Engine, installed by default with Visual Studio.
* Starts on demand by using a connection string.
* Is targeted for program development. It runs in user mode, so there's no complex configuration.
* By default creates *.mdf* files in the *C:/Users/{user}* directory.

### Examine the database

From the **View** menu, open **SQL Server Object Explorer** (SSOX).

![View menu](~/tutorials/first-mvc-app/working-with-sql/_static/ssox5.png)

Right-click on the `Movie` table **> View Designer**

![Right-click on the Movie table > View Designer](~/tutorials/first-mvc-app/working-with-sql/_static/design.png)

![Movie table open in Designer](~/tutorials/first-mvc-app/working-with-sql/_static/dv.png)

Note the key icon next to `ID`. By default, EF makes a property named `ID` the primary key.

Right-click on the `Movie` table **> View Data**

![Right-click on the Movie table > View Data](~/tutorials/first-mvc-app/working-with-sql/_static/ssox2.png)

![Movie table open showing table data](~/tutorials/first-mvc-app/working-with-sql/_static/vd22.png)

# [Visual Studio Code / Visual Studio for Mac](#tab/visual-studio-code+visual-studio-mac)

[!INCLUDE[](~/includes/rp/sqlite.md)]
[!INCLUDE[](~/includes/RP-mvc-shared/sqlite-warn.md)]

---
<!-- End of VS tabs -->

## Seed the database

Create a new class named `SeedData` in the *Models* folder. Replace the generated code with the following:

[!code-csharp[](~/tutorials/first-mvc-app/start-mvc/sample/MvcMovie3/Models/SeedData.cs?name=snippet_1)]

If there are any movies in the database, the seed initializer returns and no movies are added.

```csharp
if (context.Movie.Any())
{
    return;  // DB has been seeded.
}
```

<a name="si"></a>

### Add the seed initializer

Replace the contents of `Program.cs` with the following code:

[!code-csharp[](~/tutorials/first-mvc-app/start-mvc/sample/MvcMovie3/Program.cs)]

Test the app.

# [Visual Studio](#tab/visual-studio)

Delete all the records in the database. You can do this with the delete links in the browser or from SSOX.

Force the app to initialize, calling the methods in the `Startup` class, so the seed method runs. To force initialization, IIS Express must be stopped and restarted. You can do this with any of the following approaches:

* Right-click the IIS Express system tray icon in the notification area and tap **Exit** or **Stop Site**:

  ![IIS Express system tray icon](~/tutorials/first-mvc-app/working-with-sql/_static/iisExIcon.png)

  ![Contextual menu](~/tutorials/first-mvc-app/working-with-sql/_static/stopIIS.png)

    * If you were running VS in non-debug mode, press F5 to run in debug mode
    * If you were running VS in debug mode, stop the debugger and press F5

# [Visual Studio Code / Visual Studio for Mac](#tab/visual-studio-code+visual-studio-mac)

Delete all the records in the database. Stop and start the app  so the `SeedData.Initialize` method runs and seeds the database.

---

The app shows the seeded data.

![MVC Movie app open in Microsoft Edge showing movie data](~/tutorials/first-mvc-app/working-with-sql/_static/m55.png)

> [!div class="step-by-step"]
> [Previous: Adding a model](~/tutorials/first-mvc-app/adding-model.md)
> [Next: Adding controller methods and views](~/tutorials/first-mvc-app/controller-methods-views.md)

:::moniker-end
