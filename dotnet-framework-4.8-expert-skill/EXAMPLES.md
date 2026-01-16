# .NET Framework 4.8 Expert - Code Examples & Patterns

This document contains real-world examples, use cases, and implementation patterns for .NET Framework 4.8 development.

## Example 1: Legacy ERP System Modernization

**Scenario:** A manufacturing company needs to add real-time inventory tracking to their 10-year-old ERP system built on .NET Framework 4.5.

**Approach:**
1. **Codebase Analysis**: Mapped dependencies and identified refactoring boundaries
2. **Feature Addition**: Implemented Web API 2 endpoints alongside existing WCF services
3. **Database Integration**: Used Entity Framework 6 with existing SQL Server
4. **Testing Strategy**: Created integration tests for new functionality

**Key Deliverables:**
- Real-time inventory API serving 1000 requests/minute
- Backward compatibility maintained for desktop clients
- Migration path documented for future .NET Core upgrade

### Implementation

```csharp
// Adding Web API 2 alongside existing WCF
// WebApiConfig.cs
public static class WebApiConfig
{
    public static void Register(HttpConfiguration config)
    {
        config.MapHttpAttributeRoutes();
        
        config.Routes.MapHttpRoute(
            name: "DefaultApi",
            routeTemplate: "api/{controller}/{id}",
            defaults: new { id = RouteParameter.Optional }
        );
        
        config.Formatters.JsonFormatter.SerializerSettings.ContractResolver = 
            new CamelCasePropertyNamesContractResolver();
    }
}

// InventoryController.cs
[RoutePrefix("api/inventory")]
public class InventoryController : ApiController
{
    private readonly IInventoryService _inventoryService;
    
    public InventoryController(IInventoryService inventoryService)
    {
        _inventoryService = inventoryService;
    }
    
    [HttpGet]
    [Route("")]
    public async Task<IHttpActionResult> GetAll()
    {
        var items = await _inventoryService.GetAllAsync();
        return Ok(items);
    }
    
    [HttpGet]
    [Route("{id:int}")]
    public async Task<IHttpActionResult> GetById(int id)
    {
        var item = await _inventoryService.GetByIdAsync(id);
        if (item == null)
            return NotFound();
        return Ok(item);
    }
    
    [HttpPost]
    [Route("")]
    public async Task<IHttpActionResult> Create(InventoryItem item)
    {
        if (!ModelState.IsValid)
            return BadRequest(ModelState);
            
        var created = await _inventoryService.CreateAsync(item);
        return CreatedAtRoute("DefaultApi", new { id = created.Id }, created);
    }
}
```

---

## Example 2: WCF to Modern Integration Bridge

**Scenario:** A healthcare provider needs to integrate a legacy WCF service with a modern cloud application.

**Solution Architecture:**
1. **WCF Service**: Enhanced existing SOAP service with additional endpoints
2. **REST API Layer**: Created OWIN-hosted Web API 2 for modern clients
3. **Authentication**: Implemented token-based auth bridging to existing Windows Auth
4. **Message Queue**: Added Azure Service Bus integration for async processing

**Results:**
- Modern clients can access legacy data through REST API
- Zero downtime during deployment
- HIPAA compliance maintained throughout

### Implementation

```csharp
// OWIN Startup for self-hosted Web API
public class Startup
{
    public void Configuration(IAppBuilder app)
    {
        var config = new HttpConfiguration();
        
        // Configure OAuth token validation
        app.UseJwtBearerAuthentication(new JwtBearerAuthenticationOptions
        {
            AuthenticationMode = AuthenticationMode.Active,
            TokenValidationParameters = new TokenValidationParameters
            {
                ValidAudience = ConfigurationManager.AppSettings["jwt:Audience"],
                ValidIssuer = ConfigurationManager.AppSettings["jwt:Issuer"],
                IssuerSigningKey = new SymmetricSecurityKey(
                    Encoding.UTF8.GetBytes(ConfigurationManager.AppSettings["jwt:Secret"]))
            }
        });
        
        WebApiConfig.Register(config);
        app.UseWebApi(config);
    }
}

// Bridge service calling WCF
public class PatientBridgeService : IPatientBridgeService
{
    private readonly PatientServiceClient _wcfClient;
    
    public PatientBridgeService()
    {
        _wcfClient = new PatientServiceClient();
    }
    
    public async Task<PatientDto> GetPatientAsync(string patientId)
    {
        // Call legacy WCF service
        var wcfResult = await Task.Factory.FromAsync(
            _wcfClient.BeginGetPatient(patientId, null, null),
            _wcfClient.EndGetPatient);
        
        // Map to modern DTO
        return new PatientDto
        {
            Id = wcfResult.PatientId,
            Name = $"{wcfResult.FirstName} {wcfResult.LastName}",
            DateOfBirth = wcfResult.DOB,
            MedicalRecordNumber = wcfResult.MRN
        };
    }
}
```

---

## Example 3: Windows Service Migration Assessment

**Scenario:** A financial services firm needs to evaluate options for their core processing Windows Service.

**Assessment Deliverables:**
- Current architecture documentation and dependency map
- Performance benchmarks and bottleneck analysis
- Migration options (lift-and-shift vs. rewrite)
- Risk assessment and phased migration plan
- Cost comparison between .NET Framework and .NET 6 alternatives

### Documentation Template

```csharp
// Service architecture documentation
/*
CURRENT STATE ANALYSIS
======================
Service: PaymentProcessingService
Framework: .NET Framework 4.6.2
Database: SQL Server 2016
Dependencies:
- WCF Service (PaymentGatewayService)
- MSMQ for async processing
- COM component (LegacyValidation.dll)

BOTTLENECKS IDENTIFIED
======================
1. Synchronous database calls on main thread
2. Single-threaded message processing
3. No connection pooling for WCF calls

MIGRATION OPTIONS
=================
Option A: Upgrade in-place to .NET Framework 4.8
- Effort: Low (2-4 weeks)
- Risk: Low
- Benefits: Security updates, performance improvements

Option B: Migrate to .NET 6 Worker Service
- Effort: High (3-6 months)
- Risk: Medium
- Benefits: Cross-platform, modern hosting, better async

Option C: Strangler Fig Pattern
- Effort: Medium (ongoing)
- Risk: Low
- Benefits: Incremental migration, continuous value delivery
*/
```

---

## Common Patterns

### Global Exception Handling (ASP.NET MVC)

```csharp
// Global.asax.cs
protected void Application_Error(object sender, EventArgs e)
{
    var exception = Server.GetLastError();
    var httpException = exception as HttpException;
    
    // Log the error
    var logger = LogManager.GetLogger(typeof(MvcApplication));
    logger.Error("Unhandled exception", exception);
    
    // Clear the error
    Server.ClearError();
    
    // Redirect to error page
    var routeData = new RouteData();
    routeData.Values.Add("controller", "Error");
    routeData.Values.Add("action", "Index");
    routeData.Values.Add("exception", exception);
    
    if (httpException != null)
    {
        routeData.Values.Add("statusCode", httpException.GetHttpCode());
    }
    
    IController errorController = new ErrorController();
    errorController.Execute(new RequestContext(new HttpContextWrapper(Context), routeData));
}
```

### Dependency Injection (Unity Container)

```csharp
// UnityConfig.cs
public static class UnityConfig
{
    public static void RegisterComponents()
    {
        var container = new UnityContainer();
        
        // Register services
        container.RegisterType<IProductRepository, ProductRepository>();
        container.RegisterType<IProductService, ProductService>();
        container.RegisterType<IOrderRepository, OrderRepository>();
        container.RegisterType<IOrderService, OrderService>();
        
        // Register DbContext per request
        container.RegisterType<ApplicationDbContext>(
            new PerRequestLifetimeManager());
        
        // Set dependency resolver
        DependencyResolver.SetResolver(new UnityDependencyResolver(container));
    }
}
```

### Async/Await Pattern (EF6)

```csharp
public class ProductService : IProductService
{
    private readonly ApplicationDbContext _context;
    
    public ProductService(ApplicationDbContext context)
    {
        _context = context;
    }
    
    public async Task<IEnumerable<Product>> GetAllAsync()
    {
        return await _context.Products
            .Include(p => p.Category)
            .OrderBy(p => p.Name)
            .ToListAsync();
    }
    
    public async Task<Product> GetByIdAsync(int id)
    {
        return await _context.Products
            .Include(p => p.Category)
            .FirstOrDefaultAsync(p => p.Id == id);
    }
    
    public async Task<Product> CreateAsync(Product product)
    {
        _context.Products.Add(product);
        await _context.SaveChangesAsync();
        return product;
    }
    
    public async Task UpdateAsync(Product product)
    {
        _context.Entry(product).State = EntityState.Modified;
        await _context.SaveChangesAsync();
    }
}
```

---

## Anti-Patterns and Fixes

### Anti-Pattern: DbContext as Singleton

**Problem:**
```csharp
// BAD: DbContext as singleton causes memory leaks and stale data
public static class DatabaseHelper
{
    private static readonly ApplicationDbContext _context = new ApplicationDbContext();
    
    public static List<Product> GetProducts()
    {
        return _context.Products.ToList();
    }
}
```

**Solution:**
```csharp
// GOOD: DbContext per request with disposal
public class ProductRepository : IProductRepository, IDisposable
{
    private readonly ApplicationDbContext _context;
    
    public ProductRepository()
    {
        _context = new ApplicationDbContext();
    }
    
    public List<Product> GetProducts()
    {
        return _context.Products.ToList();
    }
    
    public void Dispose()
    {
        _context?.Dispose();
    }
}
```

### Anti-Pattern: N+1 Queries

**Problem:**
```csharp
// BAD: N+1 query problem
var products = _context.Products.ToList();
foreach (var product in products)
{
    // This causes a separate query for each product!
    Console.WriteLine(product.Category.Name);
}
```

**Solution:**
```csharp
// GOOD: Eager loading with Include
var products = _context.Products
    .Include(p => p.Category)
    .ToList();
    
foreach (var product in products)
{
    Console.WriteLine(product.Category.Name);
}
```

---

## Migration Checklist

### Before Migrating to .NET 6+

- [ ] Inventory all NuGet packages and check .NET 6 compatibility
- [ ] Identify WCF services (need CoreWCF or replacement)
- [ ] Check for COM dependencies (may need Windows-only)
- [ ] Evaluate Windows Forms/WPF usage (Windows-only in .NET 6)
- [ ] Review web.config usage (move to appsettings.json)
- [ ] Check for System.Web dependencies (not in .NET Core)
- [ ] Run .NET Upgrade Assistant for automated analysis
- [ ] Create comprehensive test suite before migration
- [ ] Document all configuration settings
- [ ] Plan for IIS to Kestrel hosting changes
