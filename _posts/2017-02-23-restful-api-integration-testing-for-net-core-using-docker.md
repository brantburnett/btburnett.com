---
id: 252
title: Restful API Integration Testing For .NET Core Using Docker
date: 2017-02-23T19:37:06+00:00
guid: http://btburnett.com/?p=252
permalink: /2017/02/restful-api-integration-testing-for-net-core-using-docker.html
categories:
  - .NET Core
  - Docker
---
I love unit tests. There's nothing quite like writing a class and feeling 100% confident it will work as described because the tests are all passing. But integration testing is also important. Sometimes I need to test the full stack and make sure it works as described.

For a recent project I have been creating [.NET Core](https://www.microsoft.com/net/core) RESTful microservices. Along with these services I have been creating client SDKs that abstract away the RESTful requests. The client SDK is then published via an internal Nuget server. This makes it easy for services in the architecture to communicate with each other using the SDK rather than using HttpClient. For easy organization and version control, the SDK is located in the same solution as the service.

The question that followed quickly was "How do I test this SDK?" Unit tests can help cover some of the functionality, but they don't test the full stack from an SDK consumer request through HTTP to the actual service. I could make a test application that uses the SDK to call the service, but this would be cumbersome.

What if I could write tests using a standard test framework like xUnit.net or NUnit? Such tests could be easily executed in Visual Studio or even in a continuous integration step. But if I use this kind of test framework, how do I easily make sure that the latest version of the service is up and running so my client SDK tests can use it? And what about other services called by the service under test?

## Enter Docker

[Docker](https://www.docker.com/) is a great tool that serves many purposes in the development pipeline, and since the release of .NET Core I've starting to fall in love with it. Integration testing is another great use for Docker. Using Docker it's possible to launch the service being tested in a container (along with any dependencies) as part of the client SDK integration tests. After the tests are complete, the containers are stopped and the resources are freed.

## Preparing The Service

The first step is to make sure the service can be started as a Docker container. Here's a brief summary of the steps I followed:

1. [Ensure that Hyper-V is enabled in Windows](https://docs.microsoft.com/en-us/virtualization/hyper-v-on-windows/quick-start/enable-hyper-v)
2. Install [Docker for Windows](https://docs.docker.com/docker-for-windows/)
3. [Configure a Shared Drive in Docker](https://docs.docker.com/docker-for-windows/#/shared-drives) for the drive where the application lives
4. Install [Visual Studio Tools for Docker](https://marketplace.visualstudio.com/items?itemName=MicrosoftCloudExplorer.VisualStudioToolsforDocker-Preview)
5. Ensure that Docker is started (it can be configured to autostart on login)
6. Right click on the project in Visual Studio and Add Docker Support ![Screenshot](https://microsoftcloudexplorer.gallerycdn.vsassets.io/extensions/microsoftcloudexplorer/visualstudiotoolsfordocker-preview/0.41.0/1482142258056/205468/1/add-docker-support.png "Add Docker Support in Visual Studio")

## Configuring Docker Compose

When running on a development machine from within Visual Studio, a [Docker Compose](https://docs.docker.com/compose/) file (docker-compose.yml) is used to control the Docker containers which are created. The default file is a great starting point, but it will require some tweaking:

```yaml
version: '2'

services:
  myservice:
    image: user/myservice${TAG}
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "80"
```

The first thing to do is change to a static port mapping. This will simplify accessing the service from the tests. By changing the port definition from "80" to "8080:80" the tests will be able to access the service on port 8080. Of course, any unused port will work.

```yaml
version: '2'

services:
  myservice:
    image: user/myservice${TAG}
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8080:80"
```

Next, the file needs to be updated to deploy additional dependent services. This could get rather complicated if there are a lot of dependencies, but here's an example of adding a single dependency with a link from the service.

```yaml
version: '2'

services:
  myservice:
    image: user/myservice${TAG}
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8080:80"
    links:
      - mydependency
  mydependency:
    image: user/mydependency:latest
    expose:
      - "80"
```

Now when the service is started, it will also start the "mydependency" service. It will be accessible at http://mydependency/ from within "myservice". Of course, how the dependencies communicate with each other can be adjusted depending on the architecture. The "image:" value should also be adjusted to refer to the correct Docker registry where the dependency is hosted.

## Overriding With Test Specific Configuration

The settings in docker-compose.yml are generic and used to define the basic configuration for running the service. Additionally, docker-compose.dev.debug.yml and docker-compose.dev.release.yml provide overrides specific to running in Debug and Release mode in Visual Studio.

However, these configurations don't start the service in the way most containers are started. They start the container as an executable that does nothing and never exits: "tail -f /dev/null". Then the service is run out of band using "docker exec". The container doesn't even contain the service, it's just reading the files from the host hard drive using Docker volumes. This is great for debugging in Visual Studio, but I found it problematic for running automated integration tests.

To address this, create an additional YAML file, docker-compose.test.yml, in the root of the service project. This file overrides the build context path so that it collects the service from the publication directory (which is created by using "dotnet publish"). It can also configure environment variables within the container which can override the default ASP.NET Core configuration from appsettings.json.

```yaml
version: '2'

services:
  myservice:
    build:
      context: bin/${CONFIGURATION}/netcoreapp1.0/publish
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
```

## Starting The Service When Running Tests

To start the container, some commands should be run during test startup. How these commands are run will vary depending on the test framework, but the basic list is:

1. Run "dotnet publish" against the service to build and publish the application
2. Run "docker-compose build" to build an up-to-date image for the application
3. Run "docker-compose up" to start the containers
4. After tests are complete, run "docker-compose down" to shutdown and remove the containers

If xUnit.net is being used as the test framework, this can be done using a collection fixture. First, define a test collection:

```cs
using System;
using Xunit;

namespace IntegrationTests
{
    [CollectionDefinition("ClientTests")]
    public class ClientTestCollection : ICollectionFixture<ServiceContainersFixture>
    {
    }
}
```

Note above the ClientTestCollection class is implementing ICollectionFixture<ServiceContainersFixture>. There must also be a definition for the referenced ServiceContainersFixture. This class will be created once for all tests in the test collection, and then disposed when they are complete.

> Note: The example below assumes that "dotnet" and "docker-compose" are accessible in the system PATH. They should be by default.

```cs
using System;
using System.Collections.Generic;
using System.Diagnostics;
using System.Linq;
using System.Net.Http;
using System.Threading;
using System.Threading.Tasks;
using Xunit;

namespace IntegrationTests
{
    public class ServiceContainersFixture : IDisposable
    {
        // Name of the service
        private const string ServiceName = "myservice";

        // Relative path to the root folder of the service project.
        // The path is relative to the target folder for the test DLL,
        // i.e. /test/MyTests/bin/Debug
        private const string ServicePath = "../../../../src/MyService";

        // Tag used for ${TAG} in docker-compose.yml
        private const string Tag = "test";

        // This URL should return 200 once the service is up and running
        private const string TestUrl = "http://localhost:8080/myservice/ping";

        // How long to wait for the test URL to return 200 before giving up
        private static readonly TimeSpan TestTimeout = TimeSpan.FromSeconds(60);

#if DEBUG
        private const string Configuration = "Debug";
#else
        private const string Configuration = "Release";
#endif

        public ApplicationFixture()
        {
            Build();

            StartContainers();

            var started = WaitForService().Result;

            if (!started)
            {
                throw new Exception($"Startup failed, could not get '{TestUrl}' after trying for '{TestTimeout}'");
            }
        }

        private void Build()
        {
            var process = Process.Start(new ProcessStartInfo
            {
                FileName = "dotnet",
                Arguments = $"publish {ServicePath} --configuration {Configuration}"
            });

            process.WaitForExit();
            Assert.Equal(0, process.ExitCode);
        }

        private void StartContainers()
        {
            // First build the Docker container image

            var processStartInfo = new ProcessStartInfo
            {
                FileName = "docker-compose",
                Arguments =
                    $"-f {ServicePath}/docker-compose.yml -f {ServicePath}/docker-compose.test.yml build"
            };
            AddEnvironmentVariables(processStartInfo);

            var process = Process.Start(processStartInfo);

            process.WaitForExit();
            Assert.Equal(0, process.ExitCode);

            // Now start the docker containers

            processStartInfo = new ProcessStartInfo
            {
                FileName = "docker-compose",
                Arguments =
                    $"-f {ServicePath}/docker-compose.yml -f {ServicePath}/docker-compose.test.yml -p {ServiceName} up -d"
            };
            AddEnvironmentVariables(processStartInfo);

            process = Process.Start(processStartInfo);

            process.WaitForExit();
            Assert.Equal(0, process.ExitCode);
        }

        private void StopContainers()
        {
            // Run docker-compose down to stop the containers
            // Note that "--rmi local" deletes the images as well to keep the machine clean
            // But it does so by deleting all untagged images, which may not be desired in all cases

            var processStartInfo = new ProcessStartInfo
            {
                FileName = "docker-compose",
                Arguments =
                    $"-f {ServicePath}/docker-compose.yml -f {ServicePath}/docker-compose.test.yml -p {ServiceName} down --rmi local"
            };
            AddEnvironmentVariables(processStartInfo);

            var process = Process.Start(processStartInfo);

            process.WaitForExit();
            Assert.Equal(0, process.ExitCode);
        }

        private void AddEnvironmentVariables(ProcessStartInfo processStartInfo)
        {
            processStartInfo.Environment["TAG"] = Tag;
            processStartInfo.Environment["CONFIGURATION"] = Configuration;
            processStartInfo.Environment["COMPUTERNAME"] = Environment.MachineName;
        }

        private async Task<bool> WaitForService()
        {
            using (var client = new HttpClient() { Timeout = TimeSpan.FromSeconds(1)})
            {
                var startTime = DateTime.Now;
                while (DateTime.Now - startTime < TestTimeout)
                {
                    try
                    {
                        var response = await client.GetAsync(new Uri(TestUrl)).ConfigureAwait(false);
                        if (response.IsSuccessStatusCode)
                        {
                            return true;
                        }
                    }
                    catch
                    {
                        // Ignore exceptions, just retry
                    }

                    await Task.Delay(1000).ConfigureAwait(false);
                }
            }

            return false;
        }

        public void Dispose()
        {
            StopContainers();
        }
    }
}
```

> **Note:** Before each call to docker-compose, AddEnvironmentVariables is being called to set some environment variables in ProcessStartInfo. These are used by docker-compose to perform substitutions in the YAML file. For example, ${COMPUTERNAME} will be replaced with the name of the development computer. This could be easily extended with other environment variables as needed.
{: .notice--info}

To specify that a test class needs the containers to be running, apply the Collection attribute to the test class. Note how the string "ClientTests" matches the CollectionDefinition attribute used earlier on ClientTestCollection.

```cs
[Collection("ClientTests")]
public class MyTests
{
    // Tests here...
}
```

## Conclusion

It's now possible to run .NET Core integration tests that depend on running services directly in Visual Studio using your test runner of choice (I use Resharper). It will build the .NET Core service, start it and its dependencies in Docker, run the integration tests, and then perform cleanup. This could also be done as part of a continuous integration process, so long as the test machines have Docker installed and access to any required Docker registries. Best of all, this simultaneously tests both the service and the client SDK, exposing problems in either.
