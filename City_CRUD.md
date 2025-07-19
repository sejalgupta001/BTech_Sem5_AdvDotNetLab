# 🏙️ City Management System - API & MVC Integration (ASP.NET Core)

## 📌 Overview
Key Features:
- ✅ Full CRUD operations for City
- ✅ Cascading dropdowns (Country → State)
- ✅ Filter cities by name/state/country/date
- ✅ Top 10 city listing
- ✅ Robust error handling

---

## 🧩 API Project (`Address_API`)

### ✅ Entity: `City.cs`
```csharp
public partial class City
{
    public int CityId { get; set; }
    public int StateId { get; set; }
    public int CountryId { get; set; }
    public string CityName { get; set; } = null!;
    public string? CityCode { get; set; }
    public DateTime CreatedDate { get; set; }
    public DateTime? ModifiedDate { get; set; }
    public string? CityImage { get; set; }

    [JsonIgnore]
    public virtual Country? Country { get; set; } = null!;
    
    [JsonIgnore]
    public virtual State? State { get; set; } = null!;
}
````

---

### ✅ API Controller: `CityController.cs`

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using Address_API.Models;

[Route("api/[controller]")]
[ApiController]
public class CityController : ControllerBase
{
    private readonly AddressDemoDbContext _context;

    public CityController(AddressDemoDbContext context)
    {
        _context = context;
    }

    // ✅ GET: All cities with related country & state info
    [HttpGet]
    public async Task<ActionResult> GetAll()
    {
        try
        {
            var cities = await _context.Cities
                .Include(c => c.State)
                .Include(c => c.Country)
                .Select(c => new
                {
                    c.CityId,
                    c.CityName,
                    c.StateId,
                    StateName = c.State.StateName,
                    c.CountryId,
                    CountryName = c.Country.CountryName,
                    c.CreatedDate,
                    c.ModifiedDate
                })
                .ToListAsync();

            return Ok(cities);
        }
        catch (Exception ex)
        {
            return StatusCode(500, $"Error retrieving cities: {ex.Message}");
        }
    }

    // ✅ GET: Top 10 cities
    [HttpGet("Top10")]
    public async Task<ActionResult> GetTop10()
    {
        try
        {
            var cities = await _context.Cities
                .Include(c => c.State)
                .Include(c => c.Country)
                .Select(c => new
                {
                    c.CityId,
                    c.CityName,
                    c.StateId,
                    StateName = c.State.StateName,
                    c.CountryId,
                    CountryName = c.Country.CountryName,
                    c.CreatedDate,
                    c.ModifiedDate
                })
                .Take(10)
                .ToListAsync();

            return Ok(cities);
        }
        catch (Exception ex)
        {
            return StatusCode(500, $"Error retrieving top cities: {ex.Message}");
        }
    }

    // ✅ GET: City by ID
    [HttpGet("{id}")]
    public async Task<ActionResult> GetById(int id)
    {
        try
        {
            var city = await _context.Cities
                .Include(c => c.State)
                .Include(c => c.Country)
                .Where(c => c.CityId == id)
                .Select(c => new
                {
                    c.CityId,
                    c.CityName,
                    c.StateId,
                    StateName = c.State.StateName,
                    c.CountryId,
                    CountryName = c.Country.CountryName,
                    c.CreatedDate,
                    c.ModifiedDate
                })
                .FirstOrDefaultAsync();

            if (city == null) return NotFound();

            return Ok(city);
        }
        catch (Exception ex)
        {
            return StatusCode(500, $"Error retrieving city: {ex.Message}");
        }
    }

    // ✅ GET: Filter cities by name/state/country/date
    [HttpGet("filter")]
    public async Task<ActionResult> Filter(
        [FromQuery] string? cityName,
        [FromQuery] string? stateName,
        [FromQuery] int? countryId,
        [FromQuery] DateTime? createdDate)
    {
        try
        {
            var query = _context.Cities
                .Include(c => c.State)
                .Include(c => c.Country)
                .AsQueryable();

            // Filter by city name (case-insensitive)
            if (!string.IsNullOrEmpty(cityName))
            {
                string cityLower = cityName.ToLower();
                query = query.Where(c => c.CityName.ToLower().Contains(cityLower));
            }

            // Filter by state name
            if (!string.IsNullOrEmpty(stateName))
            {
                string stateLower = stateName.ToLower();
                query = query.Where(c => c.State.StateName.ToLower().Contains(stateLower));
            }

            // Filter by country ID
            if (countryId.HasValue)
            {
                query = query.Where(c => c.CountryId == countryId.Value);
            }

            // Filter by created date (ignoring time)
            if (createdDate.HasValue)
            {
                query = query.Where(c => c.CreatedDate.Date == createdDate.Value.Date);
            }

            var result = await query
                .Select(c => new
                {
                    c.CityId,
                    c.CityName,
                    c.StateId,
                    StateName = c.State.StateName,
                    c.CountryId,
                    CountryName = c.Country.CountryName,
                    c.CreatedDate,
                    c.ModifiedDate
                })
                .ToListAsync();

            return Ok(result);
        }
        catch (Exception ex)
        {
            return StatusCode(500, $"Error filtering cities: {ex.Message}");
        }
    }

    // ✅ POST: Create a new city
    [HttpPost("cityInsert")]
    public async Task<ActionResult> Create(City city)
    {
        try
        {
            city.CreatedDate = DateTime.Now;
            _context.Cities.Add(city);
            await _context.SaveChangesAsync();

            // Returns 201 Created with location header
            return CreatedAtAction(nameof(GetById), new { id = city.CityId }, city);
        }
        catch (Exception ex)
        {
            return StatusCode(500, $"Error creating city: {ex.Message}");
        }
    }

    // ✅ PUT: Update an existing city
    [HttpPut("{id}")]
    public async Task<IActionResult> Update(int id, City city)
    {
        // Ensure route ID matches model ID
        if (id != city.CityId)
            return BadRequest("City ID mismatch.");

        try
        {
            city.ModifiedDate = DateTime.Now;

            // Track as modified
            _context.Entry(city).State = EntityState.Modified;
            await _context.SaveChangesAsync();

            return NoContent(); // 204 No Content
        }
        catch (Exception ex)
        {
            return StatusCode(500, $"Error updating city: {ex.Message}");
        }
    }

    // ✅ DELETE: Delete a city by ID
    [HttpDelete("{id}")]
    public async Task<IActionResult> Delete(int id)
    {
        try
        {
            var city = await _context.Cities.FindAsync(id);
            if (city == null)
                return NotFound();

            _context.Cities.Remove(city);
            await _context.SaveChangesAsync();

            return NoContent(); // 204 No Content
        }
        catch (Exception ex)
        {
            return StatusCode(500, $"Error deleting city: {ex.Message}");
        }
    }

    // ✅ GET: All countries for dropdown
    [HttpGet("dropdown/countries")]
    public async Task<ActionResult> GetCountries()
    {
        try
        {
            var countries = await _context.Countries
                .Select(c => new { c.CountryId, c.CountryName })
                .ToListAsync();

            return Ok(countries);
        }
        catch (Exception ex)
        {
            return StatusCode(500, $"Error retrieving countries: {ex.Message}");
        }
    }

    // ✅ GET: States by selected country (for cascading dropdown)
    [HttpGet("dropdown/states/{countryId}")]
    public async Task<ActionResult> GetStatesByCountry(int countryId)
    {
        try
        {
            var states = await _context.States
                .Where(s => s.CountryId == countryId)
                .Select(s => new { s.StateId, s.StateName })
                .ToListAsync();

            return Ok(states);
        }
        catch (Exception ex)
        {
            return StatusCode(500, $"Error retrieving states: {ex.Message}");
        }
    }
}
```

---

## 🎯 MVC Project (`Address_Consume`)

### ✅ Model: `CityModel.cs`

```csharp
public class CityModel
{
    public int CityId { get; set; }
    
    [Required]
    public string CityName { get; set; }

    [Required]
    public int? CountryId { get; set; }

    [Required]
    public int? StateId { get; set; }

    public string? CountryName { get; set; }
    public string? StateName { get; set; }
    public DateTime? CreatedDate { get; set; }
    public DateTime? ModifiedDate { get; set; }
}
```

---

### ✅ MVC Controller: `CityController.cs`
```csharp
// ✅ CityController.cs (MVC Project)
using Address_Consume.Models;
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.Rendering;
using Newtonsoft.Json;
using System.Text;

namespace Address_Consume.Controllers
{
    public class CityController : Controller
    {
        private readonly HttpClient _client;

        public CityController(IHttpClientFactory httpClientFactory)
        {
            _client = httpClientFactory.CreateClient();
            _client.BaseAddress = new Uri("https://localhost:7093/api/");
        }

        #region List Cities

        // ✅ Display all cities with country dropdown for filtering
        public async Task<IActionResult> Index()
        {
            try
            {
                var response = await _client.GetAsync("City");
                response.EnsureSuccessStatusCode();

                var json = await response.Content.ReadAsStringAsync();
                var cityList = JsonConvert.DeserializeObject<List<CityModel>>(json);

                // Get country list for dropdown filter
                var countryResponse = await _client.GetAsync("Country");
                countryResponse.EnsureSuccessStatusCode();

                var countryJson = await countryResponse.Content.ReadAsStringAsync();
                var countries = JsonConvert.DeserializeObject<List<CountryModel>>(countryJson);

                ViewBag.Countries = countries;

                return View(cityList);
            }
            catch (Exception ex)
            {
                TempData["Error"] = $"Unable to load cities: {ex.Message}";
                return View(new List<CityModel>());
            }
        }

        // ✅ Display top 10 cities
        public async Task<IActionResult> Top10()
        {
            try
            {
                var response = await _client.GetAsync("City/top10");
                response.EnsureSuccessStatusCode();

                var json = await response.Content.ReadAsStringAsync();
                var list = JsonConvert.DeserializeObject<List<CityModel>>(json);
                return View(list);
            }
            catch (Exception ex)
            {
                TempData["Error"] = $"Unable to load top 10 cities: {ex.Message}";
                return View(new List<CityModel>());
            }
        }

        #endregion

        #region Filter

        // ✅ Filter city list by name, state, country or created date
        public async Task<IActionResult> Filter(string? cityName, string? stateName, int? countryId, DateTime? createdDate)
        {
            try
            {
                string url = "City/filter?";
                if (!string.IsNullOrEmpty(cityName)) url += $"cityName={cityName}&";
                if (!string.IsNullOrEmpty(stateName)) url += $"stateName={stateName}&";
                if (countryId.HasValue) url += $"countryId={countryId}&";
                if (createdDate.HasValue) url += $"createdDate={createdDate:yyyy-MM-dd}";

                var response = await _client.GetAsync(url.TrimEnd('&'));
                response.EnsureSuccessStatusCode();

                var json = await response.Content.ReadAsStringAsync();
                var cityList = JsonConvert.DeserializeObject<List<CityModel>>(json);

                // Get country list again for dropdown
                var countryRes = await _client.GetAsync("Country");
                var countryJson = await countryRes.Content.ReadAsStringAsync();
                var countries = JsonConvert.DeserializeObject<List<CountryModel>>(countryJson);

                ViewBag.Countries = countries;

                return View("Index", cityList);
            }
            catch (Exception ex)
            {
                TempData["Error"] = $"Filter failed: {ex.Message}";
                return RedirectToAction("Index"); // ✅ Corrected typo: Inedx ➝ Index
            }
        }

        #endregion

        #region Add/Edit

        // ✅ GET: Display Add/Edit form
        public async Task<IActionResult> AddEdit(int? id)
        {
            try
            {
                CityModel city = new CityModel();
                await LoadCountries();

                if (id != null)
                {
                    var response = await _client.GetAsync($"City/{id}");
                    if (!response.IsSuccessStatusCode)
                    {
                        TempData["Error"] = "City not found.";
                        return RedirectToAction("Index");
                    }

                    var json = await response.Content.ReadAsStringAsync();
                    city = JsonConvert.DeserializeObject<CityModel>(json);

                    await LoadStates(city.CountryId ?? 0, city.StateId);
                }
                else
                {
                    ViewBag.States = new SelectList(new List<StateModel>(), "StateId", "StateName");
                }

                return View(city);
            }
            catch (Exception ex)
            {
                TempData["Error"] = $"Unable to load city form: {ex.Message}";
                return RedirectToAction("Index");
            }
        }

        // ✅ POST: Save new or updated city
        [HttpPost]
        public async Task<IActionResult> AddEdit(CityModel model)
        {
            if (!ModelState.IsValid)
            {
                await LoadCountries(model.CountryId);
                await LoadStates(model.CountryId ?? 0, model.StateId);
                return View(model);
            }

            // Set timestamps
            if (model.CityId == 0)
                model.CreatedDate = DateTime.UtcNow;
            else
                model.ModifiedDate = DateTime.UtcNow;

            try
            {
                var json = JsonConvert.SerializeObject(model);
                var content = new StringContent(json, Encoding.UTF8, "application/json");

                HttpResponseMessage response;

                if (model.CityId == 0)
                {
                    response = await _client.PostAsync("City/cityInsert", content);
                }
                else
                {
                    response = await _client.PutAsync($"City/{model.CityId}", content);
                }

                if (!response.IsSuccessStatusCode)
                {
                    var error = await response.Content.ReadAsStringAsync();
                    TempData["Error"] = $"API call failed: {response.StatusCode} - {error}";
                    await LoadCountries(model.CountryId);
                    await LoadStates(model.CountryId ?? 0, model.StateId);
                    return View(model);
                }

                return RedirectToAction("Index");
            }
            catch (Exception ex)
            {
                TempData["Error"] = $"Unable to save city: {ex.Message}";
                await LoadCountries(model.CountryId);
                await LoadStates(model.CountryId ?? 0, model.StateId);
                return View(model);
            }
        }

        #endregion

        #region Delete

        // ✅ Delete city by ID
        public async Task<IActionResult> Delete(int id)
        {
            try
            {
                var response = await _client.DeleteAsync($"City/{id}");
                response.EnsureSuccessStatusCode();
            }
            catch (Exception ex)
            {
                TempData["Error"] = $"Unable to delete city: {ex.Message}";
            }
            return RedirectToAction("Index");
        }

        #endregion

        #region Cascading Dropdown - AJAX

        // ✅ Get states for dropdown by countryId
        public async Task<JsonResult> GetStates(int countryId)
        {
            var response = await _client.GetStringAsync($"City/dropdown/states/{countryId}");
            var states = JsonConvert.DeserializeObject<List<StateModel>>(response);
            return Json(states);
        }

        #endregion

        #region Helpers

        // ✅ Populate countries dropdown
        private async Task LoadCountries(int? selectedCountryId = null)
        {
            var response = await _client.GetStringAsync("City/dropdown/countries");
            var countries = JsonConvert.DeserializeObject<List<CountryModel>>(response);
            ViewBag.Countries = new SelectList(countries, "CountryId", "CountryName", selectedCountryId);
        }

        // ✅ Populate states dropdown based on selected country
        private async Task LoadStates(int countryId, int? selectedStateId = null)
        {
            var response = await _client.GetStringAsync($"City/dropdown/states/{countryId}");
            var states = JsonConvert.DeserializeObject<List<StateModel>>(response);
            ViewBag.States = new SelectList(states, "StateId", "StateName", selectedStateId);
        }

        #endregion
    }
}
```


## 🖼️ Views

### ✔️ Add/Edit City Form (`AddEdit.cshtml`)

```html
@model CityModel
@{
    ViewData["Title"] = Model.CityId == 0 ? "Add City" : "Edit City";
}

<div class="container mt-4">
    <h2>@ViewData["Title"]</h2>
    <h3 class="text-danger">@TempData["Error"]</h3>

    <form asp-action="AddEdit" method="post">
        @Html.ValidationSummary(true, "", new { @class = "text-danger" })

        <input type="hidden" asp-for="CityId" />
        <input type="hidden" asp-for="CreatedDate" />
        <input type="hidden" asp-for="ModifiedDate" />

        <div class="mb-3">
            <label asp-for="CityName" class="form-label"></label>
            <input asp-for="CityName" class="form-control" />
            <span asp-validation-for="CityName" class="text-danger"></span>
        </div>

        <div class="mb-3">
            <label asp-for="CountryId" class="form-label"></label>
            <select asp-for="CountryId" asp-items="ViewBag.Countries" class="form-control" id="CountryId" onchange="loadStates(this.value)">
                <option value="">-- Select Country --</option>
            </select>
            <span asp-validation-for="CountryId" class="text-danger"></span>
        </div>

        <div class="mb-3">
            <label asp-for="StateId" class="form-label"></label>
            <select asp-for="StateId" asp-items="ViewBag.States" class="form-control" id="StateId">
                <option value="">-- Select State --</option>
            </select>
            <span asp-validation-for="StateId" class="text-danger"></span>
        </div>

        <button type="submit" class="btn btn-primary">@((Model.CityId == 0) ? "Add" : "Update")</button>
        <a asp-action="Index" class="btn btn-secondary">Back</a>
    </form>
</div>

@section Scripts {
    <partial name="_ValidationScriptsPartial" />
    <script>
        function loadStates(countryId) {
            fetch(`/City/GetStates?countryId=${countryId}`)
                .then(response => response.json())
                .then(data => {
                    const stateDropdown = document.getElementById('StateId');
                    stateDropdown.innerHTML = '<option value="">-- Select State --</option>';
                    data.forEach(state => {
                        const option = document.createElement('option');
                        option.value = state.stateId;
                        option.text = state.stateName;
                        stateDropdown.appendChild(option);
                    });
                });
        }
    </script>
}

```

### ✔️ City Listing (`Index.cshtml`)

```html
@model IEnumerable<CityModel>
@{
    ViewData["Title"] = "Cities";
}
@{
    var countries = ViewBag.Countries as List<Address_Consume.Models.CountryModel> ?? new();
}

<h2>Cities</h2>
<a asp-action="AddEdit" class="btn btn-success">Create New</a>
<a asp-action="Top10" class="btn btn-secondary">Top 10</a>

<form asp-action="Filter" method="get" class="row g-3 mt-1 mb-3">
    <div class="col-md-2">
        <input type="text" name="cityName" class="form-control" placeholder="City Name" />
    </div>
    <div class="col-md-2">
        <input type="text" name="stateName" class="form-control" placeholder="State Name" />
    </div>
    <div class="col-md-2">
        <select name="countryId" class="form-control">
            <option value="">-- Select Country --</option>
            @foreach (var country in countries)
            {
                <option value="@country.CountryId">@country.CountryName</option>
            }
        </select>
    </div>
    <div class="col-md-2">
        <input type="date" name="createdDate" class="form-control" />
    </div>
    <div class="col-md-2">
        <button type="submit" class="btn btn-primary">Search</button>
        <a href="@Url.Action("Index")" class="btn btn-secondary">Reset</a>
    </div>
</form>


<table class="table table-bordered">
    <thead>
        <tr>
            <th>Name</th>
            <th>Country</th>
            <th>State</th>
            <th>Actions</th>
        </tr>
    </thead>
    <tbody>
        @foreach (var city in Model)
        {
            <tr>
                <td>@city.CityName</td>
                <td>@city.CountryName</td>
                <td>@city.StateName</td>
                <td>
                    <a class="btn btn-sm btn-primary" asp-action="AddEdit" asp-route-id="@city.CityId">Edit</a> |
                    <a class="btn btn-sm btn-danger" asp-action="Delete" asp-route-id="@city.CityId" onclick="return confirm('Are you sure you want to delete this city?');">Delete</a>
                </td>
            </tr>
        }
    </tbody>
</table>


```
### ✔️ Top 10 Cities (`Top10.cshtml`)

```html
@model List<CityModel>

@{
    ViewData["Title"] = "Top 10 Cities";
    int count = 1;
}

<div class="container mt-4">
    <h2>@ViewData["Title"]</h2>

    @if (TempData["Error"] != null)
    {
        <div class="alert alert-danger">@TempData["Error"]</div>
    }

    <div class="mb-3">
        <a asp-action="Index" class="btn btn-secondary">Back to City List</a>
    </div>


    <table class="table table-bordered">
        <thead>
            <tr>
                <th>Sr. NO.</th>
                <th>Name</th>
                <th>Country</th>
                <th>State</th>
                <th>Actions</th>
            </tr>
        </thead>
        <tbody>
            @foreach (var city in Model)
            {
                <tr>
                    <td>@count</td>
                    <td>@city.CityName</td>
                    <td>@city.CountryName</td>
                    <td>@city.StateName</td>
                    <td>
                        <a class="btn btn-sm btn-primary" asp-action="AddEdit" asp-route-id="@city.CityId">Edit</a> |
                        <a class="btn btn-sm btn-danger" asp-action="Delete" asp-route-id="@city.CityId" onclick="return confirm('Are you sure you want to delete this city?');">Delete</a>
                    </td>
                </tr>
                count++;
            }
        </tbody>
    </table>
</div>
```
---

