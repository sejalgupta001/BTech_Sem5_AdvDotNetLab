# 👤 User Management – ASP.NET Core Web API + MVC

## 📦 Table: `User`

| Column           | Type       | Description                   |
| ---------------- | ---------- | ----------------------------- |
| UserId           | `int`      | Primary key (identity)        |
| UserName         | `string`   | Required                      |
| Email            | `string`   | Required, must be valid email |
| Password         | `string`   | Required                      |
| CreationDate     | `DateTime` | Required – user creation time |
| ModificationDate | `DateTime` | Required – last updated time  |

---

## 🧑‍💼 ASP.NET Core Web API

### 📁 `UserController.cs`

```csharp
using Address_API.Models;
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;

namespace Address_API.Controllers
{
    [Route("api/[controller]")]
    [ApiController]
    public class UserController : ControllerBase
    {
        private readonly AddressDemoDbContext _context;

        public UserController(AddressDemoDbContext context)
        {
            _context = context;
        }

        // GET: api/User
        [HttpGet]
        public async Task<ActionResult<IEnumerable<User>>> GetAll()
        {
            try
            {
                return Ok(await _context.Users.ToListAsync());
            }
            catch (Exception ex)
            {
                return StatusCode(500, ex.Message);
            }
        }

        // GET: api/User/5
        [HttpGet("{id}")]
        public async Task<ActionResult<User>> GetById(int id)
        {
            try
            {
                var user = await _context.Users.FindAsync(id);
                return user == null ? NotFound() : Ok(user);
            }
            catch (Exception ex)
            {
                return StatusCode(500, ex.Message);
            }
        }

        // POST: api/User
        [HttpPost]
        public async Task<IActionResult> Create([FromBody] User user)
        {
            try
            {
                user.CreationDate = DateTime.Now;
                user.ModificationDate = DateTime.Now;
                _context.Users.Add(user);
                await _context.SaveChangesAsync();
                return CreatedAtAction(nameof(GetById), new { id = user.UserId }, user);
            }
            catch (Exception ex)
            {
                return StatusCode(500, ex.Message);
            }
        }

        // PUT: api/User/5
        [HttpPut("{id}")]
        public async Task<IActionResult> Update(int id, [FromBody] User user)
        {
            try
            {
                if (id != user.UserId)
                    return BadRequest();
                user.ModificationDate = DateTime.Now;
                _context.Entry(user).State = EntityState.Modified;
                await _context.SaveChangesAsync();
                return NoContent();
            }
            catch (Exception ex)
            {
                return StatusCode(500, ex.Message);
            }
        }

        // DELETE: api/User/5
        [HttpDelete("{id}")]
        public async Task<IActionResult> Delete(int id)
        {
            try
            {
                var user = await _context.Users.FindAsync(id);
                if (user == null) return NotFound();

                _context.Users.Remove(user);
                await _context.SaveChangesAsync();
                return NoContent();
            }
            catch (Exception ex)
            {
                return StatusCode(500, ex.Message);
            }
        }
    }
}
```

---

## 🧩 ASP.NET Core MVC – Consuming User API

### ✅ `UserModel.cs`

```csharp
using System.ComponentModel.DataAnnotations;

namespace Address_Consume.Models
{
    public class UserModel
    {
        public int UserId { get; set; }

        [Required(ErrorMessage = "Username is required")]
        public string UserName { get; set; }

        [Required(ErrorMessage = "Email is required")]
        [EmailAddress(ErrorMessage = "Invalid Email")]
        public string Email { get; set; }

        [Required(ErrorMessage = "Password is required")]
        public string Password { get; set; }
        public DateTime CreationDate { get; set; }
        public DateTime ModificationDate { get; set; }
    }
}
```

---

### ✅ `UserController.cs` (MVC)

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
        private readonly ILogger<UserController> _logger;

        public UserController(IHttpClientFactory httpClientFactory, ILogger<UserController> logger)
        {
            _client = httpClientFactory.CreateClient();
            _client.BaseAddress = new Uri("https://localhost:7093/api/");
            _logger = logger;
        }

        public async Task<IActionResult> Index()
        {
            try
            {
                var response = await _client.GetAsync("User");
                response.EnsureSuccessStatusCode();

                var json = await response.Content.ReadAsStringAsync();
                var list = JsonConvert.DeserializeObject<List<UserModel>>(json);
                return View(list);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error occurred while fetching users.");
                TempData["Error"] = "Unable to load users.";
                return View(new List<UserModel>());
            }
        }

        public async Task<IActionResult> Delete(int id)
        {
            try
            {
                var response = await _client.DeleteAsync($"User/{id}");
                response.EnsureSuccessStatusCode();
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, $"Error occurred while deleting user with ID {id}.");
                TempData["Error"] = "Unable to delete user.";
            }

            return RedirectToAction("Index");
        }

        public async Task<IActionResult> AddEdit(int? id)
        {
            try
            {
                UserModel user = new UserModel();

                if (id != null)
                {
                    var response = await _client.GetAsync($"User/{id}");
                    if (!response.IsSuccessStatusCode)
                    {
                        TempData["Error"] = "User not found.";
                        return RedirectToAction("Index");
                    }

                    var json = await response.Content.ReadAsStringAsync();
                    user = JsonConvert.DeserializeObject<UserModel>(json);
                }

                return View(user);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, $"Error occurred while loading form for User ID {id}.");
                TempData["Error"] = "Unable to load form.";
                return RedirectToAction("Index");
            }
        }

        [HttpPost]
        public async Task<IActionResult> AddEdit(UserModel user)
        {
            if (!ModelState.IsValid)
                return View(user);

            try
            {
                var content = new StringContent(JsonConvert.SerializeObject(user), Encoding.UTF8, "application/json");

                if (user.UserId == 0)
                {
                    var response = await _client.PostAsync("User", content);
                    response.EnsureSuccessStatusCode();
                }
                else
                {
                    var response = await _client.PutAsync($"User/{user.UserId}", content);
                    response.EnsureSuccessStatusCode();
                }

                return RedirectToAction("Index");
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, $"Error occurred while saving user with ID {user.UserId}.");
                TempData["Error"] = "Unable to save user.";
                return View(user);
            }
        }
    }
}
```

---

### ✅ Razor View: `AddEdit.cshtml`

```html
@model UserModel
@{
    ViewData["Title"] = Model.UserId == 0 ? "Add User" : "Edit User";
}

<div class="container mt-4">
    <h2>@ViewData["Title"]</h2>
    <h3 class="text-danger">@TempData["Error"]</h3>
    <form asp-action="AddEdit" method="post">
        @Html.ValidationSummary(true, "", new { @class = "text-danger" })
        <input type="hidden" asp-for="UserId" />
        <input type="hidden" asp-for="CreationDate" />

        <div class="mb-3">
            <label asp-for="UserName" class="form-label"></label>
            <input asp-for="UserName" class="form-control" />
            <span asp-validation-for="UserName" class="text-danger"></span>
        </div>

        <div class="mb-3">
            <label asp-for="Email" class="form-label"></label>
            <input asp-for="Email" class="form-control" />
            <span asp-validation-for="Email" class="text-danger"></span>
        </div>

        <div class="mb-3">
            <label asp-for="Password" class="form-label"></label>
            <input asp-for="Password" class="form-control" type="password" />
            <span asp-validation-for="Password" class="text-danger"></span>
        </div>

        <button type="submit" class="btn btn-primary">@((Model.UserId == 0) ? "Add" : "Update")</button>
        <a asp-action="Index" class="btn btn-secondary">Back</a>
    </form>
</div>
@section Scripts {
    @{
        await Html.RenderPartialAsync("_ValidationScriptsPartial");
    }
}

```

---

### ✅ Razor View: `Index.cshtml`

```html
@model List<UserModel>
@{
    ViewData["Title"] = "User List";
}

<div class="container mt-4">
    <h2>User List</h2>
    <a asp-action="AddEdit" class="btn btn-success mb-3">Add New User</a>
    <h3 class="text-danger">@TempData["Error"]</h3>
    <table class="table table-bordered">
        <thead>
            <tr>
                <th>UserName</th>
                <th>Email</th>
                <th>Password</th>
                <th>Actions</th>
            </tr>
        </thead>
        <tbody>
        @foreach (var user in Model)
        {
            <tr>
                <td>@user.UserName</td>
                <td>@user.Email</td>
                <td>@user.Password</td>
                <td>
                    <a asp-action="AddEdit" asp-route-id="@user.UserId" class="btn btn-warning btn-sm">Edit</a>
                    <a asp-action="Delete" asp-route-id="@user.UserId" class="btn btn-danger btn-sm" onclick="return confirm('Are you sure?')">Delete</a>
                </td>
            </tr>
        }
        </tbody>
    </table>
</div>
```

---

## ✅ Program.cs Configuration (MVC)

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllersWithViews();
builder.Services.AddHttpClient();

var app = builder.Build();

app.UseHttpsRedirection();
app.UseStaticFiles();
app.UseRouting();
app.UseAuthorization();

app.MapControllerRoute(
    name: "default",
    pattern: "{controller=User}/{action=Index}/{id?}");

app.Run();
```
