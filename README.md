# UsefulScripts

## Azure YML step to recursively display directory contents
```
- task: PowerShell@2
  displayName: Print Directory To Check Files
  inputs:
    targetType: 'inline'
    script: 'Get-ChildItem -Path ''$(Agent.BuildDirectory)'' -recurse'
```
