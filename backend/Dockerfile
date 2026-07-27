FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build

WORKDIR /src

COPY ./OrbitBoard.Api/OrbitBoard.Api.csproj ./
RUN dotnet restore

COPY ./ ./

RUN dotnet publish -c Release -o /app/publish /p:UseAppHost=false

FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS runtime
WORKDIR /app

COPY --from=build /app/publish ./

ENV ASPNETCORE_URLS=http://0.0.0.0:5200

EXPOSE 5200

ENTRYPOINT [ "dotnet", "OrbitBoard.Api.dll" ]