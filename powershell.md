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
function dbao {
  [string[]]$foldersToIgnore='node_modules';
  Get-ChildItem .\ -Include bin,obj -Recurse|Where-Object{$_.FullName -inotmatch "\\$($foldersToIgnore -join '|')\\"}|Remove-Item -Force -Recurse
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
