# 🌐 State Management in ASP.NET Core API 

## 🧑‍💼 Controller

### `StateController.cs`
```csharp
using Address_API.Helpers;
using Address_API.Models;
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;

namespace Address_API.Controllers
{
    [Route("api/[controller]")]
    [ApiController]
    public class StateController : ControllerBase
    {
        private readonly AddressDemoDbContext _context;

        public StateController(AddressDemoDbContext context)
        {
            _context = context;
        }

        // GET: api/State
        [HttpGet]
        public async Task<ActionResult<IEnumerable<State>>> GetAll()
        {
            var states = await _context.States
            .Select(s => new
             {
                 s.StateId,
                 s.CountryId,
                 s.StateName,
                 s.StateCode,
                 s.CreatedDate,
                 s.ModifiedDate,
                 s.FilePath,
                 CountryName = s.Country.CountryName
             })
            .ToListAsync();

            return Ok(states);
        }

        // GET: api/State/Top10
        [HttpGet("Top10")]
        public async Task<ActionResult<IEnumerable<State>>> GetTop10()
        {
            var topStates = await _context.States
                .Include(s => s.Country)
                .Take(10)
                .ToListAsync();

            return Ok(topStates);
        }

        // GET: api/State/5
        [HttpGet("{id}")]
        public async Task<ActionResult<State>> GetById(int id)
        {
            var state = await _context.States.FindAsync(id);

            return state == null ? NotFound() : Ok(state);
        }

        // GET: api/State/byCountry/3
        [HttpGet("byCountry/{countryId}")]
        public async Task<ActionResult<IEnumerable<object>>> GetByCountry(int countryId)
        {
            var states = await _context.States
                .Where(s => s.CountryId == countryId)
                .Select(s => new { s.StateId, s.StateName })
                .ToListAsync();

            return Ok(states);
        }

        // GET: api/State/dropdown
        [HttpGet("dropdown")]
        public async Task<ActionResult<IEnumerable<object>>> GetDropdown()
        {
            var countries = await _context.Countries
                .Select(c => new { c.CountryId, c.CountryName })
                .ToListAsync();

            return Ok(countries);
        }

        // POST: api/State
        [HttpPost]
        public async Task<IActionResult> Create([FromBody] State state)
        {
            state.CreatedDate = DateTime.Now;
            _context.States.Add(state);
            await _context.SaveChangesAsync();

            return CreatedAtAction(nameof(GetById), new { id = state.StateId }, state);
        }

        // PUT: api/State/5
        [HttpPut("{id}")]
        public async Task<IActionResult> Update(int id, [FromBody] State state)
        {
            if (id != state.StateId)
                return BadRequest();

            _context.Entry(state).State = EntityState.Modified;
            state.ModifiedDate = DateTime.Now;

            await _context.SaveChangesAsync();
            return NoContent();
        }

        // DELETE: api/State/5
        [HttpDelete("{id}")]
        public async Task<IActionResult> Delete(int id)
        {
            var state = await _context.States.FindAsync(id);
            if (state == null)
                return NotFound();

            _context.States.Remove(state);
            await _context.SaveChangesAsync();
            return NoContent();
        }
    }
}
```

# 🌐 State Management in ASP.NET Core MVC (Consume Web API)

## 📁 Models

### `StateModel.cs`
```csharp
using System.ComponentModel.DataAnnotations;

namespace Address_Consume.Models
{
    public class StateModel
    {
        public int StateId { get; set; }

        [Display(Name = "Country")]
        [Required(ErrorMessage = "Please select a country")]
        public int? CountryId { get; set; }

        [Display(Name = "State Name")]
        [Required(ErrorMessage = "State name is required")]
        [StringLength(100, ErrorMessage = "Maximum 100 characters allowed")]
        public string StateName { get; set; }

        [Display(Name = "State Code")]
        [StringLength(10, ErrorMessage = "Maximum 10 characters allowed")]
        public string? StateCode { get; set; }

        public DateTime CreatedDate { get; set; }
        public DateTime? ModifiedDate { get; set; }

        public List<CountryDropDownModel>? CountryList { get; set; }
        public string? CountryName { get; set; }
    }

    public class CountryDropDownModel
    {
        public int CountryId { get; set; }

        [Display(Name = "Country")]
        public string CountryName { get; set; } = null!;
    }
}
```

## 🧑‍💼 Controller

### `StateController.cs`
```csharp
using Address_Consume.Models;
using Microsoft.AspNetCore.Mvc;
using Newtonsoft.Json;
using System.Text;

namespace Address_Consume.Controllers
{
    public class StateController : Controller
    {
        private readonly HttpClient _client;

        public StateController(IHttpClientFactory httpClientFactory)
        {
            _client = httpClientFactory.CreateClient();
            _client.BaseAddress = new Uri("https://localhost:7093/api/");
        }

        public async Task<IActionResult> Index()
        {
            var response = await _client.GetAsync("State");
            var json = await response.Content.ReadAsStringAsync();
            var list = JsonConvert.DeserializeObject<List<StateModel>>(json);
            return View(list);
        }

        public async Task<IActionResult> Delete(int id) 
        {
            await _client.DeleteAsync($"State/{id}");
            return RedirectToAction("Index");
        }

        public async Task<IActionResult> AddEdit(int? id)
        {
            var countryResponse = await _client.GetAsync("State/dropdown"); 
            var countryJson = await countryResponse.Content.ReadAsStringAsync();
            var countries = JsonConvert.DeserializeObject<List<CountryDropDownModel>>(countryJson);

            StateModel state;

            if (id == null)
            {
                state = new StateModel();
            }
            else
            {
                var response = await _client.GetAsync($"State/{id}");
                if (!response.IsSuccessStatusCode) return NotFound();

                var json = await response.Content.ReadAsStringAsync();
                state = JsonConvert.DeserializeObject<StateModel>(json);
            }

            state.CountryList = countries;
            return View(state);
        }

        [HttpPost]
        public async Task<IActionResult> AddEdit(StateModel state)
        {
            if (!ModelState.IsValid)
            {
                var countryResponse = await _client.GetAsync("State/dropdown");
                var countryJson = await countryResponse.Content.ReadAsStringAsync();
                state.CountryList = JsonConvert.DeserializeObject<List<CountryDropDownModel>>(countryJson);

                return View(state);
            }

            var content = new StringContent(JsonConvert.SerializeObject(state), Encoding.UTF8, "application/json");

            if (state.StateId == 0)
            {
                state.CreatedDate = DateTime.Now;
                await _client.PostAsync("State", content);
            }
            else
            {
                state.ModifiedDate = DateTime.Now;
                await _client.PutAsync($"State/{state.StateId}", content);
            }

            return RedirectToAction("Index");
        }
    }
}
```

## 💻 Razor View

### `AddEdit.cshtml`
```html
@model StateModel
@{
    ViewData["Title"] = Model.StateId == 0 ? "Add State" : "Edit State";
}

<div class="container mt-4">
    <h2>@ViewData["Title"]</h2>

    <form asp-action="AddEdit" method="post">
        @Html.ValidationSummary(true, "", new { @class = "text-danger" })
        <input type="hidden" asp-for="StateId" />
        <input type="hidden" asp-for="CreatedDate" />

        <div class="mb-3">
            <label asp-for="CountryId" class="form-label"></label>
            <select asp-for="CountryId" class="form-select" asp-items="@(new SelectList(Model.CountryList, "CountryId", "CountryName"))">
                <option value="">-- Select Country --</option>
            </select>
            <span asp-validation-for="CountryId" class="text-danger"></span>
        </div>

        <div class="mb-3">
            <label asp-for="StateName" class="form-label"></label>
            <input asp-for="StateName" class="form-control" />
            <span asp-validation-for="StateName" class="text-danger"></span>
        </div>

        <div class="mb-3">
            <label asp-for="StateCode" class="form-label"></label>
            <input asp-for="StateCode" class="form-control" />
            <span asp-validation-for="StateCode" class="text-danger"></span>
        </div>

        <button type="submit" class="btn btn-primary">@((Model.StateId == 0) ? "Add" : "Update")</button>
        <a asp-action="Index" class="btn btn-secondary">Back</a>
    </form>
</div>
```

## ⚙️ Program Configuration

### `Program.cs`
```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllersWithViews();
builder.Services.AddHttpClient();// ? Registers IHttpClientFactory

var app = builder.Build();

if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler("/Home/Error");
    app.UseHsts();
}

app.UseHttpsRedirection();
app.UseStaticFiles();
app.UseRouting();
app.UseAuthorization();

app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Home}/{action=Index}/{id?}");

app.Run();
```
