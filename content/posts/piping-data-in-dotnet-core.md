---
title: "Piping data to an application in .NET Core"
date: 2019-06-16T18:12:04+03:00
draft: false
---

Earlier today I had a need for a [PlantUML](http://plantuml.com/) API that could generate PlantUML
diagrams without having to install the necessary dependencies on my PC. To achieve this I decided
to write a small API using ASP.NET Core.

I was thinking of just calling the PlantUML binary with the input data from a HTTP POST body. There
could be some security implications but I was the only user of the system so it didn't really matter.
I was thinking piping the input data should be a very simple thing to do. It sure was in the end but
there were a few things that I forgot about when dealing with streams.

I implemented an API for it and started writing code to do the piping.

```Csharp
var process = new Process
{
    StartInfo = new ProcessStartInfo
    {
        FileName = _appSettings.JavaExecutablePath,
        Arguments = $"-jar {_appSettings.PlantUMLPath} -pipe"
        RedirectStandardInput = true,
        RedirectStandardOutput = true,
    }
};


process.Start();
await userInputtedPlantUMLStream.CopyToAsync(process.StandardInput.BaseStream);
var outputStream = new MemoryStream();
await process.StandardOutput.BaseStream.CopyToAsync(outputStream);
process.WaitForExit();
outputStream.Seek(0, SeekOrigin.Begin);
return outputStream;
```

I thought this should do the trick. I started the API and did a request but the application would just hang.
After quick debugging I noticed the application was hanging at:

```Csharp
await process.StandardOutput.BaseStream.CopyToAsync(outputStream);
```

Took me a while to figure out what was going on. I was trying to look at examples of reading standard output stream
in .NET Core documentation and Stack Overflow. I looked at the examples and it seemed all them did the same things as
I did. Turns out I missed one key thing. None of the examples were piping in the data... There was one crucial thing
I forgot to do: I should close the standard input once I'm done piping the data in. The fix was really simple after all

```Csharp
var process = new Process
{
    StartInfo = new ProcessStartInfo
    {
        FileName = _appSettings.JavaExecutablePath,
        Arguments = $"-jar {_appSettings.PlantUMLPath} -pipe"
        RedirectStandardInput = true,
        RedirectStandardOutput = true,
    }
};


process.Start();
await userInputtedPlantUMLStream.CopyToAsync(process.StandardInput.BaseStream);

// One should not forget to close the stdin.. :)
process.StandardInput.Close();

var outputStream = new MemoryStream();
await process.StandardOutput.BaseStream.CopyToAsync(outputStream);
process.WaitForExit();
outputStream.Seek(0, SeekOrigin.Begin);
return outputStream;
```
