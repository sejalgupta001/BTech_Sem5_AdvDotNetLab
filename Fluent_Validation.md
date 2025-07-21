# Fluent Validation

This lab demonstrates how to use FluentValidation in an ASP.NET Core Web API project to implement model validation.

---

### 1. Install Required Packages

Run the following commands to install necessary packages:

```bash
dotnet add package FluentValidation.AspNetCore
```

---

### 2. Update `Program.cs`

In the `Program.cs` file, add the following configuration to register FluentValidation:

#### a. Register All Validators in the Assembly:

```csharp
using System.Reflection;

builder.Services.AddControllers()
    .AddFluentValidation(fv =>
        fv.RegisterValidatorsFromAssemblies(new[] { Assembly.GetExecutingAssembly() }));
```

#### b. Register a Single Validator:

If you want to register a single validator instead of all validators in the assembly, use:

```csharp
builder.Services.AddControllers()
    .AddFluentValidation(fv =>
        fv.RegisterValidatorsFromAssemblyContaining<CountryValidator>());
```

---

### 3. Define the `CountryModel`

Create a `Models` folder and add the `CountryModel` class:

```csharp
namespace api.Model
{
    public class CountryModel
    {
        public int? CountryID { get; set; }
        public string CountryName { get; set; }
        public string CountryCode { get; set; }
    }
}
```

---

### 5. Implement Fluent Validation

Create a `Validators` folder and add the `CountryValidator` class:

```csharp
using api.Model;
using FluentValidation;

namespace api.Validators
{
    public class CountryValidator : AbstractValidator<CountryModel>
    {
        public CountryValidator()
        {
            RuleFor(c => c.CountryName)
                 .NotNull().WithMessage("Country name must not be empty.")
                 .Length(3, 20).WithMessage("Country name must be between 3 and 20 characters.")
                 .Matches("^[A-Za-z ]*$").WithMessage("Country name must contain only letters and spaces."); // Ensures only letters and spaces are allowed

            RuleFor(c => c.CountryCode)
                .NotNull().WithMessage("Country code must not be empty.")
                .MaximumLength(4).WithMessage("Country code must not exceed 4 characters.")
                .Matches("^[A-Za-z]{2,4}$").WithMessage("Country code must contain only letters and be between 2 to 4 characters.");

        }
    }
}

```
