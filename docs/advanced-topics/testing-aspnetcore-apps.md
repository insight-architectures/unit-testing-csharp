ASP.NET Core web applications are tricky to test with classic unit tests.

While it is still possible to create fixtures that target controllers or page models, there are aspects of the web application that are difficult to mock like routing and URL generation.

Rather than creating complex setups to mimic the behavior of the web framework, it's possible to leverage the `WebApplicationFactory<TEntryPoint>` to create an instance of the application in the same address space of the test runner. Unit tests will then be able to access the web application via a `HttpClient` configured ad-hoc.

## Simple example

Here is a test fixture that tests a web application created based on the built-in Web API template.

```csharp
public class Tests
{
	private HttpClient client;

	[SetUp]
	public void Setup()
	{
		var factory = new WebApplicationFactory<Startup>();

		client = factory.CreateClient();
	}

	[Test]
	public async Task Get_WeatherForecast_returns_forecasts()
	{
		var response = await client.GetFromJsonAsync<WeatherForecast[]>("/WeatherForecast");

		Assert.That(response, Has.Exactly(5).InstanceOf<WeatherForecast>());
	}
}
```

The `WebApplicationFactory` is part of the `Microsoft.AspNetCore.Mvc.Testing` package.

## Customizing the application

In the likely case the controller under test has dependencies, the `WebApplicationFactory` can be customized with a builder delegate that allows the customization of aspects of the application setup like service registration.

Here is a version of the `WebApplicationFactory` where a service is overridden with a fake.

```csharp
var factory = new WebApplicationFactory<Startup>().WithWebHostBuilder((IWebHostBuilder builder) => 
{
    builder.ConfigureServices(services => 
    {
        services.AddSingleton<IWeatherService, MyFakeWeatherService>();
    });
});
```

Through the given `builder`, it is possible to customize the application configuration via the `ConfigureAppConfiguration` method, the logging pipeline via the `ConfigureLogging` method and all other aspects of the web application.

## Leveraging a mocking framework

Alternatively, it is possible to use mocking libraries like `Moq` to provide a dynamically generated mock of the service interface.

```csharp
public class Tests
{
    private HttpClient client;
    private Mock<IWeatherService> _weatherService;

    [SetUp]
    public void Setup()
    {
        _weatherService = new Mock<IWeatherService>();

        var factory = new WebApplicationFactory<Startup>().WithWebHostBuilder(builder => 
        {
            builder.ConfigureServices(services => 
            {
                services.AddSingleton<IWeatherService>(_weatherService.Object);
            });
        });

        client = factory.CreateClient();
    }

    [Test]
    public async Task Get_WeatherForecast_returns_forecasts_from_service()
    {
        var result = new []
        {
            new WeatherForecast { /* ... */ }
        };

        _weatherService.Setup(p => p.GetAll()).Returns(result);

        var response = await client.GetFromJsonAsync<WeatherForecast[]>("/WeatherForecast");

        Assert.That(response, Has.Exactly(result.Length).InstanceOf<WeatherForecast>());
    }
}
```