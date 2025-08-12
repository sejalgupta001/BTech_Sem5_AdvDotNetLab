Here is a **properly formatted Markdown file** with necessary comments for better understanding:

# 🔐 Authentication & Role-Based Access Implementation in ASP.NET Core MVC

This example demonstrates how to **authenticate users** and **store their JWT token and role in session**, which can then be used to authorize access to different parts of the application.

---

## 📌 1 Register Services in `Program.cs`

```csharp
// ✅ Register HttpClient, IHttpContextAccessor, and AuthService
builder.Services.AddHttpClient();
builder.Services.AddHttpContextAccessor();
builder.Services.AddScoped<AuthService>();

// ✅ Enable Session for JWT storage
builder.Services.AddSession(options =>
{
    options.IdleTimeout = TimeSpan.FromMinutes(30); // Session expiry
});

var app = builder.Build();

// ✅ Enable Session Middleware
app.UseSession();
```

---

## 📌 2 Login View (`Login.cshtml`)

```html
<form method="post" class="col-4">
    <!-- Username -->
    <div class="form-group">
        <label>Username</label>
        <input type="text" class="form-control" name="Username" required />
    </div>

    <!-- Password -->
    <div class="form-group">
        <label>Password</label>
        <input type="password" class="form-control" name="Password" required />
    </div>

    <!-- Role Dropdown -->
    <div class="form-group">
        <label>Role</label>
        <select class="form-control" name="Role" required>
            <option value="Admin">Admin</option>
            <option value="Manager">Manager</option>
            <option value="User">User</option>
        </select>
    </div>

    <br />
    <button type="submit" class="btn btn-primary">Login</button>
</form>
```
---
## 📌 3 Login Controller

```csharp
[HttpPost]
public async Task<IActionResult> Login(string Username, string Password, string Role)
{
    // ✅ Call AuthService for authentication
    var jsonData = await _authService.AuthenticateUserAsync(Username, Password, Role);

    // ❌ If authentication fails
    if (jsonData == null)
    {
        ViewBag.Error = "Invalid credentials.";
        return View();
    }

    // ✅ Deserialize JSON response
    var data = JsonConvert.DeserializeObject<Dictionary<string, dynamic>>(jsonData);
    string token = data["token"];

    // ❌ If token is empty, login failed
    if (string.IsNullOrEmpty(token))
    {
        ViewBag.Error = "Invalid credentials.";
        return View();
    }

    // ✅ Store token & role in session
    _httpContextAccessor.HttpContext.Session.SetString("JWTToken", token);
    _httpContextAccessor.HttpContext.Session.SetString("UserRole", Role);

    // ✅ Redirect to home/dashboard
    return RedirectToAction("Index", "Home");
}
```

---

## 📌 4 AuthService.cs

```csharp
// ✅ AuthService is responsible for making API calls for authentication
//    It sends Username, Password, and Role to the API and returns the JSON response.
public async Task<string?> AuthenticateUserAsync(string username, string password, string? role = null)
{
    // ✅ Prepare request body
    var requestData = new 
    { 
        UserName = username, 
        Password = password, 
        Role = role // ✅ Sending role if required
    };

    // ✅ Convert request data to JSON
    var content = new StringContent(JsonConvert.SerializeObject(requestData), Encoding.UTF8, "application/json");

    // ✅ Call API endpoint
    HttpResponseMessage response = await _httpClient.PostAsync("https://localhost:7093/api/User/login", content);

    // ✅ If success, return JSON response
    if (response.IsSuccessStatusCode)
    {
        return await response.Content.ReadAsStringAsync();
    }

    // ❌ Return null if login fails
    return null;
}
````

---

## Step 5: Add Custom Action Filter as below
Add CheckAccess class inside Services folder

```csharp
/// <summary>
/// Custom authorization filter to check if a user is authenticated based on JWT token stored in session.
/// If the token is missing, it redirects the user to the login page.
/// </summary>
public class CheckAccess : ActionFilterAttribute, IAuthorizationFilter
{
    public void OnAuthorization(AuthorizationFilterContext filterContext)
    {
        // If JWT token is not found in session, redirect to login page
        if (filterContext.HttpContext.Session.GetString("JWTToken") == null)
        {
            filterContext.Result = new RedirectResult("~/Login/Login");
        }
    }

    public override void OnResultExecuting(ResultExecutingContext context)
    {
        // If JWT token is not found, add headers to prevent browser from caching the previous page
        if (context.HttpContext.Session.GetString("JWTToken") == null)
        {
            context.HttpContext.Response.Headers["Cache-Control"] = "no-cache, no-store, must-revalidate";
            context.HttpContext.Response.Headers["Expires"] = "-1";
            context.HttpContext.Response.Headers["Pragma"] = "no-cache";
        }

        base.OnResultExecuting(context);
    }
}
```
---

## Step 6: Create CommonVariables class
Create CommonVariables class inside Services folder, to retrieve session data easily.
```csharp
public static class CommonVariables
{
    // Provides access to the current HttpContext (request-specific data like session, user, etc.)
    private static IHttpContextAccessor _httpContextAccessor;

    // Static constructor initializes the IHttpContextAccessor
    static CommonVariables()
    {
        _httpContextAccessor = new HttpContextAccessor();
    }

    /// <summary>
    /// Retrieves the JWT token from the session, if available.
    /// </summary>
    /// <returns>JWT token as string if exists, otherwise null.</returns>
    public static string? Token()
    {
        string? Token = null;

        // Check if the session contains the "JWTToken" key
        if (_httpContextAccessor.HttpContext?.Session.GetString("JWTToken") != null)
        {
            // If it exists, get the token value from session
            Token = _httpContextAccessor.HttpContext.Session.GetString("JWTToken");
        }

        return Token;
    }

    public static string? UserRole()
    {
        string? UserRole = null;

        // Check if the session contains the "UserRole" key
        if (_httpContextAccessor.HttpContext?.Session.GetString("UserRole") != null)
        {
            // If it exists, get the username value from session
            UserName = _httpContextAccessor.HttpContext.Session.GetString("UserRole");
        }

        return UserName;
    }
}
```

---

## 📌 7 UserController.cs

```csharp
using Address_Consume.Models;
using Microsoft.AspNetCore.Mvc;
using Newtonsoft.Json;
using System.Text;

namespace Address_Consume.Controllers
{
    public class UserController : Controller
    {
        private readonly HttpClient _client;

        // ✅ Constructor injection for HttpClient and IHttpContextAccessor
        public UserController(IHttpClientFactory httpClientFactory)
        {
            _client = httpClientFactory.CreateClient();
            _client.BaseAddress = new Uri("https://localhost:7093/api/");
        }

        [CheckAccess]
        // ✅ Displays list of users (only if token is valid)
        public async Task<IActionResult> Index()
        {
            try
            {
                var token = CommonVariables.Token();
                _client.DefaultRequestHeaders.Authorization = new System.Net.Http.Headers.AuthenticationHeaderValue("Bearer", token);
                var response = await _client.GetAsync("User");

                // 🔒 Redirect to login if unauthorized
                if (response.StatusCode == System.Net.HttpStatusCode.Unauthorized)
                {
                    return RedirectToAction("Login", "Login");
                }

                // ✅ Deserialize response into user list
                response.EnsureSuccessStatusCode();
                var json = await response.Content.ReadAsStringAsync();
                var list = JsonConvert.DeserializeObject<List<UserModel>>(json);

                return View(list);
            }
            catch
            {
                TempData["Error"] = "Unable to load users.";
                return View(new List<UserModel>());
            }
        }

        // ⚠️ Similarly, AddEdit and Delete methods should call AddJwtToken() before API requests
    }
}
```


---

## 📌 8 Role-Based UI Display (Example)

```csharp
// ✅ Get user role from session
var role = CommonVariables.UserRole();

if (role == "Admin")
{
    // Show admin dashboard
}
else if (role == "User")
{
    // Show user dashboard
}
```

### ✅ Razor View Example (Role-Based Navigation)

```csharp
@if (CommonVariables.UserRole() == "Admin")
{
    <a href="/User/Index">Manage Users</a>
}

@if (CommonVariables.UserRole() == "User")
{
    <a href="/Profile">My Profile</a>
}
```

---
