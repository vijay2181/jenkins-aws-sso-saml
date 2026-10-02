# Jenkins AWS SSO SAML — End-to-End Architecture Diagram

```
╔══════════════════════════════════════════════════════════════════════════════════════════════╗
║                     JENKINS AWS SSO SAML 2.0 — COMPLETE FLOW                                ║
╚══════════════════════════════════════════════════════════════════════════════════════════════╝


  ┌─────────────────────────────────────────────────────────────────────────────────────────┐
  │                              AWS SETUP (One-Time Configuration)                          │
  └─────────────────────────────────────────────────────────────────────────────────────────┘

  ┌──────────────────────────────────┐       ┌──────────────────────────────────────────────┐
  │    AWS IAM IDENTITY CENTER       │       │         SAML APPLICATION (Jenkins-CI-CD)      │
  │  ─────────────────────────────   │       │  ──────────────────────────────────────────   │
  │                                  │       │                                               │
  │  👥 GROUPS                        │       │  ACS URL:                                     │
  │  ┌──────────────────────────┐    │       │  http://<EC2-IP>:8080/securityRealm/          │
  │  │ AWS-DevOps-Admins        │    │       │                        finishLogin            │
  │  │ AWS-Developers           │    │       │                                               │
  │  └──────────────────────────┘    │       │  Entity ID:                                   │
  │                                  │       │  http://<EC2-IP>:8080/securityRealm/          │
  │  👤 USER                          │       │                        finishLogin            │
  │  ┌──────────────────────────┐    │       │                                               │
  │  │ devopsvijay5@gmail.com   │    │       │  ATTRIBUTE MAPPINGS:                          │
  │  │ Group: AWS-DevOps-Admins │    │       │  ┌─────────────┬─────────────────────────┐   │
  │  └──────────────────────────┘    │       │  │ Subject     │ ${user:email}           │   │
  │                                  │       │  │ username    │ ${user:preferredUsername}│   │
  │  🔑 MFA SETTINGS                  │       │  │ email       │ ${user:email}           │   │
  │  ┌──────────────────────────┐    │       │  │ displayName │ ${user:name}            │   │
  │  │ Never (Disabled)         │    │       │  │ groups      │ ${user:groups}          │   │
  │  │ Session: 8 hours         │    │       │  └─────────────┴─────────────────────────┘   │
  │  └──────────────────────────┘    │       │                                               │
  └──────────────────────────────────┘       └──────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────────────────────────────────────┐
  │                              JENKINS SETUP (One-Time Configuration)                      │
  └─────────────────────────────────────────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────────────────────────────────┐
  │                           JENKINS  (EC2: http://13.232.168.185:8080)                  │
  │  ──────────────────────────────────────────────────────────────────────────────────   │
  │                                                                                       │
  │  PLUGINS INSTALLED:                   SAML REALM CONFIG:                             │
  │  ✅ SAML 2.0                           IdP Metadata XML  : (from AWS download)        │
  │  ✅ Role-Based Authorization Strategy  Display Name Attr : displayName               │
  │                                        Group Attribute   : groups                    │
  │  ROLE STRATEGY (RBAC):                 Username Attribute: username                  │
  │  ┌──────────────────────────────────┐  Email Attribute  : email                      │
  │  │ ROLES:                           │  Case Conversion  : Lowercase                  │
  │  │  admin     → Overall/Administer  │  Data Binding     : HTTP-POST                  │
  │  │  developer → Overall/Read        │  Logout URL       : (AWS SSO logout URL)       │
  │  │              Job/Build           │                                                 │
  │  │              Job/Read            │  AUTHORIZATION:                                 │
  │  │              Job/Workspace       │  Role-Based Strategy ✅                         │
  │  │                                  │                                                 │
  │  │ ASSIGNMENTS:                     │                                                 │
  │  │  AWS-DevOps-Admins → admin       │                                                 │
  │  │  AWS-Developers    → developer   │                                                 │
  │  │  admin (local)     → admin       │                                                 │
  │  │  devopsvijay5@..   → admin       │                                                 │
  │  └──────────────────────────────────┘                                                 │
  └──────────────────────────────────────────────────────────────────────────────────────┘


╔══════════════════════════════════════════════════════════════════════════════════════════════╗
║                          LIVE LOGIN FLOW (Every Time User Logs In)                           ║
╚══════════════════════════════════════════════════════════════════════════════════════════════╝

  ┌─────────────┐
  │    USER     │
  │  (Browser)  │
  └──────┬──────┘
         │
         │  1. Types http://13.232.168.185:8080
         │
         ▼
  ┌─────────────────────────────────────────┐
  │            JENKINS CONTROLLER           │
  │         (EC2 Ubuntu + Docker)           │
  │                                         │
  │  "I don't know who you are.             │
  │   Let me ask AWS to identify you."      │
  │                                         │
  │  Generates ──► SAML AuthnRequest        │
  └────────────────────┬────────────────────┘
                       │
                       │  2. HTTP 302 Redirect
                       │     (SAML AuthnRequest inside URL)
                       │
                       ▼
  ┌─────────────────────────────────────────┐
  │         AWS IAM IDENTITY CENTER         │
  │   portal.sso.ap-south-1.amazonaws.com  │
  │                                         │
  │  "Please prove who you are."            │
  └────────────────────┬────────────────────┘
                       │
                       │  3. Shows Login Page
                       │
                       ▼
  ┌─────────────────────────────────────────┐
  │    USER ENTERS CREDENTIALS              │
  │                                         │
  │  📧 Email:    devopsvijay5@gmail.com     │
  │  🔒 Password: **************            │
  │  📱 MFA:      (disabled — skipped)      │
  └────────────────────┬────────────────────┘
                       │
                       │  4. AWS Validates Credentials
                       │
                       ▼
  ┌─────────────────────────────────────────┐
  │         AWS IAM IDENTITY CENTER         │
  │                                         │
  │  Checks:                                │
  │  ✅ Is devopsvijay5@gmail.com valid?    │
  │  ✅ Is password correct?               │
  │  ✅ What groups is the user in?        │
  │     → AWS-DevOps-Admins                │
  │                                         │
  │  Builds SAML 2.0 Assertion (token):    │
  │  ┌───────────────────────────────────┐  │
  │  │ Subject   : devopsvijay5@gmail.com│  │
  │  │ username  : devopsvijay5          │  │
  │  │ email     : devopsvijay5@gmail.com│  │
  │  │ displayName: devops vijay         │  │
  │  │ groups    : AWS-DevOps-Admins     │  │
  │  │                                   │  │
  │  │ 🔏 Digitally Signed with          │  │
  │  │    AWS Private Key (X.509 Cert)   │  │
  │  └───────────────────────────────────┘  │
  └────────────────────┬────────────────────┘
                       │
                       │  5. HTTP POST with signed SAML Assertion
                       │     to: /securityRealm/finishLogin
                       │
                       ▼
  ┌─────────────────────────────────────────┐
  │            JENKINS CONTROLLER           │
  │                                         │
  │  Receives SAML Assertion                │
  │  Verifies AWS digital signature ✅      │
  │                                         │
  │  Extracts attributes:                   │
  │  username  → devopsvijay5@gmail.com     │
  │  groups    → AWS-DevOps-Admins          │
  │                                         │
  │  Checks Role-Strategy matrix:           │
  │  AWS-DevOps-Admins → admin role         │
  │  admin role → Overall/Administer        │
  │                                         │
  │  Result: ✅ FULL ADMIN ACCESS           │
  └────────────────────┬────────────────────┘
                       │
                       │  6. Redirects to Jenkins Dashboard
                       │
                       ▼
  ┌─────────────────────────────────────────┐
  │           JENKINS DASHBOARD             │
  │                                         │
  │  👋 Welcome, devops vijay               │
  │  🔑 Role: Administrator                 │
  │  📧 Email: devopsvijay5@gmail.com       │
  │                                         │
  │  ✅ Logged in via AWS SSO               │
  │  ✅ Zero passwords stored in Jenkins    │
  │  ✅ Session valid for 8 hours           │
  └─────────────────────────────────────────┘


╔══════════════════════════════════════════════════════════════════════════════════════════════╗
║                        SSO SESSION PERSISTENCE (2nd Visit — Same Browser)                    ║
╚══════════════════════════════════════════════════════════════════════════════════════════════╝

  ┌─────────────┐
  │    USER     │  Opens NEW TAB or revisits Jenkins within 8 hours
  │  (Browser)  │
  └──────┬──────┘
         │
         │  1. Types http://13.232.168.185:8080
         │
         ▼
  ┌─────────────────────────────────────────┐
  │            JENKINS CONTROLLER           │
  │  Redirects to AWS (SAML AuthnRequest)   │
  └────────────────────┬────────────────────┘
                       │
                       ▼
  ┌─────────────────────────────────────────┐
  │         AWS IAM IDENTITY CENTER         │
  │                                         │
  │  Checks browser session cookie          │
  │  "This user already authenticated       │
  │   30 minutes ago. Session still valid." │
  │                                         │
  │  ⚡ SKIPS login page entirely           │
  │  ⚡ SKIPS MFA entirely                  │
  │                                         │
  │  Immediately issues new SAML Assertion  │
  └────────────────────┬────────────────────┘
                       │
                       ▼
  ┌─────────────────────────────────────────┐
  │           JENKINS DASHBOARD             │
  │                                         │
  │  ⚡ INSTANT LOGIN — Zero prompts        │
  │  ✅ No password asked                   │
  │  ✅ No MFA asked                        │
  │  ✅ TRUE SINGLE SIGN-ON                 │
  └─────────────────────────────────────────┘


╔══════════════════════════════════════════════════════════════════════════════════════════════╗
║                              ROLE MAPPING SUMMARY                                            ║
╚══════════════════════════════════════════════════════════════════════════════════════════════╝

  AWS IAM Identity Center          SAML Assertion             Jenkins RBAC
  ══════════════════════           ══════════════             ════════════

  User: devopsvijay5@gmail.com  ──► groups:               ──► AWS-DevOps-Admins ──► admin role
  Group: AWS-DevOps-Admins           AWS-DevOps-Admins         (Overall/Administer)
                                                               ✅ FULL ACCESS

  User: any-developer@co.com    ──► groups:               ──► AWS-Developers ──► developer role
  Group: AWS-Developers              AWS-Developers             (Read + Build)
                                                               ✅ LIMITED ACCESS

  Local user: admin             ──► (no SAML)             ──► admin (break-glass)
  (Docker Jenkins)                                             ✅ EMERGENCY FALLBACK


╔══════════════════════════════════════════════════════════════════════════════════════════════╗
║                              DOCKER VOLUME PERSISTENCE                                       ║
╚══════════════════════════════════════════════════════════════════════════════════════════════╝

  sudo docker stop jenkins      ◄──  Stops container ONLY
         │                           Data stays safe in volume
         │
         ▼
  ┌────────────────────────┐
  │  Docker Named Volume   │    jenkins_home:/var/jenkins_home
  │  jenkins_home          │    ─────────────────────────────
  │                        │    ✅ Plugins (SAML, Role-Strategy)
  │  Lives on EC2 disk     │    ✅ SAML XML config
  │  Survives container    │    ✅ RBAC roles & assignments
  │  stop/start/remove     │    ✅ Admin user credentials
  │                        │    ✅ All jobs & pipelines
  └────────────────────────┘    ✅ Build history
         │
         ▼
  sudo docker start jenkins     ◄──  Restarts container
                                      Everything 100% restored


  ⚠️  WARNING:
  ┌──────────────────────────────────────────────────────────────────────────┐
  │  Stopping Docker container  = ✅ SAFE  (same EC2 IP, volume intact)      │
  │  Stopping EC2 instance      = ⚠️  EC2 gets NEW PUBLIC IP on restart      │
  │                               Must update ACS URL in AWS IAM Identity    │
  │                               Center → Applications → Jenkins-CI-CD      │
  │                               → Edit Configuration → Update both URLs    │
  └──────────────────────────────────────────────────────────────────────────┘
  Fix: Attach an Elastic IP to the EC2 instance to keep a permanent static IP.
```
