# PowerShell Module Architect - Technical Reference

## Profile Optimization Workflow

**Use case:** Reduce profile load time from 8 seconds to <1 second

### Step 1: Diagnose Slow Loading

```powershell
# Measure-Command each section in profile
$sw = [System.Diagnostics.Stopwatch]::StartNew()

# Section 1: Module Imports
Import-Module ActiveDirectory  # Measure this
$sw.Stop()
Write-Host "ActiveDirectory: $($sw.ElapsedMilliseconds)ms"
$sw.Restart()

# Section 2: Custom Functions
. C:\Scripts\MyHelpers.ps1  # Measure this
$sw.Stop()
Write-Host "MyHelpers: $($sw.ElapsedMilliseconds)ms"

# Expected output:
# ActiveDirectory: 3200ms ← SLOW!
# MyHelpers: 150ms
```

### Step 2: Apply Lazy Loading

```powershell
# BEFORE (slow - loads immediately)
Import-Module ActiveDirectory
Import-Module ExchangeOnlineManagement
Import-Module AzureAD

# AFTER (fast - loads on first use)
function Load-ADModule {
    if (-not (Get-Module ActiveDirectory)) {
        Import-Module ActiveDirectory
    }
}

# Create wrapper aliases that trigger load
function Get-ADUser {
    Load-ADModule
    Microsoft.ActiveDirectory.Management\Get-ADUser @args
}
```

### Step 3: Fragment Profile

```powershell
# Microsoft.PowerShell_profile.ps1 (main profile - keep MINIMAL)
$profileFragments = @(
    "$PSScriptRoot\Fragments\Core.ps1",      # Essential only
    "$PSScriptRoot\Fragments\Aliases.ps1",   # Quick aliases
)

foreach ($fragment in $profileFragments) {
    if (Test-Path $fragment) {
        . $fragment
    }
}

# Load heavy modules on-demand only
```

### Step 4: Optimize Common Tasks

```powershell
# Instead of full module load, create focused wrapper
function gadu {  # Get-ADUser shortcut
    param($Identity)
    
    # Load module lazily
    if (-not (Get-Command Get-ADUser -ErrorAction SilentlyContinue)) {
        Import-Module ActiveDirectory
    }
    
    Get-ADUser -Identity $Identity -Properties *
}

# Result: 0ms load time, 3200ms only when actually needed
```

---

## Module Manifest Template

**When to use:** Publishing module or enabling auto-discovery

```powershell
@{
    # Module Identity
    RootModule        = 'ModuleName.psm1'
    ModuleVersion     = '1.0.0'
    GUID              = 'a1b2c3d4-e5f6-4a5b-8c9d-0e1f2a3b4c5d'
    
    # Author Information
    Author            = 'Your Team'
    CompanyName       = 'Your Organization'
    Copyright         = '(c) 2024. All rights reserved.'
    
    # Module Description
    Description       = 'Comprehensive module description and purpose'
    
    # PowerShell Requirements
    PowerShellVersion = '5.1'  # Minimum version
    
    # Required Modules (dependencies)
    RequiredModules   = @(
        @{ ModuleName = 'ActiveDirectory'; ModuleVersion = '1.0.0.0' },
        @{ ModuleName = 'AzureAD'; RequiredVersion = '2.0.2.4' }
    )
    
    # Exported Functions (explicit is better than wildcards)
    FunctionsToExport = @(
        'Get-Something',
        'Set-Something',
        'New-Something',
        'Remove-Something'
    )
    
    # Exported Cmdlets, Variables, Aliases
    CmdletsToExport   = @()
    VariablesToExport = @()
    AliasesToExport   = @('gs', 'getsth')  # Aliases for functions
    
    # Private Data (tags for PowerShell Gallery)
    PrivateData       = @{
        PSData = @{
            Tags       = @('ActiveDirectory', 'Automation', 'Management')
            LicenseUri = 'https://github.com/org/repo/blob/main/LICENSE'
            ProjectUri = 'https://github.com/org/repo'
            ReleaseNotes = 'Initial release with core functionality'
        }
    }
}
```

---

## Dynamic Parameters Pattern

**When to use:** Parameter options depend on other parameter values

```powershell
function Get-LogFile {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)]
        [ValidateSet('Application', 'System', 'Security')]
        [string]$LogType
    )
    
    dynamicparam {
        # Dynamic parameter only appears if LogType is 'Application'
        if ($LogType -eq 'Application') {
            $attributes = New-Object System.Management.Automation.ParameterAttribute
            $attributes.Mandatory = $false
            
            $attributeCollection = New-Object System.Collections.ObjectModel.Collection[System.Attribute]
            $attributeCollection.Add($attributes)
            
            $validateSet = New-Object System.Management.Automation.ValidateSetAttribute(
                @('Error', 'Warning', 'Information')
            )
            $attributeCollection.Add($validateSet)
            
            $param = New-Object System.Management.Automation.RuntimeDefinedParameter(
                'Severity', [string], $attributeCollection
            )
            
            $paramDictionary = New-Object System.Management.Automation.RuntimeDefinedParameterDictionary
            $paramDictionary.Add('Severity', $param)
            return $paramDictionary
        }
    }
    
    process {
        # Access dynamic parameter
        $severity = $PSBoundParameters['Severity']
        
        Write-Host "LogType: $LogType"
        if ($severity) {
            Write-Host "Severity: $severity"
        }
    }
}

# Usage:
Get-LogFile -LogType Application -Severity Error  # Severity parameter available
Get-LogFile -LogType System  # Severity parameter NOT available
```

---

## Cross-Version Compatibility Pattern

```powershell
# Detect PowerShell version and use appropriate syntax
function Get-OrgData {
    [CmdletBinding()]
    param(
        [string]$Path
    )
    
    # Check PowerShell version
    $isPSCore = $PSVersionTable.PSEdition -eq 'Core'
    
    if ($isPSCore) {
        # PowerShell 7+ syntax
        $data = Get-Content $Path | ConvertFrom-Json -AsHashtable
        
        # Use null-coalescing (7+ only)
        $result = $data.value ?? 'default'
    }
    else {
        # PowerShell 5.1 compatible syntax
        $data = Get-Content $Path | ConvertFrom-Json
        
        # Manual null check
        $result = if ($null -ne $data.value) { $data.value } else { 'default' }
    }
    
    return $result
}

# Alternative: Feature detection
function Test-Feature {
    param([string]$Feature)
    
    switch ($Feature) {
        'NullCoalescing' { return $PSVersionTable.PSVersion.Major -ge 7 }
        'TernaryOperator' { return $PSVersionTable.PSVersion.Major -ge 7 }
        'PipelineChaining' { return $PSVersionTable.PSVersion.Major -ge 7 }
        'ParallelForEach' { return $PSVersionTable.PSVersion.Major -ge 7 }
        default { return $false }
    }
}
```

---

## Parameter Validation Patterns

```powershell
function Set-Configuration {
    [CmdletBinding(SupportsShouldProcess)]
    param(
        # Required with validation
        [Parameter(Mandatory)]
        [ValidateNotNullOrEmpty()]
        [string]$ConfigName,
        
        # Validate against set of values
        [Parameter()]
        [ValidateSet('Development', 'Staging', 'Production')]
        [string]$Environment = 'Development',
        
        # Validate range
        [Parameter()]
        [ValidateRange(1, 100)]
        [int]$MaxConnections = 10,
        
        # Validate pattern (regex)
        [Parameter()]
        [ValidatePattern('^[a-zA-Z0-9_-]+$')]
        [string]$Identifier,
        
        # Validate script (custom logic)
        [Parameter()]
        [ValidateScript({
            if (Test-Path $_) { $true }
            else { throw "Path '$_' does not exist" }
        })]
        [string]$ConfigPath,
        
        # Validate length
        [Parameter()]
        [ValidateLength(3, 50)]
        [string]$Description,
        
        # Validate count (for arrays)
        [Parameter()]
        [ValidateCount(1, 10)]
        [string[]]$Tags
    )
    
    process {
        if ($PSCmdlet.ShouldProcess($ConfigName, "Update configuration")) {
            # Implementation
        }
    }
}
```

---

## Module Loading Performance

### Benchmark Module Load Time

```powershell
function Measure-ModuleLoadTime {
    param(
        [Parameter(Mandatory)]
        [string]$ModuleName
    )
    
    # Ensure module is not loaded
    Remove-Module $ModuleName -ErrorAction SilentlyContinue
    
    $times = @()
    
    for ($i = 0; $i -lt 5; $i++) {
        $sw = [System.Diagnostics.Stopwatch]::StartNew()
        Import-Module $ModuleName
        $sw.Stop()
        $times += $sw.ElapsedMilliseconds
        
        Remove-Module $ModuleName
    }
    
    [PSCustomObject]@{
        ModuleName = $ModuleName
        MinTime = ($times | Measure-Object -Minimum).Minimum
        MaxTime = ($times | Measure-Object -Maximum).Maximum
        AvgTime = [math]::Round(($times | Measure-Object -Average).Average, 2)
    }
}

# Usage
Measure-ModuleLoadTime -ModuleName 'Organization.ActiveDirectory'
```

### Optimize with Assembly Caching

```powershell
# For modules with compiled components
$cacheDir = Join-Path $env:LOCALAPPDATA 'PowerShellModuleCache'

if (-not (Test-Path $cacheDir)) {
    New-Item -Path $cacheDir -ItemType Directory -Force
}

# Pre-compile module on first load
$compiledPath = Join-Path $cacheDir "$ModuleName.dll"
if (-not (Test-Path $compiledPath)) {
    # Compile and cache
    Add-Type -Path "$PSScriptRoot\Source\*.cs" -OutputAssembly $compiledPath
}

# Load from cache
Import-Module $compiledPath
```
