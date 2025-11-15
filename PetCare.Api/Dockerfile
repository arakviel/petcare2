# ===== SDK stage =====
FROM mcr.microsoft.com/dotnet/sdk:9.0 AS build
WORKDIR /src

# --- Встановлення утиліт ---
RUN apt-get update && \
    apt-get install -y --no-install-recommends netcat-traditional postgresql-client && \
    rm -rf /var/lib/apt/lists/*

# --- Встановлення dotnet-ef ---
RUN dotnet tool install --global dotnet-ef --version 9.0.7
ENV PATH="$PATH:/root/.dotnet/tools"

# --- Копіюємо csproj для кешу ---
COPY ["PetCare.Api/PetCare.Api.csproj", "PetCare.Api/"]
COPY ["PetCare.Application/PetCare.Application.csproj", "PetCare.Application/"]
COPY ["PetCare.Infrastructure/PetCare.Infrastructure.csproj", "PetCare.Infrastructure/"]
COPY ["PetCare.Domain/PetCare.Domain.csproj", "PetCare.Domain/"]
RUN dotnet restore "PetCare.Api/PetCare.Api.csproj"

# --- Копіюємо весь код ---
COPY . .

# --- Будуємо тільки API-проєкт (уникаємо docker-compose.dcproj) ---
RUN dotnet build PetCare.Api/PetCare.Api.csproj -c Release

# --- Публікуємо API разом з усіма залежностями (включаючи Infrastructure з міграціями) ---
RUN dotnet publish PetCare.Api/PetCare.Api.csproj \
    -c Release \
    -o /app/publish \
    /p:UseAppHost=false

# ===== Runtime stage =====
FROM mcr.microsoft.com/dotnet/aspnet:9.0 AS runtime
WORKDIR /app

RUN apt-get update && \
    apt-get install -y --no-install-recommends netcat-traditional postgresql-client && \
    rm -rf /var/lib/apt/lists/*

# --- Копіюємо dotnet-ef tools to runtime ---
COPY --from=build /root/.dotnet/tools /usr/share/dotnet/tools
ENV PATH="$PATH:/usr/share/dotnet/tools"

# --- Копіюємо результати ---
COPY --from=build /app/publish .

# --- Копіюємо скрипти очікування бази та міграцій ---
COPY --from=build /src/PetCare.Api/wait-for-db.sh /wait-for-db.sh
COPY --from=build /src/PetCare.Api/wait-for-migrations.sh /wait-for-migrations.sh
RUN chmod +x /wait-for-db.sh /wait-for-migrations.sh

# --- ENV налаштування ---
ENV ASPNETCORE_ENVIRONMENT=Development
ENV ASPNETCORE_URLS=http://+:5100

# Use the new migration script as entrypoint
ENTRYPOINT ["/wait-for-migrations.sh", "db", "5432", "dotnet", "/app/PetCare.Api.dll"]
