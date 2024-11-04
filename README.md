# UsefulScripts

## Azure YML step to recursively display directory contents
```
- task: PowerShell@2
  displayName: Print Directory To Check Files
  inputs:
    targetType: 'inline'
    script: 'Get-ChildItem -Path ''$(Agent.BuildDirectory)'' -recurse'
```

## Powershell script to wait for user input
```
Write-Host -NoNewLine 'Press any key to continue...';
$null = $Host.UI.RawUI.ReadKey('NoEcho,IncludeKeyDown');
```
