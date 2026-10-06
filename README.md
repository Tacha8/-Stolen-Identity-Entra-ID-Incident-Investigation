# Stolen-Identity-Entra-ID-Incident-Investigation

## Scenario

A compromised user account was found to have ownership of a legacy Entra ID application. The investigation focused on determining how an attacker could move from the compromised identity into application-level access and eventually abuse an OAuth-based attack path.

The main question was: **How did the compromised identity become a path to privileged application functionality?**

## Environment

- **Platform:** Microsoft Azure
- **Identity Platform:** Microsoft Entra ID
- **Applications:** `Mad-Hat-Legacy-Sync-Service`, `Mad-Hat-Labs-App`
- **Services:** App registrations, Enterprise applications, Certificates & secrets, API permissions, Expose an API, Authentication
- **Tools:** Azure Portal, CyberChef
- **Access Level:** Azure training tenant
- **Investigation Type:** Identity and application security investigation

## Investigation

### 1. Investigated the compromised user

I started by looking at the compromised user's access and ownership relationships in Entra ID.

I found that the user was listed as an **Owner** of `Mad-Hat-Legacy-Sync-Service`.

That was important because the compromised account was not only a normal user identity. It also had control over an application registration.

This meant I needed to investigate the application itself and determine what could be changed through application ownership.

**Conclusion:** The compromised user provided a potential entry point into the application's configuration because of its ownership privileges.

---

### 2. Investigated the application's credentials

I opened the legacy application's **Certificates & secrets** section and reviewed the credentials configured for the application.

I found a client secret with an expiration date of **12/31/2099**.

The extremely long expiration stood out because a credential with that kind of lifetime could remain usable for a very long time if it were compromised.

The important part was connecting the credential back to the application's ownership. The compromised user had ownership of the application, and the application had a long-lived credential that could be used for application authentication.

**Conclusion:** The application's credential configuration created a significant persistence concern and needed to be considered as part of the attack path.

**Evidence:**

<img width="1187" height="612" alt="image" src="https://github.com/user-attachments/assets/f9c4d253-8abd-4d1f-a645-8ee804e390f6" />


### 3. Investigated the application's permissions

Next, I reviewed the permissions assigned to `Mad-Hat-Legacy-Sync-Service`.

The application had broad Microsoft Graph application permissions, including:

- `Directory.Read.All`
- `User.ReadWrite.All`

This showed that the application itself had significant privileges in the tenant.

That made the credential finding more important because compromising an application with broad permissions could have a much larger impact than compromising a low-privilege application.

**Conclusion:** The application's permissions, credentials, and ownership needed to be evaluated together rather than independently.


---

### 4. Investigated the application ownership chain

I continued following the ownership relationships and found that `Mad-Hat-Labs-App` had been added as an owner of the legacy application.

This introduced another application identity into the attack path.

At this point, the investigation had moved beyond the original compromised user:

    Compromised User
           ↓
    Mad-Hat-Legacy-Sync-Service
           ↓
    Mad-Hat-Labs-App

**Conclusion:** The rogue application was connected to the legacy application through ownership and needed to be investigated as part of the same incident.

---

### 5. Investigated the rogue application's configuration

I opened `Mad-Hat-Labs-App` and reviewed its configuration to determine how it could interact with the trusted legacy application.

I focused on the areas that could establish trust or allow the application to request access to functionality provided by another application.

This led me to the legacy application's **Expose an API** configuration.

The important thing I was trying to determine was not simply whether the rogue application existed, but what access it could request from the trusted application.

**Conclusion:** The ownership relationship was only one part of the attack. I needed to determine what access the rogue application could request from the trusted application.

**Evidence:**

<img width="1046" height="610" alt="image" src="https://github.com/user-attachments/assets/8f258d5d-21a2-43df-a5c0-9b345b02f3e3" />


### 6. Investigated the exposed API scope

I opened the legacy application's:

**Expose an API → Scopes defined by this API**

I found a custom API scope.

This became an important part of the attack chain because the rogue application could request access to functionality exposed by the legacy application.

One of the things I had to understand was that the rogue application did **not** simply inherit the legacy application's full application permissions.

Instead, the OAuth flow could provide the rogue application with delegated access to the exposed API after user consent.

This distinction was important because it explained how the attacker could interact with the trusted application without directly receiving all of the application's privileged permissions.

**Conclusion:** The custom API scope created another trust relationship that became part of the attack path.

---

### 7. Investigated the redirect URI

I then reviewed the authentication configuration for the rogue application.

I found a suspicious redirect URI associated with the application.

A redirect URI is where the authentication response is sent after an OAuth authentication flow. In this investigation, the redirect URI became another piece of evidence connecting the rogue application to the attack.

This showed me that redirect URIs are not just normal application configuration. They can become an important part of an OAuth attack path.

**Conclusion:** The redirect URI was another indicator that the rogue application was being used as part of the attack.

---

### 8. Reconstructed the OAuth attack path

After reviewing the user, application ownership, credentials, permissions, API scope, and OAuth configuration, I was able to connect the findings into one attack chain.

The attacker started with a compromised user identity that had ownership of the legacy application. That access allowed the attacker to interact with the application's configuration and establish another application identity in the ownership chain.

The rogue application could then use the exposed API and OAuth flow to obtain delegated access to the API after user consent.

The important distinction was that the rogue application did **not** directly receive the legacy application's full application permissions.

Instead, the trusted legacy application could potentially become a **confused deputy**, performing privileged work using permissions it already possessed.

The attack path was:

    Compromised User
           ↓
    Application Ownership
           ↓
    Client Secret
           ↓
    Rogue Application
           ↓
    Application Ownership
           ↓
    Custom API Scope
           ↓
    OAuth Consent
           ↓
    Suspicious Redirect URI
           ↓
    Trusted Application

**Conclusion:** The investigation showed how a compromised identity could become an entry point into an application's trust relationships and eventually lead to an OAuth-based attack path.

---

## What broke / what surprised me

The part that took me the longest to understand was the **Expose an API → custom scope → OAuth consent** portion of the investigation.

My first assumption was that if the rogue application was connected to the legacy application, it would simply receive the legacy application's privileged permissions. That wasn't the case.

I had to understand the difference between:

- **Application permissions** — permissions an application uses on its own.
- **Delegated permissions** — permissions an application uses when acting on behalf of a user.

The rogue application was getting delegated access to the exposed API through the OAuth consent flow. The more important part was understanding what the trusted legacy application could then do using its own permissions.

The **confused deputy** concept was also something I had to work through. The attacker doesn't necessarily need to directly possess the privileged permissions if they can get a trusted application to perform work using the permissions it already has.

Another thing that surprised me was how difficult it would be to understand this attack by looking at only one object.

The compromised user alone didn't explain the attack.

The client secret alone didn't explain the attack.

The rogue application alone didn't explain the attack.

The attack became clear after connecting the relationships between the objects.

If I were doing this investigation again, I would map the ownership, credential, permission, API, and OAuth relationships much earlier instead of investigating each configuration area separately.

## Findings and recommendations

### Finding 1 — Excessive application ownership

The compromised user had ownership of the legacy application, creating a path from a compromised identity into application configuration.

**Recommendation:** Review application owners regularly and remove ownership from users who do not require it.

### Finding 2 — Extremely long-lived client secret

The legacy application had a client secret configured to expire on **12/31/2099**.

**Recommendation:** Use shorter credential lifetimes and establish regular credential rotation for application secrets.

### Finding 3 — Broad application permissions

The legacy application had broad Microsoft Graph application permissions, including `Directory.Read.All` and `User.ReadWrite.All`.

**Recommendation:** Review application permissions using least privilege and remove permissions that are not required for the application's function.

### Finding 4 — Rogue application in the ownership chain

`Mad-Hat-Labs-App` had been added as an owner of the legacy application.

**Recommendation:** Regularly review application ownership and investigate unexpected application owners and service principals.

### Finding 5 — Custom API scope and OAuth attack path

The legacy application exposed a custom API scope that became part of the OAuth attack path.

**Recommendation:** Regularly review exposed APIs, custom scopes, redirect URIs, and OAuth consent relationships for applications that handle sensitive functionality.

## What I learned

- **Technical:** I learned how application ownership, client credentials, API scopes, and OAuth configuration can combine to create an identity attack path in Microsoft Entra ID.

- **Technical:** I learned the difference between application permissions and delegated access and why that distinction matters when investigating OAuth-based attacks.

- **Investigation:** I learned that I need to follow relationships between users, applications, credentials, permissions, API scopes, and OAuth configuration instead of investigating each object independently.

- **What I'd do differently:** I would map the application's owners, credentials, permissions, exposed APIs, and redirect URIs much earlier instead of investigating each section separately.

- **Biggest takeaway:** A compromised identity can become much more dangerous when it has control over an application. During an identity investigation, I need to follow what the compromised identity can control, not just what the identity itself can access.
