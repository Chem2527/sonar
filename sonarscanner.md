# SonarQube & Trivy Scanner Setup and Execution Guide

This guide documents the setup, execution, and automation scripts used to run **Trivy dependency vulnerability scans** and **SonarQube static code analysis (SAST)** across all 12 Zenus / Fintech repositories.

---

## 1. Prerequisites Setup

### Java Development Kit (JDK 25)
SonarQube requires Java to execute.
1. Downloaded and installed **OpenJDK 25 (LTS)** for Windows x64.
2. Verified Java installation:
   ```powershell
   java -version
   # Output verified: OpenJDK 64-Bit Server VM
   ```

### SonarQube Server Installation
1. Downloaded **SonarQube Community Edition** (ZIP file).
2. Extracted the files to `C:\Users\nxtge\Downloads\sonarqube-26.6.0.123539`.
3. Started the SonarQube local server by running:
   ```powershell
   cd "C:\Users\nxtge\Downloads\sonarqube-26.6.0.123539\sonarqube-26.6.0.123539\bin\windows-x86-64"
   .\StartSonar.bat
   ```
4. Accessed the local dashboard at `http://localhost:9000` (Default Credentials: `admin`/`admin` -> updated on first login).

---

## 2. SonarScanner CLI Installation

To run local scans on JavaScript, TypeScript, and Python projects:
1. Installed the **SonarScanner CLI** globally via npm:
   ```powershell
   npm install -g sonar-scanner
   ```
2. Verified the installation:
   ```powershell
   sonar-scanner --version
   ```

---

## 3. Automated Scanning of 12 Repositories

The 12 Zenus / Fintech repositories are located under `C:\Users\nxtge\Downloads\zenus`:
1. `fintech_admin_webapp`
2. `fintech_retail_webapp`
3. `fintech_webapp`
4. `fintech_business_management`
5. `fintech_business_settings`
6. `fintech_documents_management`
7. `fintech_notifications_management`
8. `fintech_super_admin`
9. `fintech_user_management`
10. `fintech_management_migrations`
11. `fintech_statement_generator`
12. `fintech_processing_scripts`

### Solution A: Full Scan Command (Command Prompt / Batch Script)
To scan the entire codebase of all 12 repositories, the batch script below sets up the Java environment variables and loops through each repository to trigger the scanner.

**File:** `run_all_sonar.bat`
```batch
@echo off
set "JAVA_HOME=C:\Program Files\Eclipse Adoptium\jdk-25.0.3.9-hotspot"
set "PATH=%JAVA_HOME%\bin;%PATH%"
set "BASE_DIR=C:\Users\nxtge\Downloads\zenus"
set "SONAR_TOKEN=your_sonar_token_here"
set "SONAR_URL=http://localhost:9000"

set repos[0]=fintech_admin_webapp
set repos[1]=fintech_retail_webapp
set repos[2]=fintech_webapp
set repos[3]=fintech_business_management
set repos[4]=fintech_business_settings
set repos[5]=fintech_documents_management
set repos[6]=fintech_notifications_management
set repos[7]=fintech_super_admin
set repos[8]=fintech_user_management
set repos[9]=fintech_management_migrations
set repos[10]=fintech_statement_generator
set repos[11]=fintech_processing_scripts

for /L %%i in (0,1,11) do (
    call :scan_repo %%repos[%%i]%%
)
goto :eof

:scan_repo
set "repo=%1"
set "repo_path=%BASE_DIR%\%repo%"

echo ==================================================
echo SonarQube Scanning: %repo%
echo ==================================================

if not exist "%repo_path%" (
    echo Directory %repo_path% does not exist! Skipping.
    goto :eof
)

cd /d "%repo_path%"

sonar-scanner ^
  -Dsonar.host.url=%SONAR_URL% ^
  -Dsonar.token=%SONAR_TOKEN% ^
  -Dsonar.projectKey=%repo% ^
  -Dsonar.projectName=%repo% ^
  -Dsonar.sources=. ^
  -Dsonar.exclusions=**/node_modules/**,**/build/**,**/coverage/**,**/.next/**,**/dist/**,**/venv/**,**/env/**,**/.git/**

echo Done scanning: %repo%
goto :eof
```

---

### Solution B: Optimized Scan (PowerShell Script)
To avoid scanning thousands of untouched files (like libraries or production frameworks), we created an optimized PowerShell script that runs git diffs to only scan files that were modified or newly created.

**File:** `run_all_sonar.ps1`
```powershell
# Setup path & environment
$env:JAVA_HOME = "C:\Program Files\Eclipse Adoptium\jdk-25.0.3.9-hotspot"
$env:Path += ";$env:JAVA_HOME\bin"

$baseDir = "C:\Users\nxtge\Downloads\zenus"
$sonarToken = "your_sonar_token_here"
$sonarUrl = "http://localhost:9000"

$repos = @(
    "fintech_admin_webapp", "fintech_retail_webapp", "fintech_webapp",
    "fintech_business_management", "fintech_business_settings", "fintech_documents_management",
    "fintech_notifications_management", "fintech_super_admin", "fintech_user_management",
    "fintech_management_migrations", "fintech_statement_generator", "fintech_processing_scripts"
)

foreach ($repo in $repos) {
    $repoPath = Join-Path $baseDir $repo
    Write-Host "=================================================="
    Write-Host "SonarQube Scanning: $repo"
    Write-Host "=================================================="

    if (-not (Test-Path $repoPath)) {
        Write-Warning "Directory $repoPath does not exist! Skipping."
        continue
    }

    Push-Location $repoPath

    # 1. Gather all modified/added files compared to target branches
    $diffFiles = $null
    $baseBranches = @("origin/sandbox-qa", "origin/staging", "origin/main", "HEAD~5")
    foreach ($branch in $baseBranches) {
        $diffFiles = git diff --name-only $branch 2>$null
        if ($LASTEXITCODE -eq 0 -and $diffFiles) {
            break
        }
    }
    
    # 2. Gather untracked or unstaged files
    $untracked = git status --porcelain | ForEach-Object { 
        if ($_.Length -gt 3) { $_.Substring(3) } 
    }

    # Combine lists and filter out directories/deleted files
    $allFiles = ($diffFiles + $untracked) | Select-Object -Unique | Where-Object { 
        $_ -ne $null -and $_ -ne "" -and (Test-Path $_) -and -not (Test-Path $_ -PathType Container)
    }

    if ($allFiles.Count -eq 0) {
        Write-Host "No modified files detected in $repo. Skipping Scan."
        Pop-Location
        continue
    }

    # Join files with commas
    $sources = $allFiles -join ","
    Write-Host "Scanning files: $sources"

    # Create temporary properties file
    $propertiesContent = @(
        "sonar.host.url=$sonarUrl",
        "sonar.token=$sonarToken",
        "sonar.projectKey=$repo",
        "sonar.projectName=$repo",
        "sonar.sources=$sources",
        "sonar.exclusions=**/node_modules/**,**/build/**,**/coverage/**,**/.next/**,**/dist/**,**/venv/**,**/env/**,**/.git/**"
    )

    $propertiesPath = Join-Path $repoPath "sonar-project.properties"
    $propertiesContent | Out-File -FilePath $propertiesPath -Encoding utf8 -Force

    try {
        # Run sonar-scanner
        sonar-scanner
    } finally {
        # Cleanup properties file immediately so it never gets committed to git
        if (Test-Path $propertiesPath) {
            Remove-Item $propertiesPath -Force
            Write-Host "Cleaned up temporary properties file for $repo."
        }
    }

    Pop-Location
    Write-Host "Done scanning: $repo"
}
Write-Host "All repository scans complete!"
```

---

## 4. Trivy Security Scanning Setup & Automation

To run dependency checks and container audits on all 12 repositories:
1. Downloaded **Trivy v0.72.0** for Windows x64.
2. Extracted to `C:\Users\nxtge\Downloads\trivy_0.72.0_windows-64bit`.
3. Created a PowerShell automation script `scan_repos.ps1` to loop through repositories and save reports:

**File:** `scan_repos.ps1`
```powershell
$trivyPath = "C:\Users\nxtge\Downloads\trivy_0.72.0_windows-64bit\trivy.exe"
$baseDir = "C:\Users\nxtge\Downloads\zenus"
$reportsDir = "C:\Users\nxtge\Downloads\trivy_reports"

if (-not (Test-Path $reportsDir)) {
    New-Item -ItemType Directory -Path $reportsDir -Force
}

$repos = @(
    "fintech_admin_webapp", "fintech_retail_webapp", "fintech_webapp",
    "fintech_business_management", "fintech_business_settings", "fintech_documents_management",
    "fintech_notifications_management", "fintech_super_admin", "fintech_user_management",
    "fintech_management_migrations", "fintech_statement_generator", "fintech_processing_scripts"
)

foreach ($repo in $repos) {
    $repoPath = Join-Path $baseDir $repo
    Write-Host "Scanning repository: $repo"
    
    if (-not (Test-Path $repoPath)) {
        Write-Warning "Directory $repoPath does not exist! Skipping."
        continue
    }

    $reportPath = Join-Path $reportsDir "${repo}_trivy.json"

    # Run Trivy scan skipping library & build folders
    & $trivyPath fs --skip-dirs "node_modules,.next,dist,build,venv,env,.git" --scanners vuln --severity HIGH,CRITICAL --format json --output $reportPath $repoPath
}
Write-Host "All scans completed successfully!"
```

### Remediated Vulnerabilities Found During Scan:
* **`fintech_processing_scripts` (Python):** `cryptography==44.0.2` was upgraded to `>=48.0.1` to resolve CVE-2026-26007.
* **`fintech_user_management` (Node/Express):** `multer==2.1.1` was upgraded to `^2.2.0` to resolve CVE-2026-5079.

---

## 5. Keep Git Commits Clean
To avoid polluting branches with scanning binaries, temporary configs, and reports, we:
* Added `*.properties` and `/trivy_reports/` to local gitignores.
* Automatically deleted temporary `sonar-project.properties` files inside scripts right after scanning completed.
