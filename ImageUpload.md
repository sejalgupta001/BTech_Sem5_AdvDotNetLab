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

## Final Notes

- Ensure `app.UseStaticFiles();` is enabled in your Web API project in Program.cs.
- The form in MVC view must use `enctype="multipart/form-data"`.
- `[Consumes("multipart/form-data")]` is recommended in API methods for clarity.
