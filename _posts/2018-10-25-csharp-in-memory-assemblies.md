---
title:  "C# In Memory Assemblies"
date:   2018-10-25 12:22:23
categories: [C#]
tags: [Tradecraft]
---
2018 appears to be the year that Red Teams are making a major shift in tradecraft. PowerShell is still very popular but it is losing steam in mature organizations thanks to additional optics such as additional logging and AMSI. In the last organization I worked for, just starting PowerShell would cause an analyst to investigate why the user launched it. Ultimately, PowerShell is not required as it’s true power (pun intended) comes from being able to access .Net. This is where C# comes into play.

## PowerShell to CSharp

Most of the community who started the offensive PowerShell tradecraft have been transitioning and creating new tools to C# as it provides full access to .Net. The ability to have a compiled .Net assembly can be useful as there are many ways to load and execute a .Net assembly (e.g.DotNetToJScript, Cobalt Strike’s execute-assembly). .Net assemblies also have the ability to load and execute other .Net assemblies thanks to reflection. One popular example is creating a stager that downloads, loads and executes the main functionality of your toolset. Another example is loading additional functionality into your toolset in memory to keep the size down. You could load any of the awesome SpectreOps GhostPack tools or cobbr’s SharpSploit. Did I mention all of this is happens in memory? Let’s walk through the steps to create our own C# assembly that will load some of this awesome tooling.

We will use Seatbelt from the GhostPack project for our example. A quick download from Github and we have everything we need. I used Visual Studio 2017 to compile the project with no issues. Once we have it compiled, we can run Seatbelt.exe to verify it is working as it should.

![seatbelt.exe](/images/posts/csharp-in-memory-assemblies/seatbelt.jpg)  
Executing Seatbelt.exe

Next, we need to build an assembly that will load Seatbelt.exe into our application. Below is a sample of code that will load the assembly from disk, find the entrypoint or “Main” function and execute it with the specified parameters. Since Seatbelt is a console application, the passed parameters are the same as if we were typing it on the commandline. We will pass the “user” argument to run all “UserChecks” from Seatbelt.

{% highlight csharp %}
using System;
using System.IO;
using System.Reflection;

namespace AssemblyLoader
{
    class Program
    {
        static void Main(string[] args)
        {
            //Read the assembly from disk into a byte array
            Byte[] bytes = File.ReadAllBytes(@"C:\ProgramData\MTA\Dev\Seatbelt.exe");
            ExecuteAssembly(bytes, new string[] { "user" });

            Console.Write("Press any key to exit");
            string input = Console.ReadLine();
        }

        //Load and execute an assembly and pass cmdline parameters
        public static void ExecuteAssembly(Byte[] assemblyBytes, string[] param)
        {
            //Load the assembly
            Assembly assembly = Assembly.Load(assemblyBytes);
            //Find the Entrypoint or "Main" method
            MethodInfo method = assembly.EntryPoint;
            //Get Parameters
            object[] parameters = new[] { param };
            //Invoke the method with the specified parameters
            object execute = method.Invoke(null, parameters);
        }
    }
}
{% endhighlight %}

![assemblyloader.exe](/images/posts/csharp-in-memory-assemblies/seatbelt-user.jpg)  
Assembly loading and executing Seatbelt.exe with the “user” cmdline argument

This is great, we now have the ability to load and execute a .Net assembly into memory and execute it. But what happens if we do not have a console application and we are working with a DLL. We can call a method from a .Net assembly (EXE or DLL) without invoking the “main” method.

Using our already compiled Seatbelt assembly, we can call a specific method, provide the required parameters and execute it. The "UserCheck" method collects a lot of interesting information but what if we wanted just a small portion of that information. One of the methods called by "UserCheck" is "ListRecentRunCommands". Looking through the Seatbelt code, we can see this function on line 6101 and that there are not any required arguments. We can tweak our existing assembly loader to execute just the “ListRecentRunCommands” method.

{% highlight csharp %}
using System;
using System.IO;
using System.Reflection;

namespace AssemblyLoader
{
    class Program
    {
        static void Main(string[] args)
        {
            //Read the assembly from disk into a byte array
            Byte[] bytes = File.ReadAllBytes(@"C:\ProgramData\MTA\Dev\Seatbelt.exe");
            ExecuteAssemblyMethod(bytes, "Seatbelt.Program", "ListRecentRunCommands", null);

            Console.Write("Press any key to exit");
            string input = Console.ReadLine();
        }

        //Load an assembly and execute a method and pass required parameters
        public static void ExecuteAssemblyMethod(Byte[] assemblyBytes, string typeName, string methodName, object[] parameters)
        {
            //Load the assembly
            Assembly assembly = Assembly.Load(assemblyBytes);
            //Find the type (Namespace and class) containing the method
            Type type = assembly.GetType(typeName);
            //Create an instance of the type using a constructor
            object instance = Activator.CreateInstance(type);
            //Select the method to be called
            MethodInfo method = type.GetMethod(methodName);
            //Invoke the method from the instanciated type with the specified parameters
            object execute = method.Invoke(type, parameters);
        }
    }
}
{% endhighlight %}

Now that we have our code, we can build our assembly and test it out.

![assemblyloader.exe](/images/posts/csharp-in-memory-assemblies/seatbelt-recentruncommands.jpg)  
Assembly loading and executing Seatbelt.exe’s “ListRecentRunCommands” method

We have again successfully loaded a .Net assembly into memory and executed a method. This can be used for writing C# droppers/stagers or integrated into your own tooling.

## Defense

From a defensive perspective, there is little insight into the world of .Net. previous posts by SpectreOps and the infosec community have highlighted this. Application whitelisting is usually thrown around as a solution but for most large enterprises, this is an extremely difficult process. One thing that red teamers and attackers will not be able to avoid is calling CSC.exe when loading an assembly. When calling the System.Reflection.Assembly.Load() method, CSC.exe is called and compiles a temporary DLL in the temp directory that is then deleted immediately. This has to do with .Net assemblies being Just-In-Time compiled. CSC.exe is a tough indicator as it can be extremely noisy. However, some EDR products have the ability to examine these DLLs as they touch disk. There is a chance it could be flagged during that time and the assembly would fail to load.

One cool project that looks to address identifying this is Endgame's ClrGuard. This can monitor or block any process that attempts to loads the CLR/.Net runtime. Depending on the environment and the amount of .Net applications, this could be something worth investing time into but it could also log or block a lot of legitimate applications utilizing the .Net runtime.

This covers the basics of using reflection to load and execute .Net assemblies. None of these techniques are really that new, developers have been using the reflection to load assemblies for application plugins and other uses for a while. These techniques will be useful for red teamers and attackers until we get some optics into .Net. Thanks for reading.

## References

SpectreOps' GhostPack: <https://github.com/GhostPack>  
Cobbr's SharpSploit:   <https://github.com/cobbr/SharpSploit>  
Endgame's ClrGuard:    <https://github.com/endgameinc/ClrGuard>
