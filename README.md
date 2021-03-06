# essential-interfaces-generator
Generates interfaces for the [Xamarin.Essentials](https://github.com/xamarin/Essentials) library. 

If you're after the interfaces themselves, they're currently [published to MyGet](https://www.myget.org/feed/ei/package/nuget/Essential.Interfaces) - while I make sure it's all hunky dory. 
Eventually it will move to NuGet proper, but for now: 

`Install-Package Essential.Interfaces -Version 0.8.0-preview -Source https://www.myget.org/F/ei/api/v3/index.json`

You can also grab the raw source outputs from [here](https://essential-interfaces.azurewebsites.net/) (see the end of this readme for a few more details). 

Note that like Xamarin.Essentials itself, this is in preview. As simple as the output is, I haven't touched every member on every platform yet, so I can't hand-on-heart say that I've checked it all. 

#### Why would I want this? 
The Xamarin.Essentials library is a great initiative by the Xamarin team to provide a simple, consolidated, endorsed and low-overhead set of cross-platform apis for mobile applications. As a core design decision, the Essentials library does not include any interfaces - [for several reasons](https://github.com/xamarin/Essentials/wiki/FAQ-%7C-Essentials#where-are-the-interfaces) - and all features are accessed via static methods, properties and events. If you use dependency injection in your mobile apps, you may miss the interfaces that the 'old skool' plugins for Xamarin typically shipped with - fret not, essential-interfaces-generator is here! 

#### How does it work? 
essential-interface-generator reads the Xamarin.Essentials source and generates an intermediate model representing the apis exposed by Xamarin.Essentials (`RoslynModelGenerator`). It then uses that model to generate C# code containing matching interface definitions for the apis, and implementations for interfaces that forward members on to the static classes provided by Xamarin Essentials (`ImplementationGenerator`). These are dropped into an output project with some basic version information (`ProjectMutator`) and packed for NuGet.

#### How do I use the interfaces?
Install the NuGet package and get registering! You can register all implementatations at once by checking for types implementing `IEssentialsImplementation`*, or just register the ones you care about.

For example:

```cs 
builder
    .RegisterAssemblyTypes(typeof(IEssentialsImplementation).Assembly)
    .Where(x => x.Implements<IEssentialsImplementation>())
    .AsImplementedInterfaces();
```
*`IEssentialsImplementation` is an empty 'marker' interface attached to all implementation classes.

For mocking, just work as you would with any other mocks:
```cs 
var mockGeo = new Mock<IGeocoding>();
mockGeo.Setup(x => x.GetPlacemarksAsync(It.IsAny<double>(), It.IsAny<double>()))
       .ReturnsAsync(Enumerable.Empty<Placemark>());
// etc.
``` 

#### Does it play nice with the Mono Linker? 
Yes. The generated assembly is marked with the `LinkerSafe` attribute, making it eligible for linking regardless of the link level of your consuming project. Only the types and members you invoke from your application are included in compiled outputs, even if you register them all. You can verify this using a dissassembler like dotPeek. 

#### Is reading from the source directly a bit brittle? 
Probably. The generator makes assumptions based on the current conventions in the repo. Since it is being run against each commit in the Xamarin.Essentials repo, any breaking changes should become clear quickly enough. 

#### I just want to get the raw output!
No problems, you totally can! The interfaces and implementations being generated off each commit to Xamarin.Essentials are available [here](https://essential-interfaces.azurewebsites.net/). That's a mono/WASM page so give it a few seconds to load. 

The filename for each commit is generated based on the output of `git describe --tags --long` against the Essentials repo. Essentially - in addition to the SHA - it indicates the most recent tag in the repo and the distance (in commit count) that this commit is from that tag. For example, the latest at time of writing is `essentials-0.8.0-preview-4-g3dfa546.cs`. This means the file was generated off the Essentials commit SHA `3dfa546`, which is `4` commits after the `0.8.0-preview` tag. 

Typically, you'll want to use a set of interfaces that exactly matches a commit tag - i.e. a distance of `0` - as there is no guarantee that any given commit since the last release is compatible with the released Xamarin.Essentials package - the API may have changed or include new members that weren't available in the previous release.
