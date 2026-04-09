# SonarQube/SonarCloud Complete Guide

## 📌 What is SonarQube?

**SonarQube** is a **static code analysis tool** that scans your source code to find:
- 🔴 **Bugs** - Logic errors that will likely cause failures
- 🟡 **Vulnerabilities** - Security issues (SQL injection, XSS, etc.)
- 🟠 **Code Smells** - Poor coding practices (long methods, code duplication)
- 📊 **Code Coverage** - Test coverage metrics
- 🎯 **Maintainability** - Overall code quality score

**SonarCloud** = SonarQube as a cloud service (no installation needed)

---

## 🚀 Getting Started with SonarCloud (Recommended)

### **Step 1: Create SonarCloud Account**
1. Go to [sonarcloud.io](https://sonarcloud.io)
2. Sign up with your GitHub account
3. Authorize GitHub access

### **Step 2: Create Projects in SonarCloud**

For your **Order Service**:
1. Click "Create project" 
2. Select your repository
3. Select "With GitHub Actions" setup method
4. Choose "Other CI"
5. Note your **Project Key** (e.g., `rohit_order-service`)
6. Note your **Organization Key** (e.g., `rohit-org`)

Repeat for **Payment Service** with a different Project Key.

### **Step 3: Generate SonarCloud Token**

1. Go to [Settings → Security](https://sonarcloud.io/account/security)
2. Click "Generate token"
3. Name it: `GITHUB_SONAR_TOKEN`
4. Copy the token (save it safely!)

### **Step 4: Add GitHub Secrets**

In your GitHub repository:
1. Go to **Settings → Secrets and Variables → Actions**
2. Add these secrets:

```
SONAR_TOKEN = <your-sonarcloud-token>
SONAR_ORG = <your-organization-key>
SONAR_PROJECT_KEY_ORDER = <order-service-project-key>
SONAR_PROJECT_KEY_PAYMENT = <payment-service-project-key>
```

---

## 🔧 Local SonarQube Setup (Using Docker)

### **Option: Run SonarQube Locally**

Add to `docker-compose.yml`:

```yaml
sonarqube:
  image: sonarqube:lts-community
  ports:
    - "9000:9000"
  environment:
    SONAR_JDBC_URL: jdbc:postgresql://db:5432/sonar
    SONAR_JDBC_USERNAME: sonar
    SONAR_JDBC_PASSWORD: sonar
  depends_on:
    - db
  volumes:
    - sonarqube_data:/opt/sonarqube/data
    - sonarqube_logs:/opt/sonarqube/logs

db:
  image: postgres:15
  environment:
    POSTGRES_DB: sonar
    POSTGRES_USER: sonar
    POSTGRES_PASSWORD: sonar
  volumes:
    - pg_data:/var/lib/postgresql/data

volumes:
  sonarqube_data:
  pg_data:
```

Start it:
```bash
docker-compose up -d sonarqube db
```

Access at: http://localhost:9000 (default: admin/admin)

---

## 🔍 Running SonarQube Scans Locally

### **Option 1: Using Maven Plugin (Recommended for Java)**

```bash
cd MicroServices/order-service

# Against SonarCloud
mvn verify sonar:sonar \
  -Dsonar.projectKey=rohit_order-service \
  -Dsonar.organization=rohit-org \
  -Dsonar.host.url=https://sonarcloud.io \
  -Dsonar.login=<your-sonarcloud-token>

# Or against local SonarQube
mvn verify sonar:sonar \
  -Dsonar.projectKey=order-service \
  -Dsonar.host.url=http://localhost:9000 \
  -Dsonar.login=<local-admin-token>
```

### **Option 2: Using SonarScanner CLI**

```bash
# Install SonarScanner
npm install -g sonarqube-scanner

# Create sonar-project.properties in project root
cat > sonar-project.properties << EOF
sonar.projectKey=order-service
sonar.projectName=Order Service
sonar.projectVersion=1.0
sonar.sources=src/main
sonar.tests=src/test
sonar.java.binaries=target/classes
sonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml
EOF

# Run scan
sonar-scanner \
  -Dsonar.host.url=https://sonarcloud.io \
  -Dsonar.login=<your-token>
```

---

## 🔄 Enable SonarCloud in CI/CD Pipeline

### **Uncomment SonarCloud Job in `.github/workflows/pipeline.yml`**

```yaml
sonarcloud:
  needs: build
  runs-on: ubuntu-latest
  steps:

    - name: Checkout Code
      uses: actions/checkout@v3
      with:
        fetch-depth: 0  # Important: full history for better analysis

    - name: Setup Java 17
      uses: actions/setup-java@v3
      with:
        distribution: 'temurin'
        java-version: '17'

    - name: Cache Maven Dependencies
      uses: actions/cache@v3
      with:
        path: ~/.m2
        key: maven-${{ hashFiles('**/pom.xml') }}
        restore-keys: maven-

    - name: SonarCloud Scan — Order Service
      run: |
        cd MicroServices/order-service
        mvn verify sonar:sonar \
          -Dsonar.projectKey=${{ secrets.SONAR_PROJECT_KEY_ORDER }} \
          -Dsonar.organization=${{ secrets.SONAR_ORG }} \
          -Dsonar.host.url=https://sonarcloud.io \
          -Dsonar.login=${{ secrets.SONAR_TOKEN }}

    - name: SonarCloud Scan — Payment Service
      run: |
        cd MicroServices/payment-service
        mvn verify sonar:sonar \
          -Dsonar.projectKey=${{ secrets.SONAR_PROJECT_KEY_PAYMENT }} \
          -Dsonar.organization=${{ secrets.SONAR_ORG }} \
          -Dsonar.host.url=https://sonarcloud.io \
          -Dsonar.login=${{ secrets.SONAR_TOKEN }}
```

Then update the docker job dependency:
```yaml
docker:
  needs: [ build, gitleaks, sonarcloud, trivy-fs ]  # Add sonarcloud here
```

---

## 📊 Understanding SonarQube Reports

### **Key Metrics**

| Metric | Meaning | Good Value |
|--------|---------|-----------|
| **Code Quality Gate** | Pass/Fail rating | ✅ PASS |
| **Reliability** | Bug-free percentage | > 90% |
| **Security** | No vulnerabilities | 0 |
| **Maintainability** | Code cleanliness | A or B |
| **Code Coverage** | Test coverage % | > 80% |
| **Duplications** | Duplicate code % | < 3% |

### **Issue Types**

- 🔴 **Blocker** - Critical bugs, must fix
- 🟠 **Critical** - Security vulnerabilities, should fix
- 🟡 **Major** - Design flaws, should fix
- 🔵 **Minor** - Code smells, nice to fix
- ⚪ **Info** - Just informational

---

## 🛠️ pom.xml Configuration

Add to your `pom.xml` for Maven integration:

```xml
<properties>
  <sonar.projectKey>rohit_order-service</sonar.projectKey>
  <sonar.organization>rohit-org</sonar.organization>
  <sonar.host.url>https://sonarcloud.io</sonar.host.url>
  <sonar.sources>src/main</sonar.sources>
  <sonar.tests>src/test</sonar.tests>
  <sonar.coverage.jacoco.xmlReportPaths>target/site/jacoco/jacoco.xml</sonar.coverage.jacoco.xmlReportPaths>
</properties>

<dependencies>
  <!-- JaCoCo for Code Coverage -->
  <dependency>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.8.10</version>
  </dependency>
</dependencies>

<build>
  <plugins>
    <!-- Sonar Plugin -->
    <plugin>
      <groupId>org.sonarsource.scanner.maven</groupId>
      <artifactId>sonar-maven-plugin</artifactId>
      <version>3.9.1.2184</version>
    </plugin>

    <!-- JaCoCo for Coverage Reports -->
    <plugin>
      <groupId>org.jacoco</groupId>
      <artifactId>jacoco-maven-plugin</artifactId>
      <version>0.8.10</version>
      <executions>
        <execution>
          <goals>
            <goal>prepare-agent</goal>
          </goals>
        </execution>
        <execution>
          <id>report</id>
          <phase>test</phase>
          <goals>
            <goal>report</goal>
          </goals>
        </execution>
      </executions>
    </plugin>
  </plugins>
</build>
```

---

## 🎯 Quality Gates (Auto-Failing Builds)

Set up SonarCloud Quality Gates to fail builds if standards aren't met:

In SonarCloud → Project → Quality Gates:
1. Create custom gate or use default
2. Add conditions:
   - Reliability: A or better
   - Security: 0 vulnerabilities
   - Code Coverage: > 80%
   - Duplications: < 3%

When these fail, pipeline blocks deployment!

---

## 🔗 Common Issues & Solutions

### **Issue 1: "Project not found"**
**Solution**: Verify `sonar.projectKey` matches exactly in SonarCloud

### **Issue 2: "Code coverage showing 0%"**
**Solution**: Ensure JaCoCo is generating reports:
```bash
mvn clean verify
# Check: target/site/jacoco/jacoco.xml exists
```

### **Issue 3: "Insufficient privileges"**
**Solution**: Token doesn't have required permissions. Generate new token with full permissions.

### **Issue 4: "Maven plugin not found"**
**Solution**: Add plugin to pom.xml or use sonar-scanner CLI

---

## 📚 Quick Reference Commands

```bash
# Local Maven scan (SonarCloud)
mvn clean verify sonar:sonar \
  -Dsonar.projectKey=my-project \
  -Dsonar.organization=my-org \
  -Dsonar.host.url=https://sonarcloud.io \
  -Dsonar.login=<token>

# With test coverage
mvn clean verify jacoco:report sonar:sonar -Dsonar.login=<token>

# Skip tests, just analyze
mvn clean compile sonar:sonar -DskipTests=true -Dsonar.login=<token>

# Scan specific directory
mvn sonar:sonar -Dsonar.sources=src/main/java -Dsonar.login=<token>
```

---

## 📖 Learn More

- **SonarCloud**: https://sonarcloud.io/documentation
- **SonarQube**: https://docs.sonarqube.org
- **Maven Plugin**: https://github.com/SonarSource/sonar-scanner-maven
- **Quality Gates**: https://docs.sonarqube.org/latest/user-guide/quality-gates/
