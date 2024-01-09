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
