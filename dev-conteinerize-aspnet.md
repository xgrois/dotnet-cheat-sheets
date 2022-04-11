# ASP .NET Web API as a container

## Default webapi template

Create a new webapi project from the template:

```
> dotnet new webapi -n Weather.Api
```

Open the project with VS Code:

```
> cd Weather.Api
> code .
```

Accept the VS Code suggestion to add build and debug assets (this will add a .vscode folder).

Add a `.dockerignore` file to your project with the following content:

```
bin/
obj/
```

Add a `Dockerfile` document:

```
# https://hub.docker.com/_/microsoft-dotnet
FROM mcr.microsoft.com/dotnet/sdk:6.0 AS build
WORKDIR /source

# copy csproj and restore as distinct layers
COPY *.csproj .
RUN dotnet restore

# copy everything else and build app
COPY . .
WORKDIR /source
RUN dotnet publish -c release -o /app --no-restore

# final stage/image
FROM mcr.microsoft.com/dotnet/aspnet:6.0
WORKDIR /app
COPY --from=build /app .
ENTRYPOINT ["dotnet", "Weather.Api.dll"]
```

Build the container image:

```
> docker build -t weatherapi .
```

Launch the docker image as a container:

-   `--rm` automatically remove the container when it exits.
-   `-p 5000:80` is needed for you to check/expose your API running in the container. Note that containers are isolated from your S.O. like a VM. Note that ports in `launchSettings.json` are ignored.
-   `--it` When you docker run with this command it takes you straight inside the container. So if you do a Ctrl + C you will stop and remove the container (`--rm`). Use `-d` tag instead of `--it` if you want to just launch the container in the background.
-   You have to indicate which docker image you are going to launch.
-   You can specify the name of the container.

```
> docker run -it --rm -p 5000:80 --name aspnetcore_sample weatherapi
```

Now you can test your API running in the container with curl/Postman/Insomnia:

```
curl --request GET \
  --url http://localhost:5000/weatherforecast
```

Note that the default ASP .NET web API also exposes HTTPS endpoint and redirects all HTTP traffic to HTTPS. However, you are only mapping to por 80 and not to port 443. Thus, in your API logs you will see:

```
{"EventId":3,"LogLevel":"Warning","Category":"Microsoft.AspNetCore.HttpsPolicy.HttpsRedirectionMiddleware","Message":"Failed to determine the https port for redirect.","State":{"Message":"Failed to determine the https port for redirect.","{OriginalFormat}":"Failed to determine the https port for redirect."}}
```

### Make HTTPS to work

Generate cert and configure local machine:

```console
dotnet dev-certs https -ep $env:USERPROFILE\.aspnet\https\Weather.Api.pfx -p 123456
dotnet dev-certs https --trust
```

> Note: The certificate name, in this case _Weather.Api_.pfx must match the project assembly name.

> Note: `123456` is used as a stand-in for a password of your own choosing.

> Note: If console returns "A valid HTTPS certificate is already present.", a trusted certificate already exists in your store. It can be exported using MMC Console.

> Note: Self-signed certification file is stored in your computer in `$env:USERPROFILE\.aspnet\https`

Configure application secrets, for the certificate. First, add this to your .csproj file:

```
<Project Sdk="Microsoft.NET.Sdk.Web">

  <PropertyGroup>
    <TargetFramework>net6.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <UserSecretsId>mysecretforweatherapi-some-guid</UserSecretsId>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Swashbuckle.AspNetCore" Version="6.2.3" />
  </ItemGroup>

</Project>
```

Then, assing the values you want to that secret (stored in you computer):

```console
dotnet user-secrets -p Weather.Api.csproj set "Kestrel:Certificates:Development:Password" "123456"
```

> Note: The password must match the password used for the certificate.

> Note: User secrets file is stored in your computer in `$env:APPDATA\microsoft\UserSecrets\`

Build a container image:

```console
docker build --pull -t weatherapi .
```

Run the container image with ASP.NET Core configured for HTTPS:

```console
docker run --rm -it -p 8000:80 -p 8001:443 -e ASPNETCORE_URLS="https://+;http://+" -e ASPNETCORE_HTTPS_PORT=8001 -e ASPNETCORE_ENVIRONMENT=Development -v $env:APPDATA\microsoft\UserSecrets\:/root/.microsoft/usersecrets -v $env:USERPROFILE\.aspnet\https:/root/.aspnet/https/ weatherapi
```

> Note we include the second port mapping for HTTPS (443), and several environment variables. The user secret file and the self-signed certificate stored in your computer will be "seen" by the container. This is explicitely defined with the volume tag `-v`

> You might ask yourself where ASPNETCORE_URLS, ASPNETCORE_HTTPS_PORT and ASPNETCORE_ENVIRONMENT come from. They are specific env variables that .NET understands so you do not need to do anything.

After the application starts, navigate to `http://localhost:8000` in your web browser. Note both HTTP and HTTPS endpoins are functional. If you use the HTTP endpoint in the browser you will be redirected to HTTPS. You can try both endpoints with Postman/Insomnia.

You are done! You have your 100% functional ASP .NET Web API now running in a container.

## Use docker-compose as alternative

This approach just need the `Dockerfile` and the `docker-compose.yml` files.

Instead of using docker CLI with all those parameters, you run docker-compose CLI with those parameters in a file. This is cleaner. Docker compose has much more to offer, specially when you want to define how to launch more than one container (e.g., your API in one container and a DB in another one).

Create a `docker-compose.yml` file in your directory:

```
version: "3.9"
services:
    weatherapi:
        build: .
        ports:
            - "8000:80"
            - "8001:443"
        environment:
            ASPNETCORE_URLS: "https://+;http://+"
            ASPNETCORE_HTTPS_PORT: "8001"
            ASPNETCORE_ENVIRONMENT: Development
        volumes:
            - ${APPDATA}\microsoft\UserSecrets\:/root/.microsoft/usersecrets
            - ${USERPROFILE}\.aspnet\https:/root/.aspnet/https/
```

Build and run:

```console
> docker-compose build
> docker-compose up
```
