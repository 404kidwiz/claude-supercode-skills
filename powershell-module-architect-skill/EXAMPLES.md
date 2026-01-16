# PowerShell Module Architect - Examples & Patterns

## Anti-Patterns

### Anti-Pattern: Monolithic .psm1 File

**What it looks like:**
```powershell
# BadModule.psm1 (3000 lines in one file)
function Get-User { ... 200 lines ... }
function Set-User { ... 250 lines ... }
function Remove-User { ... 180 lines ... }
# ... 15 more functions ...
```

**Why it fails:**
- Unmaintainable, hard to navigate
- Git conflicts on every change
- No clear public/private separation

**Correct approach:**
```powershell
# Module structure:
# MyModule/
#   MyModule.psm1       <- 20 lines (just dot-sourcing)
#   MyModule.psd1       <- Manifest
#   Public/
#     Get-User.ps1      <- 200 lines
#     Set-User.ps1      <- 250 lines
#   Private/
#     _Helpers.ps1      <- Shared logic

# MyModule.psm1 (clean module file)
$Public = @(Get-ChildItem -Path $PSScriptRoot\Public\*.ps1)
$Private = @(Get-ChildItem -Path $PSScriptRoot\Private\*.ps1)

($Private + $Public) | ForEach-Object { . $_.FullName }
Export-ModuleMember -Function $Public.BaseName
```

---

### Anti-Pattern: Using Wildcards in Exports

**What it looks like:**
```powershell
# Module manifest
FunctionsToExport = '*'
CmdletsToExport = '*'
AliasesToExport = '*'
```

**Why it fails:**
- Exports private helper functions accidentally
- Slower module loading (no explicit list)
- Hard to track breaking changes

**Correct approach:**
```powershell
# Explicit exports
FunctionsToExport = @(
    'Get-OrgUser',
    'Set-OrgUser',
    'New-OrgUser',
    'Remove-OrgUser'
)
CmdletsToExport = @()
AliasesToExport = @('gou', 'sou')
```

---

### Anti-Pattern: Missing Comment-Based Help

**What it looks like:**
```powershell
function Get-OrgUser {
    param($Name)
    Get-ADUser -Identity $Name
}

# User runs: Get-Help Get-OrgUser
# Output: Minimal or no help available
```

**Why it fails:**
- No documentation for users
- Hard to remember parameter names
- No examples to learn from

**Correct approach:**
```powershell
function Get-OrgUser {
    <#
    .SYNOPSIS
        Retrieves Active Directory user by name.
    
    .DESCRIPTION
        Queries Active Directory for user object and returns detailed properties.
    
    .PARAMETER Name
        The username or SamAccountName to search for.
    
    .EXAMPLE
        Get-OrgUser -Name "jdoe"
        
        Returns all properties for user jdoe.
    
    .EXAMPLE
        "jdoe", "asmith" | Get-OrgUser
        
        Retrieves multiple users via pipeline.
    #>
    [CmdletBinding()]
    param(
        [Parameter(Mandatory, ValueFromPipeline)]
        [string]$Name
    )
    
    process {
        Get-ADUser -Identity $Name -Properties *
    }
}

# Now: Get-Help Get-OrgUser shows comprehensive help
```

---

### Anti-Pattern: Hardcoded Paths

**What it looks like:**
```powershell
function Get-Config {
    $configPath = "C:\Scripts\Config\settings.json"
    Get-Content $configPath | ConvertFrom-Json
}
```

**Why it fails:**
- Breaks on other systems
- Can't be unit tested
- No flexibility for environments

**Correct approach:**
```powershell
function Get-Config {
    [CmdletBinding()]
    param(
        [Parameter()]
        [ValidateScript({ Test-Path $_ })]
        [string]$ConfigPath = (Join-Path $PSScriptRoot 'Config\settings.json')
    )
    
    Get-Content $ConfigPath | ConvertFrom-Json
}

# Or use environment variables
$defaultPath = $env:ORG_CONFIG_PATH ?? (Join-Path $PSScriptRoot 'Config\settings.json')
```

---

## Pester Testing Patterns

### Basic Module Tests

```powershell
# Tests/Module.Tests.ps1

BeforeAll {
    $modulePath = Split-Path -Parent $PSScriptRoot
    $moduleName = Split-Path -Leaf $modulePath
    Import-Module "$modulePath\$moduleName.psd1" -Force
}

Describe "$moduleName Module" {
    Context "Module Structure" {
        It "Has a valid manifest" {
            $manifest = Test-ModuleManifest -Path "$modulePath\$moduleName.psd1"
            $manifest | Should -Not -BeNullOrEmpty
        }
        
        It "Exports functions" {
            $functions = Get-Command -Module $moduleName
            $functions.Count | Should -BeGreaterThan 0
        }
        
        It "Has no syntax errors in functions" {
            $scripts = Get-ChildItem "$modulePath\Public\*.ps1"
            foreach ($script in $scripts) {
                $errors = $null
                [System.Management.Automation.Language.Parser]::ParseFile(
                    $script.FullName, [ref]$null, [ref]$errors
                )
                $errors.Count | Should -Be 0 -Because "$($script.Name) should have no syntax errors"
            }
        }
    }
}
```

### Function Tests with Mocking

```powershell
# Tests/Get-OrgUser.Tests.ps1

BeforeAll {
    Import-Module "$PSScriptRoot\..\Organization.ActiveDirectory.psd1" -Force
}

Describe "Get-OrgUser" {
    BeforeAll {
        # Mock Active Directory cmdlet
        Mock Get-ADUser {
            [PSCustomObject]@{
                SamAccountName = $Identity
                Email = "$Identity@example.com"
                Enabled = $true
            }
        } -ModuleName Organization.ActiveDirectory
    }
    
    It "Returns user by identity" {
        $result = Get-OrgUser -Name "jdoe"
        $result.SamAccountName | Should -Be "jdoe"
        $result.Email | Should -Be "jdoe@example.com"
    }
    
    It "Accepts pipeline input" {
        $results = "jdoe", "asmith" | Get-OrgUser
        $results.Count | Should -Be 2
    }
    
    It "Throws on empty name" {
        { Get-OrgUser -Name "" } | Should -Throw
    }
}
```

---

## Error Handling Pattern

```powershell
function Invoke-OrgOperation {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)]
        [string]$Target,
        
        [Parameter()]
        [switch]$Force
    )
    
    begin {
        # Validate prerequisites
        if (-not (Get-Module ActiveDirectory)) {
            try {
                Import-Module ActiveDirectory -ErrorAction Stop
            }
            catch {
                $PSCmdlet.ThrowTerminatingError(
                    [System.Management.Automation.ErrorRecord]::new(
                        [System.InvalidOperationException]::new("ActiveDirectory module required"),
                        "ModuleNotFound",
                        [System.Management.Automation.ErrorCategory]::NotInstalled,
                        $null
                    )
                )
            }
        }
    }
    
    process {
        try {
            # Main operation
            $result = Get-ADUser -Identity $Target -ErrorAction Stop
            
            if ($null -eq $result) {
                Write-Warning "User '$Target' not found, skipping"
                return
            }
            
            # Continue processing...
            $result
        }
        catch [Microsoft.ActiveDirectory.Management.ADIdentityNotFoundException] {
            # Handle specific exception
            Write-Warning "User '$Target' does not exist in Active Directory"
        }
        catch {
            # Handle unexpected errors
            $PSCmdlet.WriteError(
                [System.Management.Automation.ErrorRecord]::new(
                    $_.Exception,
                    "OperationFailed",
                    [System.Management.Automation.ErrorCategory]::OperationStopped,
                    $Target
                )
            )
        }
    }
    
    end {
        Write-Verbose "Operation completed"
    }
}
```

---

## Script to Module Conversion Tool

```powershell
function Convert-ScriptToModule {
    <#
    .SYNOPSIS
        Converts a script with functions into a proper module structure.
    #>
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)]
        [string]$ScriptPath,
        
        [Parameter(Mandatory)]
        [string]$ModuleName,
        
        [Parameter()]
        [string]$OutputPath = ".\Modules"
    )
    
    # Create module structure
    $modulePath = Join-Path $OutputPath $ModuleName
    $publicPath = Join-Path $modulePath "Public"
    $privatePath = Join-Path $modulePath "Private"
    
    New-Item -Path $publicPath -ItemType Directory -Force | Out-Null
    New-Item -Path $privatePath -ItemType Directory -Force | Out-Null
    
    # Parse script for functions
    $ast = [System.Management.Automation.Language.Parser]::ParseFile(
        $ScriptPath, [ref]$null, [ref]$null
    )
    
    $functions = $ast.FindAll({
        param($node)
        $node -is [System.Management.Automation.Language.FunctionDefinitionAst]
    }, $true)
    
    foreach ($func in $functions) {
        $name = $func.Name
        $content = $func.Extent.Text
        
        # Private functions start with underscore or are named "Helper"
        if ($name -match '^_' -or $name -match 'Helper') {
            $destPath = Join-Path $privatePath "$name.ps1"
        }
        else {
            $destPath = Join-Path $publicPath "$name.ps1"
        }
        
        $content | Set-Content -Path $destPath -Encoding UTF8
        Write-Verbose "Extracted: $name -> $destPath"
    }
    
    # Create module file
    $psmContent = @'
$Public = @(Get-ChildItem -Path $PSScriptRoot\Public\*.ps1 -ErrorAction SilentlyContinue)
$Private = @(Get-ChildItem -Path $PSScriptRoot\Private\*.ps1 -ErrorAction SilentlyContinue)

foreach ($import in @($Private + $Public)) {
    try {
        . $import.FullName
    }
    catch {
        Write-Error "Failed to import $($import.FullName): $_"
    }
}

Export-ModuleMember -Function $Public.BaseName
'@
    
    $psmContent | Set-Content -Path (Join-Path $modulePath "$ModuleName.psm1")
    
    # Create manifest
    $publicFunctions = (Get-ChildItem $publicPath -Filter *.ps1).BaseName
    
    New-ModuleManifest -Path (Join-Path $modulePath "$ModuleName.psd1") `
        -RootModule "$ModuleName.psm1" `
        -ModuleVersion '1.0.0' `
        -FunctionsToExport $publicFunctions
    
    Write-Host "Module created at: $modulePath" -ForegroundColor Green
}
```

---

## Integration Examples

### Integration with CI/CD

```yaml
# azure-pipelines.yml
trigger:
  - main

pool:
  vmImage: 'windows-latest'

steps:
  - task: PowerShell@2
    displayName: 'Run Pester Tests'
    inputs:
      targetType: 'inline'
      script: |
        Install-Module Pester -Force -Scope CurrentUser
        $results = Invoke-Pester -Path ./Tests -PassThru
        if ($results.FailedCount -gt 0) {
          throw "Pester tests failed"
        }

  - task: PowerShell@2
    displayName: 'Publish to Gallery'
    condition: eq(variables['Build.SourceBranch'], 'refs/heads/main')
    inputs:
      targetType: 'inline'
      script: |
        Publish-Module -Path ./MyModule -NuGetApiKey $(PSGalleryKey)
```
