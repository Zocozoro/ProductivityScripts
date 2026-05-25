# This is to be stored in the `$profile` powershell file

Changes prompt to be a little more readable 
- by shrinking current path to just current directory
- adding a linespace after command output
- then add a long blue line
```
function prompt {
  $host.UI.RawUI.WindowTitle = $pwd.Path
  $folder = (Get-Location).Path.Split('\')[-1]
  Write-Host ""
  Write-Host "------------------------------------------------------------------------------------ `n$folder$('>' * ($nestedPromptLevel + 1))" -NoNewline -ForegroundColor Blue
  return " "
}
```

This is specifically for working on a Framework MVC project where Razor view compilation has been disabled to improve build time. This will toggle the build argument in a web.config file to allow for quick testing after a large refactor.
```
function tmvc {
  [string] $FilePath = "{absolute_path_to_.csproj_file}"

  $content = Get-Content -Path $FilePath -Raw

  $content = $content -replace '<MvcBuildViews>true</MvcBuildViews>', '<MvcBuildViews>temp</MvcBuildViews>'
  $content = $content -replace '<MvcBuildViews>false</MvcBuildViews>', '<MvcBuildViews>true</MvcBuildViews>'
  $content = $content -replace '<MvcBuildViews>temp</MvcBuildViews>', '<MvcBuildViews>false</MvcBuildViews>'

  $content | Out-File -FilePath $FilePath -NoNewline

  Write-Host "MvcBuildViews toggled in the file: $FilePath"
}
```

A function to delete every bin/obj folder recursively from current location. Created due to frustration with VS not doing this correctly. Bit of a nuclear option.
```
# Delete all bin and obj folders as 
function dbao {
    [CmdletBinding(SupportsShouldProcess=$true)]
    param(
        [string[]]$FoldersToIgnore = @('node_modules'),
        [switch]$Quiet
    )

    $root = (Get-Location).Path
    $ignoreRegex = "\\($($FoldersToIgnore -join '|'))\\"

    if (-not $Quiet) {
        Write-Host "Scanning for bin/obj folders under: $root"
        if ($FoldersToIgnore.Count -gt 0) {
            Write-Host "Ignoring paths matching: $($FoldersToIgnore -join ', ')"
        }
    }

    # Find candidates first so we can show progress
    $targets = Get-ChildItem -Path $root -Directory -Recurse -Force -ErrorAction SilentlyContinue |
        Where-Object { $_.Name -in @('bin','obj') -and $_.FullName -inotmatch $ignoreRegex }

    $total = @($targets).Count
    if ($total -eq 0) {
        if (-not $Quiet) { Write-Host "No bin/obj folders found." }
        return
    }

    if (-not $Quiet) { Write-Host "Found $total folder(s) to delete." }

    $i = 0
    foreach ($t in $targets) {
        $i++
        if (-not $Quiet) {
            Write-Progress -Activity "Deleting build artifacts" -Status "$i / ${$total}: $($t.FullName)" -PercentComplete (($i / $total) * 100)
        }

        if ($PSCmdlet.ShouldProcess($t.FullName, "Remove build artifact folder")) {
            try {
                Remove-Item -LiteralPath $t.FullName -Force -Recurse -ErrorAction Stop
            }
            catch {
                Write-Warning "Failed to delete: $($t.FullName) — $($_.Exception.Message)"
            }
        }
    }

    if (-not $Quiet) {
        Write-Progress -Activity "Deleting build artifacts" -Completed
        Write-Host "Done. Deleted: $total"
    }
}
```

A file to scan a .NET Framework .csproj and remove duplicated file references
```
# Path to your .csproj file in the same directory
$csprojPath = "$PSScriptRoot\Application.csproj"

# Load the .csproj file as XML
Write-Output "Loading .csproj file: $csprojPath"
[xml]$csproj = Get-Content $csprojPath

if ($csproj -eq $null) {
    Write-Output "Failed to load the .csproj file."
    exit
}

Write-Output "Successfully loaded the .csproj file."

# Initialize a hash table to track unique 'Include' attributes
$uniqueIncludes = @{}

# Flag to indicate if changes were made
$changesMade = $false

# Iterate through all ItemGroup nodes
foreach ($itemGroup in $csproj.Project.ItemGroup) {
    # Get all 'Compile' elements within the current ItemGroup
    $compileNodes = $itemGroup.Compile

    # Check if there are any Compile nodes in the current ItemGroup
    if ($compileNodes) {
        # Iterate through each Compile node
        foreach ($compileNode in $compileNodes) {
            $include = $compileNode.Include
            
            # Check if the 'Include' attribute has already been encountered
            if ($uniqueIncludes.ContainsKey($include)) {
                Write-Output "Duplicate found: $include"
                # Remove duplicate node
                $compileNode.ParentNode.RemoveChild($compileNode) | Out-Null
                $changesMade = $true
            } else {
                # Add the 'Include' attribute to the hash table
                $uniqueIncludes[$include] = $true
            }
        }
    } else {
        Write-Output "No Compile nodes found in this ItemGroup."
    }
}

# Save the cleaned .csproj file to the original file if changes were made
if ($changesMade) {
    Write-Output "Changes detected. Saving the cleaned .csproj file to: $csprojPath"
    try {
        $xmlWriterSettings = New-Object System.Xml.XmlWriterSettings
        $xmlWriterSettings.Indent = $true
        $xmlWriterSettings.IndentChars = "  "  # 2 spaces
        $xmlWriterSettings.NewLineChars = "`n"
        $xmlWriterSettings.Encoding = [System.Text.Encoding]::UTF8

        $xmlWriter = [System.Xml.XmlWriter]::Create($csprojPath, $xmlWriterSettings)
        $csproj.WriteTo($xmlWriter)
        $xmlWriter.Close()
        Write-Output "Finished processing. The .csproj file has been updated and saved."
    } catch {
        Write-Output "An error occurred while saving the cleaned .csproj file: $_"
    }
} else {
    Write-Output "No duplicates found. No changes were made to the .csproj file."
}
```

Get each owner of currently git merge conflicts
```
# Print out owners of current git merge conflicts
function conflicts {

    $files = git diff --name-only --diff-filter=U

    if (-not $files) {
        Write-Host "No merge conflicts found."
        return
    }

    $mergeHead = git rev-parse --verify MERGE_HEAD 2>$null
    if (-not $mergeHead) {
        Write-Host "You are not in an unresolved merge state."
        return
    }

    $results = foreach ($file in $files) {

        $ours = git log -1 HEAD --format="%an" -- "$file" 2>$null
        $theirs = git log -1 MERGE_HEAD --format="%an" -- "$file" 2>$null

        [PSCustomObject]@{
            File     = $file
            Current  = $ours
            Previous = $theirs
        }
    }

    $results | Format-Table -AutoSize
}
```

Some string search commands
```
# String search
function ss {
    param(
        [Parameter(Mandatory=$true, Position=0)]
        [string]$Pattern,
        [Parameter(Position=1)]
        [string]$Path = '.'
    )
    Get-ChildItem -Path $Path -Recurse -File -ErrorAction SilentlyContinue |
        Select-String -Pattern $Pattern |
        Group-Object Path |
        ForEach-Object {
            ''
            $_.Name
            '-' * 80
            $_.Group | ForEach-Object { "  Line $($_.LineNumber): $($_.Line.Trim())" }
        }
}

# String search to file
function ssf {
    param(
        [Parameter(Mandatory=$true, Position=0)]
        [string]$Pattern,
        [Parameter(Position=1)]
        [string]$Path = '.',
        [Parameter(Position=2)]
        [string]$OutFile
    )

    if (-not $OutFile) {
        $timestamp = Get-Date -Format 'yyyyMMdd_HHmmss'
        $OutFile = "ss_results_$timestamp.txt"
    }

    $output = ss $Pattern $Path

    if (-not $output) {
        Write-Host "No matches found." -ForegroundColor Yellow
        return
    }

    $output | Out-File -FilePath $OutFile -Encoding UTF8
    Write-Host "Saved results to $OutFile" -ForegroundColor Green
}
```
