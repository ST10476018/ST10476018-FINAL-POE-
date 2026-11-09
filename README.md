# ST10476018-FINAL-POE-
 RaceDay — Part 3 MVC Web Application

**Student:** Siyanda Mkhize  |  **Student number:** ST10476018

## Part 3 documents and setup sequence

The complete 11-step implementation notes are in [`Documentation/POE-Part3-Implementation-Guide.md`](Documentation/POE-Part3-Implementation-Guide.md). They cover Azure Storage account setup, Azure Storage Explorer, database image properties, Blob Storage configuration, event banner upload handling, Events views, image URL display, Docker Desktop installation and running the MVC application in Docker.

To check/install local development tools on Windows, run `scripts/setup-tools.ps1` from PowerShell. It uses `winget` where available and never installs or stores your Azure credentials. See `.env.example` for configuration key names only; keep real secrets in .NET user-secrets.

---

This Part 3 implementation extends the supplied Part 2 API solution. The MVC front end uses `HttpClient` to call the API; it has no EF Core reference, SQL connection string, database context, or direct database access.

## Projects

- `RaceDay.Api` — existing Part 2 REST API, extended with Azure Blob banner upload.
- `RaceDay.Mvc` — ASP.NET Core 8 MVC front end.
- `RaceDay.Tests` — supplied Part 2 unit tests.
- `Database/Upgrade-Part3-Banner.sql` — adds the banner URL column to an existing Part 2 database.
- `Dockerfile` — containerises the MVC project.
- `.github/workflows/dotnet.yml` — builds and tests the complete solution, including MVC.

## Prerequisites

- .NET 8 SDK
- Visual Studio 2022 (ASP.NET and web development workload) or VS Code
- SQL Server / LocalDB for the Part 2 API
- Azure Storage account with a Blob container
- Docker Desktop for the container demonstration

## 1. Database update

The new API property `Event.BannerImageUrl` is stored in SQL Server. If you already have the Part 2 database, open `Database/Upgrade-Part3-Banner.sql` in SSMS and execute it against `RaceDayDb` once.

For a fresh database, `Database/RaceDaySchema.sql` now includes the new column. Follow the Part 2 README to create the database and apply EF Core migrations.

## 2. Configure Azure Blob Storage

Create an Azure Storage account and copy its connection string from **Security + networking → Access keys**. Do not commit this secret to GitHub.

From the solution root, configure it with user secrets:

```bash
dotnet user-secrets init --project RaceDay.Api
dotnet user-secrets set "AzureBlobStorage:ConnectionString" "YOUR_STORAGE_CONNECTION_STRING" --project RaceDay.Api
dotnet user-secrets set "AzureBlobStorage:ContainerName" "event-banners" --project RaceDay.Api
```

The API creates the `event-banners` container when the first image is uploaded. This sample configures blob-level public read access so event banners can be displayed by their URL. The Storage account must permit public blob access; if your organisation disables public access, implement a private-container SAS/proxy strategy instead. Never put the Storage connection string in the MVC project.

Accepted image formats: JPG, PNG and WEBP. Maximum file size: 5 MB. Uploads are authorised by the API and restricted to the organiser who owns the event.

## 3. Run the API

The default API HTTP development port is `5142` (see `RaceDay.Api/Properties/launchSettings.json`). The MVC project calls `http://localhost:5142`.

For local development, from the solution root:

```bash
dotnet restore RaceDay.sln
dotnet run --no-launch-profile --project RaceDay.Api -- --urls http://0.0.0.0:5142
```

Keep this terminal running. Configure the API database connection string using the Part 2 instructions. Ensure the Part 2 database is running and the banner SQL upgrade has been applied.

## 4. Run MVC locally

Open a second terminal:

```bash
dotnet run --project RaceDay.Mvc
```

Browse to the local URL printed by ASP.NET Core (normally `http://localhost:5xxx` or `https://localhost:7xxx`). `RaceDay.Mvc/appsettings.json` contains the API URL. Override it with `Api__BaseUrl` if your API uses another address.

Register two test accounts using the MVC registration page: one **Organizer**, one **Participant**. Passwords must meet the API's Part 2 rule: at least 8 characters, one uppercase letter, one number and one special character.

## 5. Features implemented

### Authentication and navigation
- Registration and login call `/api/auth/register` and `/api/auth/login`.
- The API session cookie is captured server-side by MVC and forwarded on later API calls.
- MVC session stores the authenticated user ID, display name and role.
- Role-specific navigation and controller authorization filters protect Organiser and Participant pages.
- Logout calls the API logout endpoint and clears the MVC session.

### Organiser
- Dashboard with owned events, enrolment totals and upcoming-event count.
- Create, edit and delete owned events.
- Upload event banner through the API to Azure Blob Storage.
- Create age/distance categories and assign them to an event.
- View event enrolments and categories.
- Record finishing time and finishing position.

### Participant
- Browse upcoming events and view event details/categories.
- Enrol by selecting a category.
- View personal enrolments and published results.

### Design
- Responsive Bootstrap layout, RaceDay navy/orange theme, common shared layout, and distinct enrolment status styling.

## 6. Docker — run the MVC application

Start the API on the host first (step 3). Build the MVC image from the solution root:

```bash
docker build -t raceday-mvc:part3 .
```

Run it:

```bash
docker run --name raceday-mvc -p 8080:8080 -e Api__BaseUrl=http://host.docker.internal:5142 raceday-mvc:part3
```

Open `http://localhost:8080`. The MVC container calls the API running on the host through `host.docker.internal`. If the API is hosted elsewhere, replace the API URL. To stop the container:

```bash
docker stop raceday-mvc
docker rm raceday-mvc
```

The API and SQL Server are separate services in this assignment setup; the Dockerfile specifically containerises the MVC application as required.

## 7. GitHub Actions

The workflow at `.github/workflows/dotnet.yml` restores, builds, and runs tests for the full `RaceDay.sln`, which now includes `RaceDay.Mvc`. Push the project to your GitHub repository and check the **Actions** tab for a successful run. Do not claim CI passed until the workflow has actually run successfully.

## 8. Suggested meaningful commit sequence

Make these commits as you complete and verify the corresponding work. Do not create empty commits.

1. `Add RaceDay MVC project to solution`
2. `Configure API HttpClient integration`
3. `Implement API session cookie forwarding`
4. `Add MVC login and registration`
5. `Add role-based session authorization`
6. `Create shared RaceDay navigation layout`
7. `Add responsive RaceDay Bootstrap theme`
8. `Build public upcoming events page`
9. `Add organiser dashboard`
10. `Implement event creation`
11. `Implement event editing`
12. `Implement event deletion`
13. `Add category creation and assignment`
14. `Add organiser enrolment listing`
15. `Implement results entry`
16. `Build participant event browsing`
17. `Implement participant enrolment`
18. `Add personal enrolment and results page`
19. `Integrate Azure Blob banner uploads in API`
20. `Add Dockerfile and container instructions`
21. `Extend GitHub Actions to build MVC`
22. `Document Part 3 setup and verification`

Example:

```bash
git checkout -b part-3-mvc
git add .
git commit -m "Add RaceDay MVC project to solution"
git push -u origin part-3-mvc
```

Only commit each milestone after making and testing the related change. The list is a development plan, not a substitute for real commits.



## Important implementation notes

1. The Part 2 API returns enrolments immediately after they are created and does not currently expose a separate status field. The MVC UI labels successfully created enrolments **Confirmed**; it does not invent a Pending workflow.
2. The API currently supports `run`, `walk`, and `cycle` event types.
3. The API category model is global, with categories linked to events. This MVC page creates a category and then assigns it to the selected event.
4. The supplied API uses in-memory distributed session storage. It is suitable for a single-instance demonstration, not multi-instance production deployment without a shared session store.
5. The code is a starting implementation built against the supplied Part 2 API contract. Run the build and end-to-end checks above in your environment before submitting. The generated project has not been compiled in this workspace because the .NET SDK is not installed here.
