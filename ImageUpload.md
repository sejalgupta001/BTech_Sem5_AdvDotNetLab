# Image Upload Integration in ASP.NET Core Web API + MVC

## Step 1: Create a Helper to Handle Image Operations

```csharp
namespace Address_API.Helpers
{
    public static class ImageHelper
    {
        public static string SaveImageToFile(IFormFile imageFile, string directoryPath)
        {
            string directory = $"wwwroot/{directoryPath}";
            if (imageFile == null || imageFile.Length == 0)
                return null;

            if (!Directory.Exists(directory))
                Directory.CreateDirectory(directory);

            string fileExtension = Path.GetExtension(imageFile.FileName);
            if (string.IsNullOrWhiteSpace(fileExtension))
                fileExtension = ".jpeg";

            string fileName = $"{Guid.NewGuid()}{fileExtension}";
            string fullPath = $"{directoryPath}/{fileName}";
            string fullPathToWrite = $"{directory}/{fileName}";

            using (var stream = new FileStream(fullPathToWrite, FileMode.Create))
            {
                imageFile.CopyTo(stream);
            }

            return fullPath;
        }

        public static byte[] ReadFileBytes(string filePath)
        {
            if (string.IsNullOrWhiteSpace(filePath) || !File.Exists(filePath))
                return null;

            return File.ReadAllBytes(filePath);
        }

        public static string DeleteFileFromUrl(string fileUrl)
        {
            if (string.IsNullOrWhiteSpace(fileUrl))
                return "File URL is required.";

            var uri = new Uri(fileUrl);
            var fileName = Path.GetFileName(uri.LocalPath);
            var filePath = Path.Combine(Directory.GetCurrentDirectory(), "wwwroot", "Images", fileName);

            if (!System.IO.File.Exists(filePath))
                return "File not found.";

            System.IO.File.Delete(filePath);
            return "File deleted successfully.";
        }
    }
}
```

## Step 2: Update the State Model to Include Image Upload Properties

```csharp
public class State
{
    public int StateId { get; set; }
    public int CountryId { get; set; }
    public string StateName { get; set; } = null!;
    public string? StateCode { get; set; }
    [BindNever]
    public DateTime CreatedDate { get; set; }
    [BindNever]
    public DateTime? ModifiedDate { get; set; }
    [BindNever]
    public string? FilePath { get; set; }
    [NotMapped]
    public IFormFile? File { get; set; }
    [BindNever]
    public virtual ICollection<City> Cities { get; set; } = new List<City>();
    [BindNever]
    [JsonIgnore]
    public virtual Country? Country { get; set; } = null!;
}
```

## Step 3: Add Multipart Support in Web API Controller

### Create (POST)

```csharp
[HttpPost]
public async Task<IActionResult> CreateState([FromForm] State state)
{
    if (!ModelState.IsValid)
        return BadRequest();

    if (state.File != null)
    {
        var relativePath = ImageHelper.SaveImageToFile(state.File, "Images/States");
        state.FilePath = relativePath;
    }

    state.CreatedDate = DateTime.Now;

    _context.States.Add(state);
    await _context.SaveChangesAsync();

    return Ok(state);
}
```

### Update (PUT)

```csharp
[HttpPut("{id}")]
public async Task<IActionResult> UpdateState(int id, [FromForm] State updatedState)
{
    var existingState = await _context.States.FindAsync(id);
    if (existingState == null)
        return NotFound("State not found");

    existingState.StateName = updatedState.StateName;
    existingState.StateCode = updatedState.StateCode;
    existingState.CountryId = updatedState.CountryId;
    existingState.ModifiedDate = DateTime.Now;

    if (updatedState.File != null)
    {
        if (!string.IsNullOrEmpty(existingState.FilePath))
        {
            string fullPath = Path.Combine(Directory.GetCurrentDirectory(), "wwwroot", existingState.FilePath);
            if (System.IO.File.Exists(fullPath))
                System.IO.File.Delete(fullPath);
        }

        var relativePath = ImageHelper.SaveImageToFile(updatedState.File, "Images/States");
        existingState.FilePath = relativePath;
    }

    await _context.SaveChangesAsync();
    return Ok(existingState);
}
```

## Step 4: Delete Image When Deleting State

```csharp
[HttpDelete("{id}")]
public async Task<IActionResult> DeleteState(int id)
{
    var state = await _context.States.FindAsync(id);
    if (state == null)
        return NotFound();

    if (!string.IsNullOrEmpty(state.FilePath))
    {
        string fullPath = Path.Combine(Directory.GetCurrentDirectory(), "wwwroot", state.FilePath);
        if (System.IO.File.Exists(fullPath))
            System.IO.File.Delete(fullPath);
    }

    _context.States.Remove(state);
    await _context.SaveChangesAsync();

    return Ok("State deleted");
}
```

## Step 5: Add Upload Handling in MVC Client

```csharp
[HttpPost]
public async Task<IActionResult> AddEdit(StateViewModel state)
{
    if (!ModelState.IsValid)
    {
        var countryResponse = await _client.GetAsync("State/dropdown");
        var countryJson = await countryResponse.Content.ReadAsStringAsync();
        state.CountryList = JsonConvert.DeserializeObject<List<CountryViewModel>>(countryJson);
        return View(state);
    }

    using var formData = new MultipartFormDataContent();
    formData.Add(new StringContent(state.StateId.ToString()), "StateId");
    formData.Add(new StringContent(state.CountryId.ToString()), "CountryId");
    formData.Add(new StringContent(state.StateName ?? ""), "StateName");
    formData.Add(new StringContent(state.StateCode ?? ""), "StateCode");
    formData.Add(new StringContent(state.CreatedDate.ToString("o")), "CreatedDate");
    formData.Add(new StringContent(state.ModifiedDate?.ToString("o") ?? ""), "ModifiedDate");

    if (state.File != null && state.File.Length > 0)
    {
        var fileContent = new StreamContent(state.File.OpenReadStream());
        fileContent.Headers.ContentType = new System.Net.Http.Headers.MediaTypeHeaderValue(state.File.ContentType);
        formData.Add(fileContent, "File", state.File.FileName);
    }

    if (state.StateId == 0)
    {
        state.CreatedDate = DateTime.Now;
        await _client.PostAsync("State", formData);
    }
    else
    {
        state.ModifiedDate = DateTime.Now;
        await _client.PutAsync($"State/{state.StateId}", formData);
    }

    return RedirectToAction("Index");
}
```
## Step 6: StateList view page
```html
@model List<StateViewModel>
@{
    ViewData["Title"] = "State List";
}

<div class="container mt-4">
    <div class="d-flex justify-content-between align-items-center mb-3">
        <h2 class="mb-0">State List</h2>
        <a class="btn btn-success" href="/State/AddEdit">Add New State</a>
    </div>

    <table class="table table-bordered table-striped table-hover">
        <thead class="table-dark">
            <tr>
                <th>Image</th>
                <th>State ID</th>
                <th>State Name</th>
                <th>State Code</th>
                <th>Country ID</th>
                <th>Country Name</th>
                <th>Created Date</th>
                <th style="width: 150px;">Actions</th>
            </tr>
        </thead>
        <tbody>
            @if (Model != null && Model.Any())
            {
                foreach (var item in Model)
                {
                    <tr>
                        <td>@item.StateId</td>
                       
                        <td>
                            @if (!string.IsNullOrEmpty(item.FilePath))
                            {
                                <img src="https://localhost:7093/@item.FilePath" alt="State Image" width="80" class="img-thumbnail" />
                            }
                            else
                            {
                                <span class="text-muted">No Image</span>
                            }
                        </td>

                        <td>@item.StateName</td>
                        <td>@item.StateCode</td>
                        <td>@item.CountryId</td>
                        <td>@item.CountryName</td>
                        <td>@item.CreatedDate.ToString("yyyy-MM-dd")</td>
                        <td>@item.ModifiedDate?.ToString("yyyy-MM-dd")</td>
                        <td>
                            <a href="/State/AddEdit/@item.StateId" class="btn btn-sm btn-primary">Edit</a>
                            <a href="/State/Delete/@item.StateId" class="btn btn-sm btn-danger"
                               onclick="return confirm('Are you sure you want to delete this state?');">Delete</a>
                        </td>
                    </tr>
                }
            }
            else
            {
                <tr>
                    <td colspan="7" class="text-center text-muted">No states found.</td>
                </tr>
            }
        </tbody>
    </table>
</div>
```

## Step 7: Add Edit State Form
```html
@model StateViewModel

@{
    ViewData["Title"] = Model.StateId == 0 ? "Add State" : "Edit State";
}

<div class="container mt-4">
    <h2>@ViewData["Title"]</h2>
    <form asp-action="AddEdit" asp-controller="State" method="post" enctype="multipart/form-data">
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

        <!-- 👇 File Upload -->
        <div class="mb-3">
            <label asp-for="File" class="form-label">Upload Image</label>
            <input asp-for="File" type="file" class="form-control" />
            <span asp-validation-for="File" class="text-danger"></span>
        </div>

        <!-- 👇 Optional: Show existing image -->
        @if (!string.IsNullOrEmpty(Model.FilePath))
        {
            <div class="mb-3">
                <label class="form-label">Current Image</label><br />
                <img src="https://localhost:7093/@Model.FilePath" alt="State Image" width="120" class="img-thumbnail" />
            </div>
        }

        <button type="submit" class="btn btn-primary">@((Model.StateId == 0) ? "Add" : "Update")</button>
        <a asp-action="Index" class="btn btn-secondary">Back</a>
    </form>
</div>

```
## Final Notes

- Ensure `app.UseStaticFiles();` is enabled in your Web API project in Program.cs.
- The form in MVC view must use `enctype="multipart/form-data"`.
- `[Consumes("multipart/form-data")]` is recommended in API methods for clarity.
