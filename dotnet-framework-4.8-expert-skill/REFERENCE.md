# .NET Framework 4.8 Expert - Technical Reference

This document contains detailed technical specifications, configurations, and implementation patterns for .NET Framework 4.8 development.

## WCF Service Implementation

### Service Contract and Implementation

```csharp
// Service Contract
[ServiceContract]
public interface IProductService
{
    [OperationContract]
    List<Product> GetAllProducts();
    
    [OperationContract]
    Product GetProductById(int id);
    
    [OperationContract]
    void AddProduct(Product product);
    
    [OperationContract]
    [WebInvoke(Method = "POST", RequestFormat = WebMessageFormat.Json, 
               ResponseFormat = WebMessageFormat.Json, UriTemplate = "products")]
    Product AddRestProduct(Product product);
}

// Service Implementation
public class ProductService : IProductService
{
    private readonly IProductRepository _repository;
    
    public ProductService()
    {
        _repository = new ProductRepository();
    }
    
    public List<Product> GetAllProducts()
    {
        return _repository.GetAll().ToList();
    }
    
    public Product GetProductById(int id)
    {
        return _repository.GetById(id);
    }
    
    public void AddProduct(Product product)
    {
        _repository.Add(product);
    }
    
    public Product AddRestProduct(Product product)
    {
        _repository.Add(product);
        return product;
    }
}
```

### Service Configuration (Web.config)

```xml
<system.serviceModel>
  <services>
    <service name="MyApp.Services.ProductService" behaviorConfiguration="ServiceBehavior">
      <endpoint address="basic" binding="basicHttpBinding" contract="MyApp.Services.IProductService" />
      <endpoint address="rest" binding="webHttpBinding" contract="MyApp.Services.IProductService" behaviorConfiguration="RestBehavior" />
      <host>
        <baseAddresses>
          <add baseAddress="http://localhost:8080/ProductService/" />
        </baseAddresses>
      </host>
    </service>
  </services>
  <behaviors>
    <serviceBehaviors>
      <behavior name="ServiceBehavior">
        <serviceMetadata httpGetEnabled="true" />
        <serviceDebug includeExceptionDetailInFaults="true" />
      </behavior>
    </serviceBehaviors>
    <endpointBehaviors>
      <behavior name="RestBehavior">
        <webHttp />
      </behavior>
    </endpointBehaviors>
  </behaviors>
</system.serviceModel>
```

---

## ASP.NET MVC 5 Controller

```csharp
// Controller
public class ProductController : Controller
{
    private readonly IProductService _productService;
    
    public ProductController()
    {
        _productService = new ProductService();
    }
    
    // GET: Product
    public ActionResult Index()
    {
        var products = _productService.GetAllProducts();
        return View(products);
    }
    
    // GET: Product/Details/5
    public ActionResult Details(int id)
    {
        var product = _productService.GetProductById(id);
        if (product == null)
        {
            return HttpNotFound();
        }
        return View(product);
    }
    
    // GET: Product/Create
    public ActionResult Create()
    {
        return View();
    }
    
    // POST: Product/Create
    [HttpPost]
    [ValidateAntiForgeryToken]
    public ActionResult Create(Product product)
    {
        if (ModelState.IsValid)
        {
            _productService.AddProduct(product);
            return RedirectToAction("Index");
        }
        return View(product);
    }
    
    // GET: Product/Edit/5
    public ActionResult Edit(int id)
    {
        var product = _productService.GetProductById(id);
        if (product == null)
        {
            return HttpNotFound();
        }
        return View(product);
    }
    
    // POST: Product/Edit/5
    [HttpPost]
    [ValidateAntiForgeryToken]
    public ActionResult Edit(Product product)
    {
        if (ModelState.IsValid)
        {
            // Update logic here
            return RedirectToAction("Index");
        }
        return View(product);
    }
}

// View Model
public class ProductViewModel
{
    public int Id { get; set; }
    [Required(ErrorMessage = "Product name is required")]
    [StringLength(100)]
    public string Name { get; set; }
    
    [Required]
    [Range(0.01, 9999.99)]
    public decimal Price { get; set; }
    
    [DataType(DataType.MultilineText)]
    public string Description { get; set; }
}
```

---

## Entity Framework 6 Implementation

### DbContext Configuration

```csharp
// DbContext
public class ApplicationDbContext : DbContext
{
    public ApplicationDbContext() : base("name=DefaultConnection")
    {
        Database.SetInitializer(new MigrateDatabaseToLatestVersion<ApplicationDbContext, Configuration>());
    }
    
    public DbSet<Product> Products { get; set; }
    public DbSet<Category> Categories { get; set; }
    public DbSet<Order> Orders { get; set; }
    
    protected override void OnModelCreating(DbModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Product>()
            .Property(p => p.Price)
            .HasPrecision(10, 2);
            
        modelBuilder.Entity<Product>()
            .HasRequired(p => p.Category)
            .WithMany(c => c.Products)
            .HasForeignKey(p => p.CategoryId);
            
        base.OnModelCreating(modelBuilder);
    }
}
```

### Repository Pattern

```csharp
// Repository Pattern
public class ProductRepository : IProductRepository
{
    private readonly ApplicationDbContext _context;
    
    public ProductRepository()
    {
        _context = new ApplicationDbContext();
    }
    
    public IEnumerable<Product> GetAll()
    {
        return _context.Products.Include(p => p.Category).ToList();
    }
    
    public Product GetById(int id)
    {
        return _context.Products.Include(p => p.Category).FirstOrDefault(p => p.Id == id);
    }
    
    public void Add(Product product)
    {
        _context.Products.Add(product);
        _context.SaveChanges();
    }
    
    public void Update(Product product)
    {
        _context.Entry(product).State = EntityState.Modified;
        _context.SaveChanges();
    }
    
    public void Delete(int id)
    {
        var product = _context.Products.Find(id);
        if (product != null)
        {
            _context.Products.Remove(product);
            _context.SaveChanges();
        }
    }
}
```

---

## Windows Service Implementation

```csharp
// Windows Service
public class FileProcessorService : ServiceBase
{
    private Timer _timer;
    private readonly IFileProcessor _fileProcessor;
    
    public FileProcessorService()
    {
        ServiceName = "FileProcessorService";
        _fileProcessor = new FileProcessor();
    }
    
    protected override void OnStart(string[] args)
    {
        _timer = new Timer(1000 * 60 * 5); // Every 5 minutes
        _timer.Elapsed += ProcessFiles;
        _timer.Start();
        
        EventLog.WriteEntry("File Processor Service started");
    }
    
    protected override void OnStop()
    {
        _timer?.Stop();
        _timer?.Dispose();
        EventLog.WriteEntry("File Processor Service stopped");
    }
    
    private void ProcessFiles(object sender, ElapsedEventArgs e)
    {
        try
        {
            _fileProcessor.ProcessPendingFiles();
        }
        catch (Exception ex)
        {
            EventLog.WriteEntry($"Error processing files: {ex.Message}", EventLogEntryType.Error);
        }
    }
}

// Service Installer
[RunInstaller(true)]
public class FileProcessorServiceInstaller : Installer
{
    public FileProcessorServiceInstaller()
    {
        var serviceProcessInstaller = new ServiceProcessInstaller
        {
            Account = ServiceAccount.LocalSystem
        };
        
        var serviceInstaller = new ServiceInstaller
        {
            ServiceName = "FileProcessorService",
            DisplayName = "File Processor Service",
            Description = "Processes files from incoming directory",
            StartType = ServiceStartMode.Automatic
        };
        
        Installers.Add(serviceProcessInstaller);
        Installers.Add(serviceInstaller);
    }
}
```

---

## COM Interop Example

```csharp
// COM Interop for Excel
using Excel = Microsoft.Office.Interop.Excel;
using System.Runtime.InteropServices;

public class ExcelExporter
{
    public void ExportDataToExcel(List<Product> products, string filePath)
    {
        Excel.Application excelApp = null;
        Excel.Workbook workbook = null;
        Excel.Worksheet worksheet = null;
        
        try
        {
            excelApp = new Excel.Application();
            excelApp.Visible = false;
            
            workbook = excelApp.Workbooks.Add();
            worksheet = (Excel.Worksheet)workbook.ActiveSheet;
            
            // Headers
            worksheet.Cells[1, 1] = "ID";
            worksheet.Cells[1, 2] = "Name";
            worksheet.Cells[1, 3] = "Price";
            worksheet.Cells[1, 4] = "Category";
            
            // Data
            int row = 2;
            foreach (var product in products)
            {
                worksheet.Cells[row, 1] = product.Id;
                worksheet.Cells[row, 2] = product.Name;
                worksheet.Cells[row, 3] = product.Price;
                worksheet.Cells[row, 4] = product.Category?.Name;
                row++;
            }
            
            // AutoFit columns
            worksheet.Columns.AutoFit();
            
            workbook.SaveAs(filePath);
            workbook.Close();
            excelApp.Quit();
        }
        finally
        {
            // Cleanup COM objects
            if (worksheet != null) Marshal.ReleaseComObject(worksheet);
            if (workbook != null) Marshal.ReleaseComObject(workbook);
            if (excelApp != null) Marshal.ReleaseComObject(excelApp);
            
            GC.Collect();
            GC.WaitForPendingFinalizers();
        }
    }
}
```

---

## Project Configuration

### Web.config

```xml
<!-- Web.config configuration -->
<configuration>
  <connectionStrings>
    <add name="DefaultConnection" connectionString="Data Source=.;Initial Catalog=MyAppDb;Integrated Security=True" 
         providerName="System.Data.SqlClient" />
  </connectionStrings>
  
  <appSettings>
    <add key="webpages:Version" value="3.0.0.0" />
    <add key="webpages:Enabled" value="false" />
    <add key="ClientValidationEnabled" value="true" />
    <add key="UnobtrusiveJavaScriptEnabled" value="true" />
  </appSettings>
  
  <system.web>
    <compilation debug="true" targetFramework="4.8" />
    <httpRuntime targetFramework="4.8" />
    <authentication mode="Forms">
      <forms loginUrl="~/Account/Login" timeout="2880" />
    </authentication>
  </system.web>
</configuration>
```

### Development Workflow

**Testing Strategy:**
- Unit tests with MSTest or NUnit
- Integration tests with test databases
- WCF service testing with test clients
- ASP.NET MVC controller testing with mocked services
- Performance testing with load testing tools

**Deployment and Configuration:**
- Web Deploy for ASP.NET applications
- Windows Installer for Windows Services
- Configuration transforms for different environments
- IIS configuration and application pool management
