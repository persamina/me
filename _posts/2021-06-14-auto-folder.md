---
layout: post
title: "Auto Folder Creation"
author: "Antti"
tags: Reference
---
A couple of brother-in-laws run a contracting business and to help them a bit with organization I wrote a script that anytime a new folder is created in a specified location, a series of subfolders are created in the new folder. Let me show you what I mean.

| Quotes Folder
|   - New Job folder (manual created)
|       - Plan folder (auto created)
|       - Permit folder (auto created)
|       - CustomerInfo folder (auto created)
|       - Choices folder (auto created)

Once the the "New Job" folder is created the Plan, Permit, CustomerInfo and choices folder are created underneath it. 

# Powershell Script
We use the [System.IO.FileSystemWatcher](https://docs.microsoft.com/en-us/dotnet/api/system.io.filesystemwatcher) .Net class to watch for folders created in our directory. The script is fairly simple and short but we create a FilesSystemWatcher ($fsw) event (Register-ObjectEvent) that create waits for a "Created" event to happen. Once the "Created" event happens the subfolder creation action ($Action) happens.
```ps1
Write-Host "Creating File System Watcher" -fore green 
$folder = 'C:\Users\ahpet\Documents\TestCreate'
$filter = '*.*'  # You can enter a wildcard filter here.
$fsw = New-Object IO.FileSystemWatcher $folder, $filter -Property  @{IncludeSubdirectories = $false; NotifyFilter = [IO.NotifyFilters]'DirectoryName'}
$fsw.EnableRaisingEvents = $true

# Subfolder Creation Action
$Action = { 
    $folder = 'C:\Users\ahpet\Documents\TestCreate'
    $name = $Event.SourceEventArgs.Name 
    $changeType = $Event.SourceEventArgs.ChangeType 
    $Event.SourceEventArgs
    $timeStamp = $Event.TimeGenerated 
    $subfolders = @("Plan", "Permit", "Info", "Choices") #Subfolders to create
    Write-Host "The folder '$name' was $changeType at $timeStamp" -fore green 
    foreach ($subfolder in $subfolders) {
	    $path = $folder + "\" + $name + "\" + $subfolder
        if (!(Test-Path $path)) # If subfolder hasn't been created
        {
            mkdir $path #create subfolder
        }
	    else
	    {
	        write-host "Folder exists " + $path 
	    }
    }

 }

Write-Host "Registering File System Watcher" -fore green 
$handlers = . {
    Register-ObjectEvent -InputObject $fsw -EventName Created -SourceIdentifier FolderCreated -Action $Action
}
try
{
    # Loop that keeps script running until script is ended
    do
    {
        Wait-Event -Timeout 5
        
    } while ($true)
}
finally #Once script is ending then stop events and dispose of $fsw
{
    Unregister-Event -SourceIdentifier FolderCreated
    $handlers | Remove-Job
    $fsw.EnableRaisingEvents = $false
    $fsw.Dispose()
}
```

# starting script and startup and running in background

## WindowsTaskScheduler

## Startup Command

# Signing Script
To create folders automatically use following script
1. Modify file to run from appropriate locations- In FolderWatcher.ps1 File modify $folder variable to point to correct directory (in both locations)- Modify $subfolders variables with any variables to change2. Put file in location and make note of location and modify startupFolderCreate.cmd file to point to FolderWatcher.ps1 file location.3. change log file location to log file on computer
3. hit Windows+R type shell:startup to get to startup location.4. Paste startupFolderCreate.cmd into startup location5.